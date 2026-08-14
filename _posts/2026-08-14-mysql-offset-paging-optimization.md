---
title: MySQL OFFSET 페이징은 왜 100만 건에서 폭사할까? EXPLAIN ANALYZE와 InnoDB Buffer Pool로 파헤친 배치 최적화
date: 2026-08-14 13:45:00 +0900
categories: [Database, MySQL]
tags: [mysql, database, performance, batch, spring-batch, indexing, optimization]
description: 100만 건 대용량 정산 배치 가동 시 발생하는 RDBMS 물리적 병목을 범위 파티셔닝과 커서 스트리밍으로 해결한 성능 최적화 과정을 분석합니다.
---

## MySQL OFFSET 페이징은 왜 100만 건에서 폭사할까? EXPLAIN ANALYZE와 InnoDB Buffer Pool로 파헤친 배치 최적화

> **📌 개요**
> 
> 본문은 입점사별 100만 건 대용량 정산 배치 가동 시 발생하는 RDBMS 물리적 병목을 분석하고 이를 범위 파티셔닝과 커서 스트리밍으로 극복한 과정을 다룹니다. (작성 중인 임시 포스팅입니다.)

<br>

### 1. 대용량 정산 배치와 RDBMS의 한계

- 100만 건 이상의 데이터를 처리할 때 발생하는 페이징 및 디스크 I/O 병목 현상에 대한 고찰
- 단일 Pod 제약 환경에서의 리소스 포화 문제 식별

<br>

### 2. [AS-IS 분석] 무엇이 DB를 죽게 만드는가?

- **LIMIT/OFFSET 페이징의 O(N^2) 스캔 오버헤드:** 앞서 읽고 버린 데이터를 다시 스캔하는 물리적 메커니즘
- **단건 DML과 `fsync()` 폭격:** 1.8GB에 달하는 물리 Disk Write 유발 원인
- **복합 유니크 제약 Gap Lock 데드락:** 멀티스레드 동시 쓰기 시 발생하는 순환 대기 현상

<br>

### 3. [TO-BE 설계] 엔진 관점의 튜닝 파이프라인

- `PartnerIdPartitioner`를 통한 락 격리 범위 설정
- `JdbcCursorItemReader` 기반의 커서 스트리밍 O(N) 선형 스캔 전환
- `aggregatedMap.merge`와 `rewriteBatchedStatements=true`를 통한 Multi-Row Bulk Write 최적화

<br>

### 4. 실측 벤치마크 및 결론

- Scouter APM 계측 데이터 비교 (`sqlCount` 100만 회 ➔ 33회, 처리 시간 14분 16초 ➔ 1분 9초 단축)
- ORM 추상화 뒤에 숨겨진 DB 엔진 메커니즘 이해의 중요성
