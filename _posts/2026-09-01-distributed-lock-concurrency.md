---
title: "분산 락을 걸었는데 왜 재고가 깨질까? @Transactional 생명주기 불일치와 SpEL 정렬 락으로 해결한 동시성 제어"
date: 2026-09-01 10:00:00 +0900
categories: [Architecture, Concurrency]
tags: [distributed-lock, redis, redisson, spring-boot, transaction, deadlock, circuit-breaker, performance]
description: 선착순 주문 고부하 환경에서 분산 락과 DB 트랜잭션 생명주기 불일치로 인한 초과 판매와 데드락을 SpEL 정렬 락, REQUIRES_NEW 격리, Read/Write 서킷브레이커로 해결한 과정을 분석합니다.
---

> **📌 개요**
> 
> 
> 본 포스팅은 이커머스 플랫폼 KickSync의 선착순 주문 시스템(피크 1,000 TPS 급)을 구축하며 식별한 **분산 락과 DB 트랜잭션 간 생명주기 불일치, 다중 락 교착 상태, 외부 PG 연동에 따른 DB 커넥션 풀 고갈 병목**을 분석하고 **SpEL 정렬 락, `REQUIRES_NEW` 트랜잭션 격리, Read/Write 서킷브레이커 분리**를 통해 시스템 가용성과 데이터 정합성을 확보한 엔지니어링 과정을 다룹니다.
> 

<br>

## 1. 선착순 주문의 딜레마: “분산 락을 적용했는데 왜 초과 판매와 데드락이 발생할까?”

한정판 신발 발매 이벤트와 같이 순간적으로 대량의 트래픽이 인입되는 도메인에서는 재고 차감의 동시성 제어가 시스템의 핵심입니다. 단일 JVM 동기화(`synchronized`, `ReentrantLock`)는 분산 서버 환경에서 동작하지 않으므로 Redis 기반의 분산 락을 도입하여 임계 영역을 보호하도록 설계했습니다.

그러나 리소스가 제한된 단일 Pod 환경(WAS 0.8 vCPU / 1.5 GiB, DB 1.0 vCPU / 2.0 GiB, HikariCP 10)에서 500 VUs 부하 테스트(k6)를 가동하자 동시성 결함과 시스템 정체가 발생했습니다.

- **초과 판매 발생:** 분산 락으로 재고 차감 로직을 감쌌음에도 최종 재고가 음수로 떨어지는 데이터 정합성 균열 발생
- **스레드 풀 포화 및 지연:** 톰캣 작업 스레드 200개가 `TIMED_WAITING` 상태로 정체되며 평균 응답 속도가 **6.96초(6,960ms)**까지 증가
- **커넥션 풀 고갈 및 연쇄 장애:** 외부 PG 결제 검증 지연(500ms)으로 인해 10개의 DB 커넥션이 고갈되었고 무관한 일반 사용자의 JWT 인증 필터 조회까지 `ConnectionTimeoutException`을 유발하며 가용성이 0%로 저하

```java
// AS-IS: 락과 트랜잭션이 결합된 흔한 안티패턴
@Transactional
public void createOrder(OrderRequest request) {
    RLock lock = redissonClient.getLock("PRODUCT:" + request.getProductId());
    lock.lock();
    try {
        decreaseStock(request.getProductId(), request.getQuantity()); // 1. 재고 차감
        paymentClient.verifyPayment(request.getPaymentId());         // 2. 외부 PG 검증 (500ms 지연)
    } finally {
        lock.unlock(); // 3. 락 해제 (문제 발생 지점)
    }
} // 4. 트랜잭션 커밋 및 커넥션 반납 (지연 발생 지점)
```

표면적으로는 락을 획득하고 데이터를 수정한 뒤 락을 정상 해제하는 것처럼 보입니다.

‘왜 분산 락을 적용했는데도 동시 요청에서 과거 재고를 읽는 현상이 발생하고 톰캣 스레드는 교착 상태에 빠질까?’

프레임워크의 편리한 AOP 추상화 뒤에 숨겨진 **스프링 트랜잭션 프록시와 분산 락 생명주기의 물리적 불일치 메커니즘**을 분석했습니다.

<br>

## 2. 물리 계층 딥다이브: 무엇이 동시성을 무너뜨리는가?

<img width="1422" height="1012" alt="image" src="https://github.com/user-attachments/assets/321583a1-1cc6-4f28-a39e-96ee27a51a78" />

### 1) 결함 1: Spring AOP 프록시 순서와 락 조기 해제

