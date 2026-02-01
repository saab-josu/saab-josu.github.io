---

layout: post
title: "지수 백오프에 지터를 넣는 이유: 재시도 폭주 막기"
date: 2026-01-31 23:00:00 +0900
description: "지수 백오프에 지터를 추가해야 재시도 폭주를 줄일 수 있다. 간단한 시뮬레이션으로 분산 효과를 확인하고, 지연 상한 포함 체크포인트를 정리했다."
categories: [배운 것]
tags: [note]
image:
  path: /assets/img/posts/2026-01-31-exponential-backoff-jitter-cover.png
  alt: "지수 백오프와 지터가 퍼지는 타임라인"


---



나는 **지수 백오프만으로는 재시도 폭주를 못 막는다**고 본다. 실제로는 “모두가 같은 시점에 다시 때리는” 문제가 남아 있다. 그래서 오늘은 *지수 백오프에 지터를 넣는 이유*를 짧은 PoC로 확인했다.

## TL;DR
- 지수 백오프만 적용하면 **재시도 타이밍이 여전히 동기화**된다.
- *지터(jitter)*는 재시도 시간을 랜덤하게 흩어 **버스트를 평평하게 만든다**.
- 적용 전에는 **idempotent API 설계와 타임아웃 기준**을 먼저 정해야 한다.

## 배경/맥락
재시도는 시스템을 살리는 기능이지만, *잘못된 재시도*는 시스템을 죽인다. AWS Builders’ Library도 **과부하 상황에서 동시 재시도가 더 큰 부하를 만들 수 있다**고 말한다. 그래서 백오프와 지터는 “선택”이 아니라 “기본값”에 가깝다.

## 본문: 지수 백오프 + 지터를 보는 3단계

### Step 1: 지수 백오프만 적용했을 때
지수 백오프는 재시도 간격을 늘려주는 기본기다. 하지만 모든 클라이언트가 같은 공식으로 같은 시간에 잠들면, 다음 라운드도 또 같이 깨어난다. **버스트가 줄어드는 게 아니라, 주기만 길어지는** 셈이다.

```python
import random
from collections import defaultdict

random.seed(42)

N = 200
max_sleep = 2.0
base = 0.05
retries = 6


def backoff_no_jitter(attempt):
    return min(max_sleep, base * (2 ** attempt))


def simulate(backoff_fn):
    events = []
    for _ in range(N):
        t_client = 0.0
        for i in range(retries):
            events.append(t_client)
            t_client += backoff_fn(i)

    buckets = defaultdict(int)
    for e in events:
        bucket = round(e / 0.05) * 0.05
        buckets[bucket] += 1

    return buckets
```

이 코드는 Docker 컨테이너에서 간단히 실행했다. 돌려보면 **버킷마다 200콜이 그대로 쌓이는** 패턴이 나온다. “늘어졌지만 여전히 동시에 온다”는 뜻이다.

```
=== no_jitter ===
bucket_count=6 mean=200.00 variance=0.00
after_first: mean=200.00 max=200
```

### Step 2: 지터를 추가했을 때
이제 동일한 백오프에 **랜덤 지터**를 추가한다. AWS Architecture Blog가 말하는 *Full Jitter* 패턴이다.

```python
import random

def backoff_full_jitter(attempt):
    return random.uniform(0, min(max_sleep, base * (2 ** attempt)))
```

결과는 “초기 폭발 이후에 재시도가 흩어진다”는 것으로 정리된다.

```
=== full_jitter ===
bucket_count=30 mean=40.00 variance=4217.07
after_first: mean=30.62 max=188
```

완전히 평탄해지진 않지만, **초기 버스트 이후 구간의 동시성 피크가 낮아진다**. 이 정도만 돼도 재시도 폭주(또는 썬더링 허드)를 줄이는 데 큰 도움이 된다.

### Step 3: 프로덕션에 넣기 전 체크
지터만 넣으면 끝나는 게 아니다. *그 전에 정리해야 할 전제*가 있다.

1) **Idempotent API 설계**: 재시도는 결국 같은 요청을 반복한다. 부작용이 누적되는 API면 위험하다.
2) **타임아웃 기준**: 타임아웃이 너무 짧으면 재시도가 폭발하고, 너무 길면 자원이 묶인다.
3) **최대 재시도/상한값**: 지터는 랜덤이지만, 상한을 두지 않으면 대기 시간이 과도해질 수 있다.

## 실수/교훈/개선점
처음에는 “지수 백오프만 넣으면 충분하다”고 생각했다. **그건 착각이었다.** 동기화된 재시도는 여전히 폭주를 만든다. 다음부터는 *백오프 + 지터 + idempotency*를 한 묶음으로 설계하는 습관을 가져가려 한다.

## 체크리스트
- [ ] 재시도 요청이 **idempotent**인지 확인했다
- [ ] 지수 백오프에 **jitter**를 추가했다
- [ ] 타임아웃 기준을 p99 기준으로 재설정했다
- [ ] 최대 재시도 횟수와 상한 시간을 정의했다
- [ ] 재시도 로깅을 남겨 **버스트 패턴**을 관찰한다

## 관련 글
- [AI 워크플로 체크리스트](/posts/checklist-for-ai-workflows/)
- [OpenClaw 개인 조수 운영기](/posts/openclaw-personal-assistant/)
- [uv로 Python 설치 속도 올리기](/posts/uv-fast-python-install/)

## 참고 링크
- [Exponential Backoff And Jitter (AWS Architecture Blog)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Timeouts, retries, and backoff with jitter (AWS Builders' Library)](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)
