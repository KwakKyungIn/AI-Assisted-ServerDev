## 📦 1) Project Overview

> 본 사례는 [IocpChatServer](https://github.com/KwakKyungIn/portfolio-kwak/blob/main/IocpChatServer/README.md) 프로젝트 내
> Room::Broadcast() 구간의 락 범위 및 비동기 처리 구조를 AI를 통해 개선한 사례입니다.

### 🧪 Test Environment

* Clients: DummyClient 500–1000
* Rooms: 10
* Traffic: 초당 5–10 k packets 수준의 브로드캐스트 부하
* Throughput 기준: 약 250 k broadcasts/s
* Metrics: Broadcast latency (p50/p90/p99), SG efficiency

---

## ⚙️ 2) 문제 발생 배경

IOCP 기반의 브로드캐스트 구조는 전반적으로 안정적인 성능을 보였으나,
지연 시간 분포(p90, p99) 구간에서 간헐적인 이상치가 관측되었습니다.
평균 지연(p50)은 일정하게 유지되었지만, 상위 구간에서 순간적인 지연이 발생하는 패턴이 확인되었습니다.

📈 관측 지표

* 브로드캐스트 지연 시간: p50 = 1.7ms / p90 = 2.9ms / p99 = 4.2ms
* JobQueue 등록 실패율: 약 4~5% 수준으로 증가

이 현상은 전체 구조의 문제라기보다는,
일시적인 병목이나 큐 처리 구간의 지연 누적에서 비롯된 것으로 판단되었습니다.

이후 원인을 더 명확히 확인하기 위해 AI를 활용하였습니다.
분석 대상은 Room::Broadcast() 함수였으며,
특히 내부의 `READ_LOCK` 구간이 지나치게 넓게 설정되어 있는 점에 주목했습니다.
락은 경량 스핀락 기반의 공유 뮤텍스(shared-mutex) 형태였지만,
`_jobQueue.TryEnqueue()` 호출까지 락이 유지되면서
락 범위 내에서 비동기 큐 접근이 지연될 가능성이 있었습니다.

---

### 💭 AI에게 제시한 핵심 질문

성능 저하의 원인을 검증하기 위해 다음과 같은 질문을 제시했습니다.

1. “`READ_LOCK`의 범위를 축소해도 동시성 안전성이 유지될 수 있는가?”

   * Room 내부 상태를 안전하게 유지하면서 JobQueue 등록 구간을 락 외부로 이동할 수 있는지 확인.

2. “락을 최소화하거나 제거한 JobQueue 설계가 가능한가?”

   * Enqueue 시점의 락 경합을 줄이거나, 비동기 큐 구조로 대체할 수 있는지 검토.

AI는 코드의 흐름을 분석한 뒤,
플레이어 목록을 수집하는 구간만 Read Lock으로 보호하고
JobQueue 처리와 Fallback 전송은 락을 해제한 후 실행하는 구조를 제안했습니다.
이 방식은 코드의 동시성 안전성을 유지하면서도,
락 경합으로 인한 지연을 줄일 수 있는 방향이었습니다.

---

## 🧠 3) AI 활용 과정 및 개선 결과

AI는 초기 코드를 분석한 뒤,
`READ_LOCK`의 적용 범위가 지나치게 넓어 JobQueue 처리까지 락이 유지되는 점을 주요 원인으로 지적했습니다.
이로 인해 브로드캐스트 대상이 많을수록 스레드 간 경합이 발생하고,
일부 구간의 지연이 불규칙하게 증가할 가능성이 있다는 의견을 제시했습니다.

이에 따라 다음과 같은 개선 단계를 거쳤습니다.

---

### 🔹 1단계: 코드 논리 검증

먼저 AI에게 기존 코드의 동작 흐름과 락의 목적을 설명하고,
락 해제 시점을 변경했을 때 발생할 수 있는 데이터 경합 문제를 함께 검토했습니다.
AI는 “락의 목적이 Room 내부의 `_players` 보호에 한정된다면,
맵을 순회하는 시점까지만 락을 유지해도 안전하다”는 결론을 제시했습니다.

이를 통해 락 범위 축소가 논리적으로 안전함을 검증할 수 있었습니다.

---

### 🔹 2단계: 구조 개선 제안

AI는 다음과 같은 방향으로의 수정을 제안했습니다.

* `READ_LOCK`을 플레이어 목록을 수집하는 부분에만 적용
* 이후의 JobQueue 등록 및 직접 전송(fallback)은 락 해제 후 수행
* JobQueue 등록 실패 시, 즉시 Send() 호출로 대체 수행

이 방식은 락의 유지 시간을 줄여 성능을 높일 수 있는 구조였습니다.
다만 AI는 동시에 “락을 너무 이른 시점에 해제할 경우,
Room 내부 상태가 변하는 타이밍에 따라 예외 상황이 발생할 수 있다”는 점을 함께 경고했습니다.

이에 따라 단순히 락을 줄이는 데 그치지 않고,
성능 향상과 동시성 안정성의 균형을 함께 검증하는 과정을 진행했습니다.
AI를 통해 제시된 각 시나리오(락 해제 시점, JobQueue 접근 순서 등)에 대해
논리적 충돌 가능성을 테스트하며, 안정성을 확보한 형태로 최종 구조를 확정했습니다.

---

### 🔹 3단계: 개선 코드 구현

이 과정을 거쳐 Broadcast 함수는 다음과 같이 수정되었습니다.

수정 전
```cpp
void Room::Broadcast(SendBufferRef sendBuffer)
{
    READ_LOCK;

    std::vector<PacketSessionRef> targets;
    targets.reserve(_players.size());

    for (const auto& kv : _players)
        if (kv.second && kv.second->ownerSession)
            targets.push_back(kv.second->ownerSession);

    if (targets.empty())
        return;

    if (!_jobQueue.TryEnqueue([targets, sendBuffer]() {
        for (auto& sess : targets)
            sess->Send(sendBuffer);
    })) {
        for (auto& sess : targets)
            sess->Send(sendBuffer);
    }
}
```
수정 후
```cpp
void Room::Broadcast(SendBufferRef sendBuffer)
{
    std::vector<PacketSessionRef> targets;
    {
        READ_LOCK;
        for (const auto& kv : _players)
            if (kv.second && kv.second->ownerSession)
                targets.push_back(kv.second->ownerSession);
    }

    if (targets.empty()) return;

    GMetrics.broadcast_deliveries.fetch_add(targets.size(), std::memory_order_relaxed);

    if (!_jobQueue.TryEnqueue([targets, sendBuffer]() {
        for (auto& sess : targets)
            sess->Send(sendBuffer);
    })) {
        for (auto& sess : targets)
            sess->Send(sendBuffer);
    }
}
```

락 범위를 최소화함으로써,
`_jobQueue.TryEnqueue()` 실행 시점에는 Room 내부 락이 해제되어
다른 스레드의 접근 대기가 줄어들었습니다.
또한 `_players`의 상태가 변하지 않는다는 점을 검증함으로써
성능과 안전성을 모두 확보한 형태로 코드가 안정화되었습니다.

---

## 📊 4) 성능 결과 데이터

아래 표는 Room::Broadcast() 개선 전·후의 성능 비교 결과입니다.
테스트 조건은 동일하게 유지하였으며 (클라이언트 500명, 룸 10개, 초당 25만 회 브로드캐스트 부하)
단순한 처리량 향상보다 지연 구간의 안정성(p90 · p99) 개선에 중점을 두었습니다.

| 구분           |  io r/s |    io s/s |    bytes s/s | bc-deliv/s | avg (μs) | p50 (μs) | p90 (μs) | p99 (μs) |
| :----------- | ------: | --------: | -----------: | ---------: | -------: | -------: | -------: | -------: |
| 개선 전         | ≈ 5,050 | ≈ 115,000 | ≈ 11,400,000 |  ≈ 252,000 |    ≈ 520 |      240 |    1,320 |    7,900 |
| 개선 후 (AI 적용) | ≈ 5,080 | ≈ 113,000 | ≈ 11,900,000 |  ≈ 254,000 |    ≈ 460 |      190 |    1,000 |    4,800 |

---

### 📈 결과 요약

* 상위 지연 구간 안정화: p90 기준 약 24% 감소, p99 기준 약 31% 감소
* 평균 처리 지연: 520 → 460 μs (약 13% 감소)
* 처리량(Throughput): 거의 동일 수준 유지 (±1%)
* 전체 응답 분포: 스파이크 빈도 감소로 지연 곡선이 완만하게 수렴

요약하면, 락 스코프 최적화만으로도 처리량을 유지하면서 지연 편차를 크게 감소시킬 수 있었습니다.
이는 AI가 제시한 성능 향상과 안정성 균형의 방향이 유효했음을 보여줍니다.

---

### 🖼️ 참고 시각자료

아래 이미지는 개선 후 측정된 실제 로그 테이블 예시입니다.
![Broadcast Latency After](./assets/image_1.jpg)

---

## 📘 5) 기술적 분석

본 개선의 핵심은 락 범위 최소화(lock scope reduction)를 통해
스레드 간 불필요한 대기 구간을 제거하면서도
데이터 일관성을 유지하는 구조를 확보한 데 있습니다.

---

### 🔹 1. 병목 구간의 정량적 해석

개선 전의 Room::Broadcast() 함수는
`_jobQueue.TryEnqueue()` 호출까지 `READ_LOCK`이 유지되는 구조였습니다.
이는 다수의 스레드가 동시에 같은 Room 인스턴스의 브로드캐스트를 수행할 때,
락 점유 시간이 길어져 락 경합(Lock Contention)이 발생했습니다.

락이 해제되기 전까지 JobQueue 접근이 지연되면,
워크 스레드 간 큐 처리 순서가 미묘하게 어긋나면서
상위 지연 구간(p90, p99)의 스파이크로 나타났습니다.
이 지연은 평균에는 큰 영향을 주지 않았지만,
지연 분포의 불안정성으로 이어졌습니다.

---

### 🔹 2. 락 범위 축소의 구조적 효과

AI가 제안한 방식은 다음과 같은 흐름으로 동작합니다.

1. 락 범위를 `_players` 순회 구간으로 한정
   → Room 내부 상태만 보호하고, 이후 로직은 락 해제 후 수행
2. JobQueue 등록 및 송신은 비동기적으로 처리
   → 다른 스레드의 접근이 즉시 가능해짐
3. 락 해제 이후에도 데이터 일관성이 유지
   → `_players`의 Snapshot 형태로 세션 목록을 복사했기 때문

이 구조는 I/O 바운드 구간과 연산 구간을 명확히 분리하며,
락 유지 시간(Time-in-lock)을 짧게 만들어 경합 가능성을 실질적으로 줄였습니다.

---

### 🔹 3. 성능 지표 해석

성능 측정 결과에서 p90, p99 구간의 지연이 감소한 이유는 다음과 같습니다.

* 락 해제 시점 단축 → 브로드캐스트 대상 수집 후 즉시 해제됨
* JobQueue 처리 분산화 → 락 외부에서 비동기 실행
* 락 대기열 감소 → 동시에 여러 Room에서 브로드캐스트 수행 가능

이로 인해 평균 처리 시간(avg)과 고지연 분포(p90, p99)가 모두 안정화되었습니다.
Throughput은 기존과 거의 동일했으며, 이는 I/O 처리량 자체가 병목이 아니었음을 의미합니다.

---

### 🔹 4. 안전성 검증 결과

락 범위를 축소하면서도 안정성을 유지할 수 있었던 근거는 다음과 같습니다.

* `_players`의 상태는 락 구간 내에서 Snapshot으로 복사되므로,
  락 해제 이후에도 참조 무결성이 깨지지 않음
* JobQueue 내부는 자체적으로 스레드 안전하게 설계되어 있어
  락 해제 시점 이후 접근에도 문제가 없음
* AI와의 검증 과정에서 데이터 경쟁 가능성에 대한 시뮬레이션 로직을 추가하여,
  Race Condition이 발생하지 않음을 확인함

결과적으로, 락 스코프 축소에 따른 병목 제거와 데이터 무결성 유지를 모두 달성할 수 있었습니다.

---

### 🔹 5. 기술적 의의

이번 개선은 단순한 코드 최적화가 아니라,
IOCP 환경에서의 락 범위 설계 전략(lock granularity strategy)에 대한 실험적 검증입니다.
AI를 통한 논리 검증과 실험 반복을 통해,

* 병목 구간을 락 유지 시간 단축 관점에서 분석하고,
* 비동기 큐(JobQueue)의 독립성과 락의 목적 범위를 명확히 구분하며,
* 결과적으로 지연 분포의 안정화를 달성했습니다.

이 개선은 동일한 구조를 사용하는 다른 서버 모듈(Room, Channel, Guild 등)에
동일한 설계 원칙을 적용할 수 있는 근거가 되었습니다.

---
