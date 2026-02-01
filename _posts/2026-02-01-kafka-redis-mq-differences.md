---
title: "Kafka vs Redis: MQ 방식의 본질 차이"
date: 2026-02-01 23:20:00 +0900
categories: [배운 것]
tags: [kafka, redis, mq, streaming, pubsub]
description: "Kafka와 Redis를 메시지 큐로 쓸 때의 사용 방식, 보장 수준, 운영 포인트를 실무 관점에서 비교 정리합니다."
image:
  path: /assets/img/posts/2026-02-01-kafka-redis-mq-differences-cover.png
  alt: "Kafka 로그 스트림과 Redis 큐의 대비"

---


# Kafka vs Redis: MQ 방식의 본질 차이

Kafka와 Redis를 ‘둘 다 MQ로 쓸 수 있다’고 뭉뚱그리면 사고가 난다. 나는 확실히 말할 수 있다. **Kafka는 로그 기반 스트리밍 플랫폼이고, Redis는 메모리 기반 데이터 구조 서버**다. 비슷해 보이는 건 인터페이스일 뿐, 운영 포인트와 실패 모드는 완전히 다르다.

이 글은 실무자 기준으로 **사용 방식**, **보장 수준**, **운영 난이도**를 직설적으로 비교한다. 결론부터 말하면, **내구성·재처리·재현성이 중요하면 Kafka**, **초저지연·간단한 fan-out이 중요하면 Redis**다.

## 배경: 둘 다 MQ처럼 보이는 이유

Kafka는 토픽에 메시지를 쌓고 컨슈머가 읽는다. Redis도 리스트, Pub/Sub, Streams로 메시지를 전달한다. 표면만 보면 둘 다 “큐”다. 하지만 구조가 다르다.

Kafka는 디스크에 **append-only 로그**로 쌓고, offset으로 읽는다. Redis는 메모리 위 데이터 구조로 큐를 흉내낸다. That said, Redis Streams는 Kafka에 꽤 가까워졌지만, **Kafka의 설계 철학(로그 보존 + 재처리)**과는 여전히 다르다.

## 핵심 차이 1: 데이터 모델과 보존 철학

Kafka는 “메시지”가 아니라 “로그 이벤트”를 저장한다. 즉, **메시지는 소비돼도 사라지지 않는다.** retention 기간 동안 언제든 재처리 가능하다. 반면 Redis의 List/PubSub은 소비되면 사라지거나(또는 즉시 휘발) 재처리가 어렵다. Streams는 비교적 보존이 가능하지만 기본은 메모리다.

### Kafka: 로그 중심

```bash
# 토픽 생성 (retention 7일)
kafka-topics.sh \
  --create --topic orders \
  --partitions 12 --replication-factor 3 \
  --config retention.ms=604800000
```

Kafka에서 중요한 건 **offset 관리**다. 컨슈머가 어디까지 읽었는지가 “상태”이고, 이게 재처리/리플레이의 기반이다.

### Redis: 데이터 구조 중심

Redis는 구조가 많다. MQ 용도로는 **List**, **Pub/Sub**, **Streams**를 쓴다. 각자 철학이 다르다.

```bash
# Redis List로 간단한 큐
LPUSH queue:email "user:123"
BRPOP queue:email 0
```

List는 간단하지만 **내구성/재처리**를 기대하면 안 된다. 빠르고 쉬운 대신, 운영에서 문제를 만들기 딱 좋다.

## 핵심 차이 2: 소비 모델과 재처리

Kafka는 **Consumer Group**으로 수평 확장하고, offset을 기준으로 재처리한다. Redis Pub/Sub은 **구독 시점 이후** 메시지만 받는다. 즉, 구독이 끊기면 그 사이 메시지는 유실된다. Redis Streams는 소비자 그룹을 제공하지만, ack 관리와 레이턴시 튜닝은 직접 해야 한다.

### Kafka 소비 예시

```python
# kafka-python 예시
from kafka import KafkaConsumer

consumer = KafkaConsumer(
    "orders",
    bootstrap_servers=["kafka-1:9092"],
    group_id="billing",
    auto_offset_reset="earliest",
    enable_auto_commit=False,
)

for msg in consumer:
    process(msg.value)
    consumer.commit()
```

Kafka의 핵심은 **idempotency**와 **offset 커밋 전략**이다. 이걸 잘못 잡으면 재처리 때 중복이 터진다.

### Redis Streams 소비 예시

```bash
# 소비자 그룹 생성
XGROUP CREATE stream:orders billing $ MKSTREAM

# 메시지 읽기
XREADGROUP GROUP billing consumer-1 COUNT 10 BLOCK 5000 STREAMS stream:orders >
```

Streams는 “Kafka 흉내”를 낼 수 있지만, **보장 수준**은 애플리케이션이 만들어야 한다. pending list, retry, dead-letter 처리를 직접 구현해야 한다.

