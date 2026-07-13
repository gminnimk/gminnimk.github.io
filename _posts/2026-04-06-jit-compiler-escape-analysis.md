---
title: 자바 객체는 무조건 힙에 생성될까? 스칼라 치환과 어셈블리로 증명한 메모리 최적화
date: 2026-04-06 20:30:00 +0900
categories: [Java, JVM]
tags: [jit, jvm, optimization, escape-analysis, performance, assembly]
description: JIT 컴파일러가 객체를 해체하고 레지스터에 할당하는 스칼라 치환의 원리를 데이터와 기계어 수준에서 분석합니다.
---

## 자바 객체는 무조건 힙에 생성될까? 스칼라 치환과 어셈블리로 증명한 메모리 최적화

### 1. 객체지향의 딜레마와 메모리 성능

프로젝트에 도메인 주도 설계(DDD)와 불변 객체 패턴을 적용하며 상태 변경 시마다 새로운 객체를 생성해 반환하는 방식을 사용했습니다. 안정성과 가독성은 확보했으나 고부하 환경에서의 성능 저하 요인을 식별했습니다.

'수천 번 반복되는 루프 안에서 매번 new 키워드로 단명 객체를 생성해도 괜찮을까? 잦은 힙 메모리 할당과 GC에 의한 Stop-The-World 현상이 시스템 병목을 유발하지 않을까?'

과거에는 객체 풀 패턴이나 원시 타입 나열 같은 방어적 프로그래밍으로 이를 회피했습니다. 하지만 유지보수성을 포기하고 성능만 좇는 설계는 장기적인 리스크를 동반합니다. '객체 생성이 곧 성능 저하'라는 추측에 의존하기보다 JVM 런타임 내부 분석을 통해 설계의 근거를 확보하기로 했습니다.

<br>

### 2. JVM의 판단: 탈출 분석과 스칼라 치환

Java 객체는 기본적으로 힙에 할당됩니다. 그러나 현대 JVM의 C2 JIT 컴파일러 환경에서는 항상 성립하지 않습니다. JIT 컴파일러는 코드 형태를 넘어 런타임 실행 흐름을 추적하는 탈출 분석을 수행합니다.

탈출 분석은 메서드 내부에서 생성된 객체의 스코프 이탈 여부를 판단합니다.

* **Global-Escape (전역 탈출):** 객체가 정적 필드나 메서드 반환값으로 사용되어 외부에서 접근 가능한 상태
* **Arg-Escape (인자 탈출):** 객체가 다른 메서드의 인자로 전달되는 상태
* **No-Escape (탈출 불가):** 객체가 생성된 메서드 내부에서만 소비되는 상태

컴파일러가 특정 객체를 No-Escape로 판별하면 JVM은 해당 객체를 스택에 올리는 것을 넘어 필드를 원시 타입(Scalar) 단위로 분해합니다. 이를 CPU 레지스터나 지역 변수로 분산시키는 과정을 스칼라 치환이라 합니다

> <img width="1650" height="354" alt="image" src="https://github.com/user-attachments/assets/fe7cc8b5-0559-442f-b9bf-39666fe98775" />
> 
> *스칼라 치환이 적용되면 객체의 헤더가 제거되고 필드 데이터만 레지스터에 할당됩니다.*

결과적으로 힙 할당이 생략되어 GC 대상 자체가 발생하지 않습니다.

<br>

### 3. JMH 마이크로 벤치마크 실험

스칼라 치환의 메모리 영향을 확인하기 위해 JMH(Java Microbenchmark Harness)로 실험 환경을 구축했습니다.

```java
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@State(Scope.Thread)
public class EscapeAnalysisBenchmark {

    static class Point {
        int x, y;
        Point(int x, int y) { this.x = x; this.y = y; }
    }

    private int a = 10;
    private int b = 20;

    @Benchmark
    public void testScalarReplacement(Blackhole blackhole) {
        Point p = new Point(a, b);
        int result = p.x + p.y;
        blackhole.consume(result); 
    }
}

```

JMH의 `-prof gc` 옵션으로 탈출 분석 활성화 여부에 따른 지표를 추출했습니다.

