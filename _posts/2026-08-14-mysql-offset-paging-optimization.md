---
title: "MySQL OFFSET 페이징이 100만 건에서 급격히 지연되는 이유: EXPLAIN ANALYZE와 InnoDB Buffer Pool로 분석한 커서 스트리밍 최적화"
date: 2026-08-14 13:45:00 +0900
categories: [Database, MySQL]
tags: [mysql, database, performance, batch, spring-batch, indexing, optimization]
description: 100만 건 대용량 정산 배치 가동 시 발생하는 RDBMS 물리적 병목을 범위 파티셔닝과 커서 스트리밍으로 해결한 성능 최적화 과정을 분석합니다.
---

> **📌 개요**
> 
> 본 포스팅은 KickSync 대용량 결제 정산 시스템(100만 건)을 운영하며 발생한 RDBMS 물리적 병목(`LIMIT/OFFSET` O(N^2) 스캔, InnoDB Buffer Pool LRU Churn, `fsync()` I/O 부하)을 식별하고 **범위 파티셔닝, 커서 스트리밍, 인메모리 Micro-batch 집계**를 통해 배치 처리 성능을 최적화(14분 16초 ➔ 1분 9초, Disk Write 1.8GB ➔ 26.9MB)한 엔지니어링 과정을 다룹니다.

<br>

## 1. 100만 건 정산 배치의 딜레마: “1,000건만 읽는데 왜 DB가 멈출까?”

이커머스 플랫폼 KickSync에서는 매일 자정 입점사별 결제 내역 100만 건을 취합하여 일별 정산 통계 테이블(`daily_sales_stats`)을 생성하는 배치 작업을 수행합니다. 안정적인 대용량 처리를 위해 Spring Batch의 표준 페이징 리더인 `JpaPagingItemReader`를 적용했습니다.

그러나 단일 Pod 자원 제한 환경(WAS CPU 0.8 Core / Memory 1.5 GiB, DB CPU 1.0 Core / Memory 2.0 GiB, HikariCP 20)에서 배치를 가동하자 임계치에 도달하는 리소스 병목이 발생했습니다.

* **DB CPU 사용률:** 가동 직후 최대 **87.43%** 까지 상승하며 임계값 포화
* **총 소요 시간:** 100만 건 처리에 **14분 16.29초 (856,294 ms)** 소요
* **커넥션 고갈:** 쿼리 지연으로 인한 `HikariCP Connection Leak` 경보 빈발

```sql
-- JpaPagingItemReader가 900번째 페이지에서 생성한 쿼리
SELECT *
FROM settlements
WHERE settlement_date = '2026-06-01'
ORDER BY id ASC
LIMIT 1000 OFFSET 900000;
```

표면적으로 위 쿼리는 1,000건의 레코드만 가져옵니다. 하지만 배치가 후반부 페이지로 진입할수록 단일 쿼리 응답 속도가 지속적으로 지연되었습니다.

‘단순히 1,000건만 메모리에 올리는 쿼리가 왜 DB CPU를 80% 이상 점유하고 기가바이트 단위의 디스크 I/O를 유발할까?’

프레임워크의 추상화 뒤에 숨겨진 **MySQL InnoDB 스토리지 엔진의 물리적 데이터 탐색 메커니즘**을 분석했습니다.

<br>

## 2. InnoDB 내부 동작: B+Tree 스캔과 Buffer Pool의 이면

`LIMIT N OFFSET M` 쿼리가 실행될 때 MySQL 엔진 내부에서는 다음과 같은 물리적 작업이 발생합니다.

<img width="1152" height="583" alt="image" src="https://github.com/user-attachments/assets/53f77d98-52ee-4a54-bae4-1ddd7e8963ef" />

### 1) B+Tree Leaf 노드 순회와 O(N) 폐기 비용

InnoDB의 Clustered Index는 B+Tree 구조로 이루어져 있고 리프 노드들은 양방향 연결 리스트로 연결되어 있습니다.