가장 빈번하게 발생하는 동시성 버그의 원인은 **스프링의 `@Transactional` AOP 프록시와 커스텀 분산 락 AOP 프록시 간의 동작 순서 역전**에 있습니다.

```
[ AS-IS 프록시 실행 흐름 ]
1. Client ──> DistributedLockAop (락 획득: lock.tryLock())
2.               └──> TransactionInterceptor (트랜잭션 시작: Connection 점유)
3.                       └──> 비즈니스 로직 실행 (재고 UPDATE)
4.               <── [비즈니스 로직 종료]
5.            DistributedLockAop (락 해제: lock.unlock()) ⚠️ [위험 구간 발생]
6. ──> TransactionInterceptor (물리 DB Commit 및 Connection 반납)
```

- **원인:** 커스텀 분산 락 AOP가 메서드 실행 직후 락을 먼저 해제(`unlock()`)한 뒤 바깥쪽의 `TransactionInterceptor`가 DB에 물리 `commit`을 호출합니다.
- **물리적 결과:** 락이 풀린 시점(5번)부터 실제 DB 커밋이 완료되는 시점(6번) 사이에 미세한 시간 공백이 발생합니다.
- 이때 대기 중이던 타 스레드가 락을 획득하고 DB를 조회하면 아직 디스크에 반영되지 않은 과거 재고를 읽게 됩니다. 결과적으로 두 스레드가 동일한 재고를 차감하여 **초과 판매**가 발생합니다.

### 2) 결함 2: 다중 락 키 획득 순서 미정렬과 순환 교착

장바구니 주문이나 복수 상품 동시 결제 시 여러 상품의 락을 한 번에 잡아야 하는 상황이 발생합니다.

- **원인:** 스레드 A는 상품 `[B, A]`를 주문하고 스레드 B는 상품 `[A, B]`를 주문합니다.
- **물리적 결과:** 스레드 A가 Redis에서 키 `B`를 선점하고 키 `A`를 요청하는 순간 스레드 B는 키 `A`를 선점하고 키 `B`를 요청합니다.
- 두 스레드는 서로가 쥐고 있는 락이 풀리기만을 기다리는 **순환 대기**에 빠집니다. Redisson의 기본 `waitTime` 동안 스레드가 풀리지 않아 톰캣 스레드 200개가 `TIMED_WAITING` 상태로 고사합니다.

### 3) 결함 3: 외부 API I/O의 DB 트랜잭션 침투와 연쇄 장애

- **원인:** PortOne 등 외부 PG 결제 검증 API 호출(HTTP 통신, 500ms 지연)이 `@Transactional` 스코프 내부에 위치했습니다.
- **물리적 결과:** 결제 검증이 끝날 때까지 10개의 가용 `HikariCP` 커넥션이 아무런 연산 없이 단순 소켓 I/O 블로킹 상태로 점유됩니다.
- 커넥션 풀이 0ms 만에 고갈되면서 주문과 무관한 일반 사용자의 로그인 및 상품 조회 요청마저 Spring Security 필터의 유저 조회 단계에서 `ConnectionTimeoutException`을 유발하며 시스템 전체가 다운되는 연쇄 장애가 발생했습니다.

<br>

## 3. TO-BE: 트랜잭션 격리와 Fail-Fast 아키텍처 설계

위의 3대 물리적 결함을 해결하기 위해 락과 트랜잭션의 생명주기를 물리적으로 분리하고 외부 통신망을 서킷브레이커로 격리했습니다.

<img width="1682" height="1541" alt="image" src="https://github.com/user-attachments/assets/552cc111-facc-42e6-8ab1-e1b1c1caa6eb" />

### 1) SpEL 파싱 기반 ID 오름차순 정렬 락 (`DistributedLockAop`)

- **개선 메커니즘:** AOP에서 SpEL 표현식을 파싱해 추출한 락 대상 상품 ID 목록을 `Collections.sort()`로 오름차순 정렬(`[ID_1, ID_2]`)한 뒤 Redisson `MultiLock`을 생성하도록 구현했습니다.
- **물리적 이점:** 어떤 스레드가 어떤 순서로 요청하더라도 Redis 상의 락 획득 순서가 항상 동일하게 정규화되므로 **순환 대기 데드락이 구조적으로 배제**됩니다.
- **성능 튜닝 (Fast-Path):** 단일 상품 주문 시 불필요한 `MultiLock` 오버헤드를 줄이고 단일 `RLock`으로 우회하여 Redis CPU 점유율을 4.16%에서 2.15%로 절감했습니다.

### 2) 트랜잭션 생명주기 격리 (`AopForTransaction` REQUIRES_NEW)

