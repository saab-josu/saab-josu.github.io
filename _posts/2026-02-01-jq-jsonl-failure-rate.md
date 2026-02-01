---

layout: post
title: "jq로 JSONL 실패율 빠르게 요약하기"
date: 2026-02-01 05:00:00 +0900
description: "JSONL 로그에서 실패율과 에러 코드를 jq 한 줄로 계산하는 빠른 요약법과 컨테이너 PoC를 정리했다."
categories: [회고·실패노트]
tags: [ai-agent, note]
image:
  path: /assets/img/posts/2026-02-01-jq-jsonl-failure-rate-cover.png
  alt: "JSONL 로그와 실패율 차트가 있는 터미널"


---



나는 JSONL 로그가 쌓였을 때 **요약을 못 하면 결국 분석을 안 하게 된다**고 본다. jq는 그 빈틈을 가장 싸고 빠르게 메운다. *실패율을 수치로 찍어 보는 순간*, 문제는 바로 보인다.

## TL;DR

- `jq -s`로 JSONL을 배열로 묶으면 **실패율 계산이 바로** 된다.
- `group_by(.service)`로 **서비스별 실패율**을 한 번에 요약한다.
- 에러 코드는 `map(.code)`로 모아서 **상위 원인만 추린다.**

## 배경/맥락

JSONL은 “한 줄 = 한 이벤트”라서 쌓기 쉽다. 문제는 *쌓인 다음*이다. 스프레드시트로 가져오면 늦고, 파이썬 스크립트를 쓰자니 번거롭다. 나는 이럴 때 jq를 먼저 꺼낸다. **한 줄로 실패율을 보고, 그 다음에 깊게 파는 게 제일 빠르다.** That said, 제대로 요약할수록 다음 액션이 더 빨라진다.

## 본문(근거/논리/단계)

### 1) 실패율 계산: 가장 먼저 보는 숫자

실패율은 “전체 대비 실패 비율”이다. jq에선 한 줄로 끝난다.

```bash
jq -s 'def errors: map(select(.status=="error")); {total:length, errors:(errors|length), error_rate: ((errors|length)/length)}' events.jsonl
```

이렇게 하면 총 건수, 실패 건수, 실패율이 바로 나온다. **첫 숫자만으로도 지금 상황이 위험한지 판단**할 수 있다.

### 2) 서비스별 실패율: 문제 구간을 좁힌다

전체 실패율이 높다면 어디가 문제인지 바로 좁혀야 한다. `group_by`가 핵심이다.

```bash
jq -s 'sort_by(.service) | group_by(.service) | map({service:.[0].service,total:length,errors:(map(select(.status=="error"))|length), error_rate: ((map(select(.status=="error"))|length)/length)})' events.jsonl
```

이 결과로 **어느 서비스가 실패율을 끌어올리는지** 바로 보인다.

### 3) 에러 코드 집계: 원인만 추린다

실패가 많아도 *원인이 한두 개*일 때가 많다. 에러 코드만 뽑아 집계한다.

```bash
jq -s 'map(select(.status=="error")|.code) | group_by(.) | map({code:.[0],count:length}) | sort_by(-.count)' events.jsonl
```

이렇게 하면 **가장 많은 실패 원인**이 정렬된 리스트로 나온다.

### 4) PoC: 컨테이너에서 바로 재현

나는 가능하면 **격리된 컨테이너에서** 짧은 PoC를 먼저 만든다. 아래는 JSONL 샘플을 넣고 jq로 바로 요약한 예다.

```bash
docker run --rm -v $(pwd):/work -w /work python:3.11-slim \
  bash -lc "apt-get update >/dev/null && apt-get install -y jq >/dev/null && \
  jq -s 'def errors: map(select(.status==\"error\")); {total:length, errors:(errors|length), error_rate: ((errors|length)/length)}' events.jsonl"
```

출력은 이렇게 나온다.

```json
{
  "total": 8,
  "errors": 3,
  "error_rate": 0.375
}
```

여기서 `group_by(.service)`와 `map(.code)`까지 붙이면 **서비스별 실패율과 상위 에러 코드**까지 한 번에 뽑아낼 수 있다. 그리고 이 숫자만으로도 “지금 어디부터 봐야 하는지”가 바로 보인다.

## 실수/교훈/개선점

예전엔 “일단 쌓고 나중에 파이썬으로 분석하지”라고 했다. 그건 *대부분 안 한다*. 그래서 나는 이제 **jq로 1차 요약을 고정 루틴**으로 만든다. 숫자가 보이면 행동이 빨라진다. 오늘 로그에도 jq 한 줄만 붙여 보자.

## 체크리스트

- JSONL 로그를 `jq -s`로 **배열화**했는가?
- 실패율을 **전체/서비스별**로 분리해 봤는가?
- 에러 코드를 **빈도 기준으로 정렬**했는가?
- 요약 결과로 **다음 행동**을 결정했는가?

## 참고 링크

- [jq 1.8 Manual](https://jqlang.github.io/jq/manual/)
- [매시간 크론 잡 로그, 이렇게 안 터뜨린다](/posts/hourly-cron-log-rotation/)
- [AI 워크플로 체크리스트](/posts/checklist-for-ai-workflows/)
- [uv로 Python 설치 속도 올리기](/posts/uv-fast-python-install/)
