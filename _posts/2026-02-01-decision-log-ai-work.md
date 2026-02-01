---
layout: post
title: "AI 작업 로그: 결정 기록으로 품질을 지키는 법"
date: 2026-02-01 15:00:00 +0900
description: "결정 로그를 고정 포맷으로 남기면 AI 작업의 재현성과 리뷰 속도가 확 올라간다."
categories: [배운 것]
tags: [AI, Decision Log, ADR, Workflow, Documentation]
image:
  path: /assets/img/posts/2026-02-01-decision-log-diagram.svg
  alt: "결정 로그가 작업 흐름에 붙는 간단한 플로우"
---

나는 AI 작업에서 **결정 로그를 남기지 않으면 품질이 무너진다**고 믿는다. 로그가 없으면 “왜 이렇게 했는지”가 사라지고, 같은 삽질을 반복한다. That said, 로그를 길게 쓰는 건 해결책이 아니다. *결정의 이유를 고정 포맷으로 남기는 것*이 핵심이다.

## TL;DR

- 결정 로그는 **긴 회고가 아니라 짧은 포맷**이어야 지속된다.
- `결정/이유/대안/검증` 4칸만 고정해도 재현성이 올라간다.
- AI 작업은 속도가 빠르다. **기록이 없으면 속도만큼 품질이 떨어진다.**

## 배경/맥락

AI 작업은 매일 옵션이 바뀐다. 모델, 파라미터, 프롬프트, 데이터. 나는 이 변화 속에서 “왜 이 결정을 했는지”가 가장 빨리 잊힌다.

문제는 리뷰다. 리뷰는 사실상 *결정의 재판*이다. 로그가 없으면 논쟁이 길어지고, 결국엔 감으로 끝난다. 이건 프로페셔널의 방식이 아니다.

## 본문(근거/논리/단계)

### 1) 로그는 길수록 나쁘다

처음엔 “더 자세히 적어야지”라고 생각했다. 그런데 길어지는 순간, 아무도 읽지 않는다. 기록이 아니라 **소설**이 된다.

결정 로그는 *즉시 회수 가능한 정보*여야 한다. 그래서 나는 4칸으로 고정했다.

- 결정(Decision)
- 이유(Reason)
- 대안(Alternatives)
- 검증(Verification)

이 네 가지가 없으면 나중에 어떤 질문도 답할 수 없다.

### 2) 포맷이 고정되면 회의가 줄어든다

포맷은 논쟁을 줄인다. “왜 이걸 택했지?”가 나오면 이유 칸을 보면 된다. “다른 선택지는?”이 나오면 대안 칸을 보면 된다. 검증 칸은 실험 결과를 붙여두는 자리다.

나는 이걸 *ADR(Architecture Decision Record)* 방식에서 가져왔다. 이름은 중요하지 않다. **결정의 구조를 고정**하는 게 핵심이다.

### 3) PoC: 템플릿 자동 생성으로 속도 높이기

작업 흐름에 붙일 수 있는지 확인하려고, 결정 로그 템플릿을 **한 번에 찍어내는 PoC**를 만들었다. Docker 안에서 실행해 최소 검증만 했다.

```python
from datetime import datetime

context = {
    "title": "ADR-0001: 배치 요약 로그 형식 고정",
    "status": "Accepted",
    "date": datetime.now().strftime('%Y-%m-%d'),
    "context": "AI 작업 로그가 길어지면서 ‘결정의 이유’가 사라졌다.",
    "decision": "결정/이유/대안/검증 결과를 4칸으로 고정한다.",
    "consequences": "재현성이 올라가고, 리뷰 시간이 줄어든다.",
}

md = f"""# {context['title']}

- Status: {context['status']}
- Date: {context['date']}

## Context
{context['context']}

## Decision
{context['decision']}

## Consequences
{context['consequences']}
"""

print(md)
```

출력은 단순하지만 목적은 분명했다. **결정 로그가 ‘자동으로’ 만들어지는 순간**, 기록은 의지가 아니라 습관이 된다.

### 4) 이게 왜 AI 작업에 특히 중요할까

AI 작업은 속도가 빠르다. 빠르면 빠를수록 *사람은 이유를 잊는다*. 나는 이게 품질을 갉아먹는 진짜 원인이라고 본다.

- 같은 실험을 반복한다
- 논쟁이 증거가 아니라 기억으로 흘러간다
- 모델을 바꿨는데 왜 바꿨는지 설명이 안 된다

결정 로그는 결국 **품질을 지키는 브레이크**다. 속도를 버리지 않고, 실수를 줄인다.

## (있다면) 실수/교훈/개선점

처음엔 “로그는 남겨야 하니까”라고 생각했다. 그 결과가 긴 기록이었다. **긴 기록은 결국 기록이 아니다.**

지금은 포맷을 고정하고, 딱 4칸만 채우도록 강제한다. 그게 실제로 남는 유일한 방법이었다.

## 체크리스트

- [ ] 결정/이유/대안/검증 4칸이 고정되어 있는가
- [ ] 로그는 한 화면에 읽히는 길이인가
- [ ] 검증 결과(실험/데이터)가 포함되었는가
- [ ] 회의/리뷰에서 로그를 먼저 확인했는가
- [ ] 반복 실험을 줄였는가

## 참고 링크

- Documenting Architecture Decisions — https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions
- Architectural Decision Records (ADR) — https://adr.github.io/
- Atlassian Team Playbook: Decision Log — https://www.atlassian.com/team-playbook/plays/decision-log
- 내 메모: AI 워크플로우 체크리스트 — /posts/checklist-for-ai-workflows/
- 내 메모: 비평가 루프 설계 — /posts/critique-loop-ai-writing/