* 엔진은 루트 노드에서 시작해 `ORDER BY id ASC`의 시작 지점을 찾은 후 리프 노드를 따라 OFFSET 900,000개의 레코드를 물리적으로 하나씩 순회합니다.
* 핵심은 “지나치는 90만 건도 디스크 및 버퍼 풀에서 메모리로 로드한 뒤 버린다”는 점입니다.
* 배치가 1페이지(OFFSET 0)부터 1,000페이지(OFFSET 999,000)까지 진행되면 총 스캔 횟수는 아래와 같이 누적됩니다.


<img width="70%" height="50%" alt="image" src="https://github.com/user-attachments/assets/26a3d608-4fd3-41b8-be8e-872a88a8b8b9" />


100만 건 처리 시 엔진이 내부적으로 읽고 버린 레코드는 약 5억 건(500,500,000건)에 달합니다.

### 2) Buffer Pool 오염과 LRU 리스트 Churn

InnoDB는 디스크 I/O를 줄이기 위해 16KB 단위의 데이터 페이지를 메모리(InnoDB Buffer Pool)에 캐싱합니다. 버퍼 풀은 **Young(5/8)과 Old(3/8) 서브리스트**로 구성된 LRU 알고리즘으로 관리됩니다.

* OFFSET이 커질수록 수만 개의 16KB 데이터 페이지가 디스크에서 읽혀 Buffer Pool의 Old 영역으로 대량 유입됩니다.
* MySQL의 기본 `innodb_old_blocks_time` 보호 설정에도 불구하고 100만 건의 연속적인 풀 스캔성 OFFSET 연산은 수만 개의 16KB 페이지를 단시간에 밀어내며 Old Sublist를 넘어 버퍼 풀 전체의 Cache Churn(오염 및 Eviction)을 유발합니다.
* 이로 인해 온라인 트랜잭션(OLTP)에서 빈번히 사용 중이던 핫 캐시 페이지가 버퍼 풀 밖으로 밀려나며 버퍼 풀 적중률이 하락하고 물리 디스크 Random Read가 발생합니다.

<br>

## 3. EXPLAIN ANALYZE로 계측한 물리적 실행 비용

MySQL 8.0의 `EXPLAIN ANALYZE`를 통해 OFFSET 방식과 인덱스 범위 탐색의 실제 엔진 실행 트리를 계측했습니다.

### AS-IS: `LIMIT 1000 OFFSET 900000` 실행 계획

```text
-> Limit/Offset: 1000/900000 row(s)  (cost=91243.50 rows=1000) (actual time=682.114..774.230 rows=1000 loops=1)
    -> Index scan on settlements using PRIMARY  (cost=91243.50 rows=901000) (actual time=0.045..712.890 rows=901000 loops=1)
```

* `Index scan on settlements`: 엔진은 901,000개의 인덱스 엔트리를 순차 스캔하는 데 **712.89ms**를 소비했습니다.
* `Limit/Offset`: 90만 개를 버리고 1,000개만 반환하는 필터링 단계까지 총 **774.23ms**의 지연이 발생했습니다. 배치가 후반부로 갈수록 단일 쿼리 지연이 **0.77초**에 도달하며 1,000번의 페이징 쿼리가 누적되어 전체 배치에서 총 **774초(약 13분)** 의 SQL 지연을 유발합니다.

### TO-BE: 커서 기반 조건절 탐색 (`WHERE id > ? LIMIT 1000`) 실행 계획

```text
-> Limit: 1000 row(s)  (cost=201.50 rows=1000) (actual time=0.032..0.854 rows=1000 loops=1)
    -> Index range scan on settlements using PRIMARY, with index condition: (settlements.id > 900000)  (cost=201.50 rows=1000) (actual time=0.030..0.762 rows=1000 loops=1)
```

* B+Tree의 루트부터 수직 탐색( O(log N) )하여 `id = 900001` 리프 노드로 이동합니다.
* 불필요한 레코드 스캔 없이 필요한 1,000건의 페이지만 읽어 **0.854ms** 만에 연산을 종료합니다.

