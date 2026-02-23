+++
date = '2026-02-23'
draft = false
title = '[스터디] Kafka Consumer Offset 관리: Auto-commit의 함정과 올바른 처리 전략'
+++

Kafka consumer의 기본 설정은 놀라울 정도로 간단하다. 메시지를 꺼내서 처리하면 끝이다. 하지만 운영 환경에서 consumer가 crash하면, **이미 처리한 메시지가 다시 처리되는 중복 문제**를 겪게 된다.

이 글에서는 auto-commit이 왜 중복 처리를 유발하는지, 그리고 이를 어떻게 해결하는지 정리한다.

---

## Kafka는 Message Queue가 아니라 Distributed Log다

중복 처리 문제를 이해하려면 먼저 Kafka의 설계 철학을 알아야 한다.

### 전통적인 Message Queue (RabbitMQ, SQS)

```
Broker가 메시지별 상태를 관리

msg1: delivered → consumer A → ack 받음 → 삭제
msg2: delivered → consumer A → ack 받음 → 삭제
msg3: delivered → consumer A → ack 안 옴 → 다른 consumer에게 재전달
```

Broker가 **메시지 단위로** 전달 상태를 추적한다. consumer가 ack를 보내면 그 메시지를 처리 완료로 표시한다.

### Kafka

```
Broker는 로그만 유지, 메시지별 상태 추적 안 함

partition: [msg0][msg1][msg2][msg3][msg4][msg5][msg6]...
                                    ↑
                          consumer의 offset (= 책갈피)
```

Kafka broker는 **"이 consumer group은 이 partition의 offset N까지 읽었다"** 라는 숫자 하나만 `__consumer_offsets` topic에 저장한다. 개별 메시지의 처리 완료 여부는 전혀 모른다.

### 왜 이렇게 설계했을까?

**처리량(Throughput) 최적화** 때문이다.

```
RabbitMQ:
  메시지 전달 → 상태 변경(unacked) → ack 수신 → 상태 변경(acked) → 삭제
  메시지당 3번의 상태 변경 + 디스크 I/O

Kafka:
  consumer가 poll() → sequential read로 바이트 스트림 전송
  commit() → 숫자 하나 저장
  메시지 1,000개든 100,000개든 commit은 숫자 하나
```

메시지별 상태 추적을 포기하고, **sequential I/O + 단일 offset**이라는 구조로 초당 수백만 건의 처리량을 달성한 것이다.

대신 **"어디까지 처리했는지"는 consumer가 스스로 관리해야 한다.** 이 offset 관리를 제대로 하지 않으면 중복 처리가 발생한다.

---

## Auto-commit의 실제 동작

kafka-python의 `KafkaConsumer` 기본 설정은 다음과 같다:

```python
consumer = KafkaConsumer(
    "my-topic",
    enable_auto_commit=True,          # 기본값
    auto_commit_interval_ms=5000,     # 기본값: 5초
)
```

별도 설정 없이 consumer를 생성하면 **5초마다 자동으로 offset을 커밋**한다.

### Auto-commit이 커밋하는 offset은?

여기가 핵심이다. Auto-commit은 **"마지막으로 처리한 메시지"가 아니라 consumer의 내부 position**을 커밋한다.

Kafka consumer는 내부적으로 각 partition마다 `position`이라는 값을 관리한다. 이 값은 **"다음에 fetch할 offset"**을 의미하며, auto-commit은 이 position을 그대로 커밋한다.

```
consumer 내부 상태:
  TopicPartition("my-topic", 0) → position = 6

auto-commit → offset 6을 __consumer_offsets에 커밋
            → "다음 소비는 offset 6부터" 라는 의미
```

kafka-python의 iterator(`for msg in consumer`)를 사용하면, position은 **메시지가 yield될 때마다** 갱신된다:

```
for message in consumer:
  msg1 yield → position = 2
  msg2 yield → position = 3
  ...
  msg5 yield → position = 6
```

그리고 auto-commit은 다음 메시지를 가져오는 시점(`next()` → 내부 `_poll_once()`)에서만 발동한다. 즉 **이전 메시지의 처리가 완료된 후에 커밋**되므로, position과 실제 처리 상태가 일치한다.