| **실험 케이스** | **JVM Options** | **평균 응답 시간** | **작업당 힙 할당량 (B/op)** | **GC 발생 횟수 (1분당)** |
| --- | --- | --- | --- | --- |
| **Case A (최적화 OFF)** | `-XX:-DoEscapeAnalysis` | `3.007 ns/op` | `24.000 B/op` | `704 counts` |
| **Case B (최적화 ON)** | `-XX:+DoEscapeAnalysis` | `0.938 ns/op` | `≈ 10⁻⁶ B/op` | `≈ 0 counts` |

Case A에서는 Point 객체 생성 시마다 메타데이터와 필드를 포함해 약 24바이트의 단명 객체가 Eden 영역에 쌓였습니다. 반면 탈출 분석이 활성화된 Case B에서는 할당률이 0바이트로 수렴했습니다. 객체 생성 생략으로 메모리 접근 비용이 절감되어 응답 시간이 단축되었습니다.

<br>

### 4. 기계어 수준의 확인: 어셈블리 코드 분석

`hsdis` 플러그인을 활용해 JIT 컴파일러가 생성한 x86 어셈블리 코드를 분석했습니다.

**최적화 이전: 힙 할당 방식 (Escape Analysis OFF)**

객체 생성을 위해 TLAB(Thread Local Allocation Buffer) 여유 공간을 확인하고 객체 헤더를 설정합니다.

```assembly
; TLAB 공간 확인
0x00007f...: mov    r10, QWORD PTR[r15+0x60]   
0x00007f...: lea    r11, [r10+0x18]             ; 24바이트(헤더+필드) 할당 계산
0x00007f...: cmp    r11, QWORD PTR [r15+0x70]   ; 공간 부족 시 느린 할당 호출

; 객체 헤더 및 필드 세팅
0x00007f...: mov    QWORD PTR [r10], 0x1        ; Mark Word 초기화
0x00007f...: mov    DWORD PTR [r10+0x8], 0x...  ; Klass Pointer 설정
0x00007f...: mov    DWORD PTR[r10+0xc], r8d    ; x 값 할당
0x00007f...: mov    DWORD PTR[r10+0x10], r9d   ; y 값 할당

```

`new Point()` 호출 시 메모리 공간 계산과 클래스 메타데이터 설정 등 관리 비용이 발생합니다.

**최적화 이후: 스칼라 치환 적용 (Escape Analysis ON)**

탈출 분석 적용 코드는 다음과 같습니다.

```assembly
; 객체 할당 및 헤더 설정 코드 소거
0x00007f...: mov    eax, DWORD PTR [r12+0x10]   ; 변수 a를 eax 레지스터로 로드
0x00007f...: add    eax, DWORD PTR[r12+0x14]   ; 변수 b를 더함 (x + y)

```

힙 메모리 할당(`lea`, `cmp`) 명령어가 제거되었습니다. 컴파일러는 `Point` 객체 구조를 런타임에 불필요하다고 판단해 필드 값만 레지스터(`eax`)로 가져와 연산합니다.

<br>

### 5. 스칼라 치환이 실패하는 경우

JIT 컴파일러의 최적화가 모든 상황에 적용되지는 않습니다.

1. **객체 크기 초과:** 치환할 필드가 많아 레지스터나 스택 프레임에 적재 불가능한 경우 (통상 64바이트 초과 등 JVM 옵션 종속)
2. **복잡한 제어 흐름:** 객체 생성 및 사용 위치 사이에 루프나 조건문이 얽혀 생명주기를 정적으로 분석하기 어려운 경우

코드가 비대하거나 복잡할수록 컴파일러의 최적화 이점을 누리기 어렵습니다.

<br>

### 6. 결론: 런타임 환경 중심의 설계

애플리케이션 레벨에서 메모리 효율을 위해 억지로 객체를 재사용하거나 코드를 복잡하게 만드는 것은 오히려 컴파일러의 최적화를 방해합니다.

성능 확보를 위해 무작정 객체 생성을 피하기보다 메서드를 작고 응집도 있게 유지하여 JIT 컴파일러가 분석하기 좋은 코드를 작성해야 합니다. 대규모 트래픽 시스템 아키텍처 설계 시 프레임워크 기능을 넘어 코드가 실행되는 런타임 환경 특성을 식별하고 통제하는 엔지니어링 접근이 필수적입니다.