> **💡 Keyset 페이징과 소켓 커서 스트리밍의 차이**  
> Keyset(No-Offset) 방식이 매 청크마다 인덱스 B+Tree 수직 탐색( O(log N) )을 통해 성능 개선을 이끌어낸다면 본 프로젝트에 적용한 `JdbcCursorItemReader`는 단 한 번의 커넥션 소켓 오픈으로 커서를 유지하며 O(N) 선형 전진만 수행하므로 쿼리 파싱 및 B+Tree 재탐색 오버헤드를 배제합니다.

<br>

## 4. TO-BE: 소켓 커서 스트리밍과 파이프라인 설계

OFFSET 방식의 한계를 극복하기 위해 배치 파이프라인 구조를 재설계했습니다.

<img width="1022" height="1333" alt="image" src="https://github.com/user-attachments/assets/cd7626e2-3b15-4d7f-86a0-0f25ee21bebd" />

### 1) `JdbcCursorItemReader`를 통한 소켓 스트리밍 전환

반복적인 `LIMIT/OFFSET` 쿼리 발송을 중단하고 데이터베이스 네트워크 커넥션 소켓을 유지한 채 `ResultSet.next()`로 레코드를 1건씩 스트리밍 수신했습니다.

* **B+Tree 재탐색 제거:** 쿼리 파싱 및 B+Tree 순회 비용을 단 1회로 통제했습니다.
* **선형 탐색 O(N) 유지:** DB 엔진은 이미 열린 커서의 위치에서 포워드 스캔만 수행하므로 전체 100만 건 조회 시간을 선형적으로 단축했습니다.

### 2) `PartnerIdPartitioner` 기반의 락 격리

10개 워커 스레드가 단일 테이블에 동시 쓰기를 수행할 때 발생하는 `(partner_id, settlement_date)` 복합 유니크 인덱스상의 **공유 갭 락 데드락**을 해결하기 위해 입점사 ID 범위별 파티셔닝을 적용했습니다.

* 각 스레드가 서로 다른 `partner_id` 범위(예: 1~1,000, 1,001~2,000)를 전담하도록 분할하여 스레드 간 인덱스 락 경합을 구조적으로 배제했습니다.

### 3) In-Memory 집계 및 `rewriteBatchedStatements`

* **JVM Micro-batch:** 1,000건의 청크 데이터를 개별 DB 쓰기로 처리하지 않고 JVM 힙 메모리에서 `aggregatedMap.merge()`로 1차 집계하여 쓰기 요청 수를 99% 절감했습니다.
* **Multi-Row Bulk Write:** JDBC URL에 `rewriteBatchedStatements=true`를 활성화하여 단건 DML 1,000건을 `INSERT INTO ... VALUES (...), (...), (...)`의 Multi-row 구문으로 통합 전송했습니다.
* **fsync() 동기화 오프로딩:** 건건이 트랜잭션 커밋 시 발생하던 100만 번의 디스크 Redo Log `fsync()` 동기 호출을 일괄 처리로 묶어 물리 Disk Write를 **1.8GB에서 26.9MB로 98.5% 절감**했습니다.

<br>

## 5. 실측 벤치마크 및 계측 데이터 검증

자원 제한 환경(WAS 0.8 Core, DB 1.0 Core)에서 100만 건 결제 정산 배치를 실행하고 Scouter APM을 통해 계측한 정량적 성능 지표입니다.