문제는 **auto-commit 주기(기본 5초) 사이에 crash가 발생하면**, 마지막 커밋 이후 처리한 메시지들의 offset이 커밋되지 않는다는 것이다.

---

## Auto-commit의 함정: crash 시 중복 처리

auto-commit=true인 상태에서 consumer가 crash하면 어떤 일이 발생하는지 살펴보자.

### crash 시나리오

```
for msg in consumer:  (auto_commit_interval_ms=5000)
  msg1 yield → position = 2 → 처리 ✅
  msg2 yield → position = 3 → 처리 ✅
  (next 호출 시 5초 경과 → auto-commit → offset 3 커밋)
  msg3 yield → position = 4 → 처리 ✅
  msg4 yield → position = 5 → 처리 ✅
  msg5 yield → position = 6 → 처리 💥 crash
  ─── 다음 auto-commit 전에 crash ───

재시작 → offset 3부터 소비 → msg3, msg4, msg5 다시 수신
→ msg3, msg4는 이미 처리했는데 중복 처리됨
```

**auto-commit이 마지막으로 커밋한 offset 이후의 모든 메시지가 재처리**된다.

### 중복 범위를 결정하는 요인

중복 범위는 **마지막 auto-commit 시점부터 crash 시점까지 처리한 메시지 수**다.

| 요인 | 중복 범위에 미치는 영향 |
|------|----------------------|
| `auto_commit_interval_ms` | 주기가 길수록 중복 범위 ↑ |
| 메시지 처리 속도 | 빠를수록 같은 주기 내 더 많은 메시지 처리 → 중복 범위 ↑ |

`auto_commit_interval_ms`를 줄이면 중복 범위가 줄어들지만, **아무리 줄여도 근본 문제는 동일**하다. 1초든 30초든 그 윈도우 안에서 crash가 발생하면 중복이 발생한다. 정확성이 필요하다면 주기를 조정하는 것이 아니라 manual commit으로 전환해야 한다.

---

## 해결하기: Manual Commit + Idempotent Handler

중복 처리 문제의 해결은 두 단계로 나뉜다. **Manual commit으로 중복 범위를 최소화**하고, **idempotent handler로 남은 중복을 안전하게 처리**하는 것이다.

### 1단계: Manual Commit으로 중복 범위 줄이기

```python
consumer = KafkaConsumer(
    "my-topic",
    enable_auto_commit=False,     # auto-commit 비활성화
)

for message in consumer:
    process(message)              # 처리 먼저
    consumer.commit()             # 성공 후 커밋
```

같은 상황에서의 동작:

```
msg1 처리 ✅ → commit(offset=2)
msg2 처리 ✅ → commit(offset=3)
msg3 처리 💥 crash

재시작 → offset 3부터 → msg3부터 재처리
→ 중복 범위 = 최대 1건 (현재 처리 중인 메시지)
```

auto-commit에서는 중복 범위가 수십~수백 건이었지만, 메시지마다 commit하면 **최대 1건**으로 줄어든다.

commit 단위에 따라 중복 범위가 달라진다:

| commit 방식 | crash 시 중복 범위 |
|-------------|-------------------|
| 메시지마다 commit | 최대 1건 |
| N건마다 batch commit | 최대 N건 |
| poll() 단위 commit | 최대 `max_poll_records`건 |

### 2단계: Idempotent Handler로 중복을 안전하게

manual commit도 완벽하지 않다. **처리 성공 후 commit 직전에 crash**하면 그 1건이 재처리된다:

```
msg1 처리 ✅ → commit ✅
msg2 처리 ✅      ← DB에 저장 완료
💥 crash           ← commit 전에 죽음

재시작 → msg2 다시 소비 → 또 처리됨 (중복)
```

이 1건의 중복을 안전하게 만드는 것이 idempotency다. 실전에서 자주 쓰이는 세 가지 패턴을 소개한다.

#### 1) 업스트림에서 Idempotency Key 강제

모든 중복 방지의 출발점은 **메시지를 고유하게 식별할 수 있는 키**다. 이벤트에 `event_id`(UUID)를 포함하거나, `(topic, partition, offset)` 조합을 논리적 키로 사용한다.

```python
# Producer 측에서 event_id를 메시지에 포함
producer.send("orders", value={
    "event_id": str(uuid4()),       # 고유 식별자
    "order_id": order.id,
    "amount": order.amount,
})
```