락과 트랜잭션의 실행 순서를 정상화하기 위해 별도의 트랜잭션 위임 컴포넌트를 설계했습니다.

```java
// TO-BE: 트랜잭션 커밋 완료 후 락 해제를 보장하는 구조
@Aspect
@Component
@Order(1) // 락 AOP가 트랜잭션보다 바깥쪽에서 실행되도록 순서 보장
public class DistributedLockAop {

    private final AopForTransaction aopForTransaction;

    @Around("@annotation(distributedLock)")
    public Object lock(ProceedingJoinPoint joinPoint, DistributedLock distributedLock) throws Throwable {
        RLock lock = getSortedLock(joinPoint, distributedLock);
        boolean isLocked = lock.tryLock(distributedLock.waitTime(), distributedLock.leaseTime(), TimeUnit.MILLISECONDS);

        if (!isLocked) {
            throw new CustomException(ErrorCode.LOCK_ACQUISITION_FAILED); // 0ms Fail-Fast
        }

        try {
            // REQUIRES_NEW로 별도 물리 트랜잭션을 실행하고 물리 커밋 완료를 보장받음
            return aopForTransaction.proceed(joinPoint);
        } finally {
            try {
                lock.unlock(); // DB 물리 커밋이 완료된 후 락 해제
            } catch (IllegalMonitorStateException ignored) {
                // 이미 타임아웃으로 만료된 경우 방어
            }
        }
    }
}
```

```java
@Component
public class AopForTransaction {
    // 반드시 독립적인 새 물리 트랜잭션으로 커밋까지 완료 후 반환
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public Object proceed(ProceedingJoinPoint joinPoint) throws Throwable {
        return joinPoint.proceed();
    }
}
```

- **물리적 이점:** `AopForTransaction`이 `REQUIRES_NEW`를 통해 물리 커밋을 완료하고 반환한 뒤에만 바깥쪽 AOP의 `finally` 블록에서 `lock.unlock()`이 호출됩니다.
- **결과:** 다른 스레드가 락을 획득하는 순간에는 항상 최신 커밋된 재고가 조회되므로 **초과 판매 오류를 구조적으로 배제**했습니다.

### 3) 외부 결제 연동 분리 및 Read/Write 서킷브레이커 격리

- **트랜잭션 외부 분리:** 500ms가 소요되는 외부 결제 조회를 `@Transactional` 바깥으로 이동시켜 실제 DB 커넥션을 점유하는 시간을 순수 재고 UPDATE 및 INSERT가 일어나는 수 ms 구간으로 축소했습니다.
- **Read/Write 서킷 분리:** Resilience4j를 적용해 조회 전용 `paymentReadClient`와 승인 전용 `paymentWriteClient`의 서킷 스코프를 물리적으로 분리했습니다.
    - 외부 PG사의 조회 API 장애 시 Read 서킷만 즉시 `OPEN`되어 **0ms Fail-Fast**로 톰캣 스레드를 보호합니다.
    - 이때도 실제 결제 승인/취소 회로(Write)는 정상 작동하여 **1.09ms** 만에 결제를 정상 수용하는 독립 가용성을 확보했습니다.

<br>

## 4. 실측 벤치마크 및 계측 데이터 검증

자원 제한 환경(WAS 0.8 vCPU / 1.5 GiB, DB 1.0 vCPU / 2.0 GiB, HikariCP 10)에서 k6를 통해 **500명의 가상 사용자(VUs)로 5분간 1,000 TPS 급 피크 부하**를 가하고 Scouter APM으로 계측한 정량적 성능 지표입니다.

| 지표 항목 | AS-IS (단일 트랜잭션 결합) | TO-BE (정렬 락 + 서킷 격리) | 개선 효과 및 의의 |
| --- | --- | --- | --- |
| **총 처리 트랜잭션** | 19,307회 | **185,692회** | **처리량 약 9.6배 (961.7%) 향상** |
| **평균 응답 속도** | 6,960 ms | **83.03 ms** | **응답 지연 98.8% 단축** |
| **중앙값 응답 속도** | 7,590 ms | **1.09 ms** | **실제 사용자 대기 시간 99.9% 단축** |
| **P95 응답 지연** | 9,000 ms 이상 (타임아웃) | **Read `432ms` / Write `513ms`** | **피크 부하 구간 500ms 대 방어** |
| **평균 처리율 (TPS)** | 50 ~ 70 TPS | **평균 627.39 TPS** | **가용 대역폭 약 9배 확장** |
| **HikariCP 커넥션 에러** | **수백 건의 Timeout 고갈** | **0건** | **커넥션 점유 시간 단축으로 고갈 해결** |
| **WAS CPU 점유율 (`docker stats`)** | 97.13% (스레드 병목 포화) | **48.13% (안정 상태)** | **유휴 대기 제거 및 CPU 부하 안정화** |
| **MySQL CPU 점유율 (`docker stats`)** | 3.53% (락 대기 정체) | **0.56%** | **DB 불필요 경합 제거 및 부하 84% 절감** |
| **Scouter Active 스레드** | 200개 포화 후 Hanging (정체) | **최대 3개 (`RUNNABLE`) 이하 회수** | **스레드 자원 누수 및 고갈 해결** |
| **초과 판매 수량** | 초과 판매 발생 (정합성 파괴) | **0건 (오차율 0.00%)** | **18.5만 건 중 재고 오차율 0.00% 통제** |
| **시스템 가** | 0.00% (인증 연쇄 마비로 전면 실패) | **100.00% (Error Rate 0.00%)** | **외부 장애 전이 차단 및 비즈니스 연속성 유지** |