| 지표 항목 | AS-IS (단일 스레드 OFFSET) | TO-BE (파티셔닝 + 커서 스트리밍) | 개선 효과 및 의의 |
| :--- | :--- | :--- | :--- |
| **총 처리 소요 시간** | **14분 16.29초 (856,294 ms)** | **1분 9.49초 (69,493 ms)** | **약 12.3배 연산 속도 단축** |
| **데이터베이스 CPU 점유율** | **최대 87.43%** (임계값 포화) | **평균 16.35%** (안정 상태) | **DB CPU 부하 71.08%p 감소 (가용성 확보)** |
| **애플리케이션 CPU 점유율** | 평균 16.00% (I/O 대기 정체) | **평균 80.02%** (연산 가동) | **병목 주도권을 WAS 연산 영역으로 전환** |
| **물리 Disk Write I/O** | **1.8 GB 이상** (100만 회 Upsert) | **26.9 MB로 감소** | **물리 디스크 쓰기 부하 98.5% 절감** |
| **Scouter SQL Time (Avg)** | **774,656 ms** (1,004,027 회) | **104 ms** (33 회) | **SQL 쿼리 지연 99.9% 절감** |
| **HikariCP 커넥션 상태** | **Connection Leak 경보** 다발 | **0건** (안정적 반납 순환) | **장기 점유로 인한 커넥션 고갈 해결** |
| **정산 금액 정합성** | 100.00% 일치 (150억 원) | **100.00% 일치 (150억 원)** | **150억 원 규모 정산 데이터 정합성 확인** |
| **내결함성 (Fault Tolerant)** | 예외 1건 발생 시 전체 롤백 | **DLQ (`settlement_error_logs`)** | **최대 100회 Skip 허용 (가동률 100.00% 유지)** |

```text
[AS-IS JpaPagingItemReader]
► txid     = x67an93bkkbga1
► elapsed  = 856,294 ms (약 14분 16초)
► sqlCount = 1,004,027 회
► sqlTime  = 774,656 ms

[TO-BE JdbcCursorItemReader + Bulk Write]
► txid     = x67an93bkkbga4
► elapsed  = 69,493 ms (약 1분 9초)
► sqlCount = 33 회 (99.99% 절감)
► sqlTime  = 104 ms (99.99% 절감)
```

<br>

## 6. 트레이드오프와 런타임 한계: 커서 스트리밍 도입 시 고려 및 통제 요소

커서 스트리밍을 도입할 때 사전에 통제해야 하는 런타임 리스크입니다.

1. **DB Connection 장기 점유 (Connection Starvation):**
    * 스트리밍 방식은 데이터 판독이 끝날 때까지 단일 DB 커넥션을 계속 유지합니다. 배치 작업 처리 시간이 길어지면 커넥션 풀이 고갈되어 일반 사용자의 웹 요청이 블로킹될 수 있습니다.
    * **대응:** 배치 전용 `DataSource` 및 `HikariCP` 풀을 분리하고 파티션 워커 수(10개)와 커넥션 풀 크기(20개)를 격리 통제했습니다.
2. **네트워크 소켓 타임아웃 (`socketTimeout`):**
    * `ItemProcessor` 단계에서 무거운 외부 연산이 지연될 경우 MySQL 소켓의 `net_write_timeout` 또는 드라이버 `socketTimeout`이 트리거되어 커넥션이 강제 종료될 수 있습니다.
    * **대응:** Processor 로직을 순수 인메모리 연산으로 유지하고 청크 버퍼 크기를 최적화했습니다.
3. **스레드 스큐 현상:**
    * `PartnerIdPartitioner`로 단순 ID 범위를 균등 분할할 경우 특정 대형 입점사에 데이터가 편중되어 1개 스레드만 지연되는 롱테일 병목이 발생할 수 있습니다.
    * **대응:** 데이터 편차가 심한 도메인일 경우 ID 범위 분할 대신 사전 집계 통계 기반의 동적 파티셔닝 구조를 검토해야 합니다.

<br>

## 7. 결론: ORM 추상화 너머 RDBMS 스토리지 엔진을 통제하라

`JpaPagingItemReader`의 간결한 인터페이스 뒤에는 **B+Tree O(N^2) 순회, Buffer Pool 오염, 대량의 `fsync()` I/O 부하**라는 물리적 비용이 발생하고 있었습니다.

대규모 데이터를 다루는 백엔드 엔지니어링의 핵심은 프레임워크의 편의 기능에 머무르지 않고 “이 코드가 스토리지 엔진과 물리 디스크, 네트워크 소켓에서 실제로 어떻게 실행되는가?”를 추적하여 검증하는 데 있습니다.

데이터 규모가 확장될수록 ORM의 추상화 계층을 넘어 커서 스트리밍, 인메모리 버퍼링, 벌크 I/O 파이프라인처럼 물리 계층의 비용을 통제하는 아키텍처 설계가 필수적입니다.