이 키를 다운스트림(DB, 외부 API)에 요청을 보낼 때도 함께 전달하면, 전체 파이프라인에서 중복을 추적할 수 있다.

#### 2) DB 레벨에서 중복 방지를 원자적으로

가장 강력한 패턴은 **DB의 unique constraint로 중복 삽입 자체를 차단**하는 것이다.

```python
# ❌ Non-idempotent: 중복 실행 시 데이터 불일치
def handle(message):
    balance = get_balance(account_id)
    update_balance(account_id, balance + message.amount)  # 중복 시 2배 증가

# ✅ Idempotent: unique constraint + 조건부 후속 처리
def handle(message):
    # event_id 컬럼에 UNIQUE 제약 조건 필요
    created = EventLog.objects.get_or_create(
        event_id=message.event_id,          # 중복 시 생성 안 됨
        defaults={"payload": message.value}
    )[1]

    if created:  # 삽입 성공했을 때만 후속 처리
        update_balance(account_id, message.amount)
        send_notification(account_id)
```

DB별 중복 방지 구문:

| DB | 구문 |
|----|------|
| **PostgreSQL** | `INSERT ... ON CONFLICT DO NOTHING` |
| **MySQL** | `INSERT IGNORE` 또는 `ON DUPLICATE KEY UPDATE` |
| **Django ORM** | `get_or_create()` / `update_or_create()` |

핵심은 **"삽입 성공 여부"로 후속 처리를 분기**하는 것이다. 컨슈머가 몇 번을 재처리하든 DB가 한 번만 반영한다.

#### 3) Outbox 패턴: 외부 사이드이펙트가 있는 경우

이벤트 처리 결과로 **외부 시스템에 이벤트나 사이드이펙트**(알림 발송, 다른 토픽 produce 등)가 나가야 한다면, Outbox 패턴이 유효하다.

```python
def handle(message):
    with transaction.atomic():
        # 1. 비즈니스 로직 + outbox 레코드를 같은 트랜잭션에 기록
        order = process_order(message)
        OutboxEvent.objects.create(
            event_id=message.event_id,
            topic="order-completed",
            payload=serialize(order),
        )

    # 2. 별도 퍼블리셔가 outbox 테이블을 폴링하여 발행
    #    (또는 CDC로 자동 발행)
```

```
DB 트랜잭션 경계
┌──────────────────────────────┐
│ 비즈니스 로직  +  outbox 기록  │ ← 원자적
└──────────────────────────────┘
              ↓
    별도 퍼블리셔가 outbox → Kafka 발행
```

DB 트랜잭션이 커밋되지 않으면 outbox도 기록되지 않으므로, **중복/재시도/장애 복구 시에도 사이드이펙트의 정합성이 보장**된다.

### Producer를 사용하는 경우: broker ack 확인 후 commit

consume → produce → commit 패턴에서는 **broker가 메시지를 수신했음을 확인한 후** offset을 커밋해야 한다:

```python
for message in consumer:
    result = process(message)
    producer.send(output_topic, result)   # 비동기! 아직 전송 안 됐을 수 있음
    producer.flush()                      # broker ack 대기
    consumer.commit()                     # ack 확인 후 커밋
```

ack 확인 없이 commit하면, 메시지가 broker에 도착하기 전에 offset이 커밋되어 **메시지가 유실**될 수 있다.

> **참고**: `flush()`의 실제 보장 수준은 producer의 `acks` 설정에 따라 다르다. `acks=all`이면 ISR 전체 복제 완료를 보장하고, `acks=1`이면 leader 저장만 보장한다.

---

## 네 가지 Offset 관리 전략

여기까지의 내용을 바탕으로, Kafka에서 사용할 수 있는 네 가지 offset 관리 전략을 비교한다.

### 1. Auto-commit (기본값)

```python
consumer = KafkaConsumer("my-topic")  # enable_auto_commit=True가 기본

for message in consumer:
    process(message)
```

| 항목 | 값 |
|------|-----|
| 보장 수준 | At-least-once (crash 시 중복 발생) |
| 중복 범위 | auto-commit 주기 내 처리 메시지 수 |
| 처리량 | 최고 |
| 적합한 경우 | 로그 수집, 메트릭, 실시간 대시보드 |