```
[AS-IS k6 & Scouter APM]
► Active Threads = 200 (TIMED_WAITING 정체)
► HikariCP State = 10 Active (ConnectionTimeout 다발)
► Avg Latency    = 6,960 ms
► Result         = 초과 판매 발생 및 시스템 가용성 0%

[TO-BE k6 & Scouter APM]
► Active Threads = 3 이하 (RUNNABLE 실시간 회수)
► HikariCP State = 0~2 Active (수 ms 단위 초단기 순환)
► Avg Latency    = 83.03 ms (Median 1.09 ms)
► Result         = 185,692건 무결성 처리 (초과 판매 0건, 가용성 100%)
```

<br>

## 5. 트레이드오프와 런타임 한계: 분산 락 도입 시 통제 요소

1. **`REQUIRES_NEW`에 따른 커넥션 풀 이중 점유 리스크:**
    - 바깥쪽 트랜잭션이 살아있는 상태에서 `REQUIRES_NEW`를 호출하면 단일 스레드가 동시에 2개의 DB 커넥션을 점유하게 되어 커넥션 고갈이 발생할 수 있습니다.
    - **대응:** 바깥쪽 서비스 메서드에서는 `@Transactional`을 제거하고 순수 비즈니스 상태 변경이 일어나는 내부 지점만 `AopForTransaction`으로 진입하도록 스코프를 엄격히 제한했습니다.
2. **락 대기 시간(`waitTime`)과 사용자 경험 간의 트레이드오프:**
    - 락 대기 시간(`waitTime`)을 길게 잡으면 재고를 얻을 확률은 늘어나지만 톰캣 스레드가 장시간 점유됩니다. 반대로 너무 짧으면 409 Conflict 예외가 증가합니다.
    - **대응:** 선착순 한정판 도메인 특성에 맞춰 `waitTime`을 **50ms(또는 0ms Fail-Fast)**로 짧게 튜닝하여 경쟁에서 밀린 요청은 0ms 만에 즉시 409를 반환하고 톰캣 스레드를 다른 사용자 요청으로 회수했습니다.
3. **Redis 단일 장애점(SPOF) 리스크:**
    - 분산 락은 Redis 클러스터의 가용성에 의존합니다. Redis 인스턴스 다운 시 주문 전체가 중단될 수 있습니다.
    - **대응:** 실제 운영 환경에서는 Redis Sentinel 또는 Cluster 구성을 통해 고가용성을 확보하고 필요 시 비상 모드로 RDBMS 비관적 락(`PESSIMISTIC_WRITE`)으로 자동 우회하는 Fallback 전략을 수립해야 합니다.

<br>

## 6. 결론: 분산 락의 환상을 넘어 트랜잭션 물리 경계를 설계하라

많은 개발자들이 분산 락(`Redisson`) 라이브러리를 메서드에 감싸기만 하면 동시성 문제가 자동으로 해결될 것이라 예상합니다.

그러나 실무 고부하 환경에서의 동시성 제어는 단순 락 적용을 넘어 AOP 프록시 순서, 데이터베이스 물리 커밋 시점, 외부 네트워크 I/O의 트랜잭션 격리, 락 획득 키의 정렬이라는 물리적 실행 흐름을 통제하는 데 있습니다.

프레임워크의 편리한 애노테이션 뒤에 숨겨진 물리 계층(스레드, 커넥션, 네트워크 소켓)의 생명주기를 정밀하게 일치시킬 때 1,000 TPS 급 피크 트래픽 환경에서도 데이터 오차 없이 안정적으로 동작하는 시스템을 구축할 수 있습니다.