## 핵심 차이 3: 내구성 vs 지연시간

Kafka는 디스크 기반이라 **내구성**이 좋고, 재처리와 감사(Audit)가 가능하다. 대신 최저 지연을 바라면 부담이 있다. Redis는 메모리 기반이라 **지연이 매우 낮고 단순**하지만, 장애 시 손실 리스크를 감수해야 한다.

현장에서 내가 본 패턴은 명확했다. **“몇 개 메시지가 날아가도 괜찮다”**면 Redis가 훨씬 쉽다. **“한 건이라도 놓치면 안 된다”**면 Kafka를 쓴다. 이건 기술 문제가 아니라 *비즈니스 리스크* 문제다.

## Kafka MQ 사용 방식 정리

Kafka를 MQ로 쓸 때는 아래를 기본으로 깔고 시작한다.

1. **토픽을 이벤트 로그로 본다** (삭제되지 않는다는 전제)
2. **Consumer Group으로 확장**한다
3. **offset 커밋을 신중하게 설계**한다
4. **재처리 전략**을 먼저 잡는다

```bash
# 컨슈머 그룹 상태 확인
kafka-consumer-groups.sh \
  --bootstrap-server kafka-1:9092 \
  --group billing --describe
```

Kafka의 운영 난이도는 “클러스터”가 아니라 **데이터 계약**에서 온다. 스키마 버전, 이벤트 구조, backward compatibility를 팀 차원에서 다뤄야 한다.

## Redis MQ 사용 방식 정리

Redis는 목적에 맞게 고르면 된다.

- **List**: 단순 큐, 작업 분배
- **Pub/Sub**: 즉시 fan-out, 실시간 알림
- **Streams**: 로그에 가까운 큐, 그룹 소비

```python
# Pub/Sub 예시
import redis

r = redis.Redis(host="redis", port=6379)

pubsub = r.pubsub()
pubsub.subscribe("alerts")

for msg in pubsub.listen():
    handle(msg)
```

Pub/Sub은 “실시간 이벤트”로는 최고다. 하지만 구독이 끊기면 메시지는 사라진다. **이걸 보완하려면 Streams로 가야 한다.**

## 실무에서 나는 이렇게 선택한다

- **주문/결제/정산**: Kafka
- **알림/웹소켓 fan-out**: Redis Pub/Sub
- **비동기 작업 큐(재시도 필요)**: Redis Streams 또는 Kafka
- **짧은 캐시성 이벤트**: Redis List

내가 가장 많이 본 사고는 “Redis로 했는데 메시지가 없어졌다”다. 그건 Redis가 나빠서가 아니라 **우리가 요구사항을 과하게 기대했기 때문**이다.

## 비교표: 한눈에 정리

| 항목 | Kafka | Redis (List / PubSub / Streams) |
|---|---|---|
| 데이터 모델 | 로그(append-only) | 메모리 데이터 구조 |
| 보존 | retention 기반 장기 보존 | 구조별 상이, 기본은 휘발 | 
| 소비 모델 | offset, consumer group | 구독/큐/그룹 다양 |
| 재처리 | 강력 | Streams에서만 제한적 |
| 운영 난이도 | 높음(스키마/오프셋) | 낮음(단, 보장 설계 필요) |
| 지연 | 중간 | 매우 낮음 |

## 다르게 할 것들 (솔직한 회고)

나도 한때 Redis List로 모든 비동기 작업을 처리했다. 초기에는 빨랐고 단순했다. Boy was I wrong. 장애 한 번 나면 **재처리와 추적**이 지옥이었다. 결국 Streams로 옮겼고, 중요한 흐름은 Kafka로 분리했다. 이 과정을 겪고 나니, **MQ 선택은 기술 스펙이 아니라 실패 시 복구 가능성**이 기준이라는 걸 뼈저리게 느꼈다.

## 결론: 정답은 하나가 아니라 “리스크 기준”이다

Kafka는 **내구성과 재현성**을 leverage할 때 최고의 선택이다. Redis는 **속도와 단순성**을 leverage할 때 게임 체인저다. 그걸 바꾸면 사고 난다.

그래서 나는 이렇게 말한다. **“이 메시지가 유실되면 회사가 얼마나 아픈가?”** 이 질문에 답하고 도구를 고르는 게 실무자의 역할이다.

---

## 더 읽을거리

- [지수 백오프에 지터를 넣는 이유: 재시도 폭주 막기](/posts/exponential-backoff-jitter/)
- [매시간 크론 잡 로그, 이렇게 안 터뜨린다](/posts/hourly-cron-log-rotation/)
- [AI 글쓰기, 초안보다 ‘비평 루프’가 더 중요하다](/posts/critique-loop-ai-writing/)