### 2. 메시지별 수동 commit

```python
consumer = KafkaConsumer("my-topic", enable_auto_commit=False)

for message in consumer:
    process(message)
    consumer.commit()
```

| 항목 | 값 |
|------|-----|
| 보장 수준 | **At-least-once** |
| 중복 범위 | 최대 1건 |
| 처리량 | 낮음 (매 메시지마다 broker 왕복) |
| 적합한 경우 | **일반적인 비즈니스 로직 (가장 보편적)** |

### 3. Batch 수동 commit

```python
consumer = KafkaConsumer("my-topic", enable_auto_commit=False, max_poll_records=100)

count = 0
for message in consumer:
    process(message)
    count += 1
    if count % 50 == 0:
        consumer.commit()
```

| 항목 | 값 |
|------|-----|
| 보장 수준 | At-least-once |
| 중복 범위 | 최대 batch 크기 |
| 처리량 | 중간 |
| 적합한 경우 | 높은 처리량이 필요하면서 일부 중복은 허용 가능 |

### 4. Transactional (Exactly-once)

```python
# confluent-kafka 기준
producer.init_transactions()

for message in consumer:
    producer.begin_transaction()
    result = process(message)
    producer.produce(output_topic, result)
    producer.send_offsets_to_transaction(
        consumer.position(consumer.assignment()),
        consumer.consumer_group_metadata()
    )
    producer.commit_transaction()
```

| 항목 | 값 |
|------|-----|
| 보장 수준 | **Exactly-once** |
| 중복 범위 | 없음 |
| 처리량 | 가장 낮음 |
| 제약 | **Kafka-to-Kafka에서만 동작**. DB 쓰기는 보장 불가 |
| 적합한 경우 | Kafka consume → 변환 → Kafka produce 파이프라인 |

---

## "Kafka → DB 저장"에서 Exactly-once가 필요하다면

많은 애플리케이션은 Kafka에서 메시지를 소비하고 DB에 저장한다. 이 경우 Kafka의 transactional exactly-once만으로는 부족하다.

Kafka의 transactional exactly-once는 **offset commit과 Kafka produce를 하나의 트랜잭션으로 묶는 것**이다. DB 쓰기는 이 트랜잭션 범위 밖이다.

```
              Kafka Transaction 경계
             ┌─────────────────────┐
offset commit│                     │producer send  ← exactly-once 보장
             └─────────────────────┘
                       ↑
                  DB 쓰기는 밖               ← Kafka가 보장 못 함
```

DB까지 포함하는 exactly-once를 달성하려면 별도 아키텍처가 필요하다:

| 방법 | 원리 |
|------|------|
| **Offset in DB** | DB transaction 안에서 offset과 데이터를 함께 저장. 재시작 시 DB에서 offset을 읽음 |
| **Outbox 패턴** | DB에 이벤트를 저장하고, CDC(Change Data Capture)로 Kafka에 전달 |
| **Idempotent write** | unique key로 중복 INSERT를 방지하여 재처리해도 결과 동일 |

대부분의 경우 이런 아키텍처는 과도한 복잡도를 수반한다. 그래서 실용적으로는 **at-least-once + idempotent handler** 조합이 가장 많이 선택된다.

---

## 정리

| 전략 | 중복 범위 | 처리량 | 복잡도 |
|------|----------|--------|--------|
| Auto-commit | auto-commit 주기 내 처리량 | 최고 | 최저 |
| 메시지별 Manual commit | 최대 1건 | 낮음 | 낮음 |
| Batch Manual commit | 최대 batch 크기 | 중간 | 중간 |
| Transactional | 없음 (Kafka-to-Kafka만) | 최저 | 높음 |

대부분의 비즈니스 애플리케이션에서는 **"메시지별 Manual commit + Idempotent handler"**가 가장 실용적인 best practice다.

- Manual commit으로 중복 범위를 최대 1건으로 줄이고
- Idempotent handler로 그 1건의 중복도 안전하게 처리

Kafka의 auto-commit은 편리하지만, crash 시 중복 처리가 발생한다는 것을 이해하고, 비즈니스 요구사항에 맞는 전략을 선택해야 한다.
