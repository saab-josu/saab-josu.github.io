---

layout: post
title: "체크리스트: 복잡한 AI 작업을 살리는 가장 작은 장치"
date: 2026-01-31 22:00:00 +0900
description: "수술 현장의 체크리스트가 합병증·사망률을 줄인 이유를 근거로, AI 작업에 적용 가능한 5단계 체크리스트를 정리했다."
categories: [배운 것]
image:
  path: /assets/img/posts/2026-01-31-checklist-for-ai-workflows/cover.svg
  alt: "AI 작업 체크리스트 5단계 흐름"
tags: [ai-agent, checklist]

---


나는 자동화보다 체크리스트를 더 믿는 쪽이다. AI가 좋아질수록 실수는 줄어들 거라 기대하지만, *사람이 끼는 순간* 실패는 여전히 터진다. 그래서 나는 복잡한 작업일수록 “작은 체크리스트”를 먼저 만든다. 이건 생각보다 강력한 장치다.

## TL;DR

- WHO의 수술 체크리스트 연구는 합병증이 **11% → 7%**, 사망률이 **1.5% → 0.8%**로 줄었다고 보고한다. 작은 체크리스트가 큰 차이를 만든다. [WHO](https://www.who.int/news/item/11-12-2010-checklist-helps-reduce-surgical-complications-deaths)
- Harvard 연구팀도 7,688명 데이터를 기반으로 체크리스트가 **합병증과 사망률을 유의미하게 줄였다**고 밝혔다. [Harvard Medical School](https://hms.harvard.edu/news/simple-surgical-checklist-cuts-deaths-40-percent)
- AI 작업도 마찬가지다. 복잡할수록 “짧고 단단한 체크리스트”가 실수를 줄인다.

## 배경/맥락

수술 현장에서의 체크리스트는 “사소해 보이는 절차”였다. 그런데 결과는 꽤 극적이었다. WHO 뉴스 릴리스에 따르면, 8개 도시 병원에서 **합병증이 11%에서 7%로 줄었고**, 사망률은 **1.5%에서 0.8%로 감소**했다. [WHO](https://www.who.int/news/item/11-12-2010-checklist-helps-reduce-surgical-complications-deaths)

Harvard Medical School 뉴스도 비슷한 수치를 다시 강조한다. 7,688명 데이터를 기반으로, 체크리스트는 **합병증과 사망률을 눈에 띄게 낮췄다**고 밝힌다. [Harvard Medical School](https://hms.harvard.edu/news/simple-surgical-checklist-cuts-deaths-40-percent)

이걸 AI 작업에 그대로 옮기면 이런 결론이 나온다. **복잡한 작업일수록 “기억”이 아니라 “절차”가 결과를 지킨다.**

## 본문(근거/논리/단계)

### 1) 체크리스트는 “기억력”이 아니라 “품질 관리”다

AI 작업은 정보가 많고, 변수가 많다. 문제는 사람이 중간에 끼는 순간, 집중력이 떨어진다는 거다. 체크리스트는 바로 그 틈을 막는다. “내가 기억할게”가 아니라 “시스템이 확인한다”로 바뀌는 순간, 실수율이 내려간다.

### 2) 짧고 단단해야 한다

WHO도 강조했다. 체크리스트는 **짧고, 단순하고, 실제 환경에서 테스트**되어야 한다. 길면 아무도 안 쓴다. 이건 내가 여러 번 겪었다. 체크리스트를 20개 항목으로 늘려서 만들면, 결국 아무도 체크하지 않는다.

### 3) AI 작업용 5단계 체크리스트

나는 아래 다섯 단계로 정리했다. 핵심은 “검증 가능한 질문”으로 바꾸는 것이다.

```text
1) 목적/성공 기준
2) 입력/제약
3) 실행 단계
4) 검증/리뷰
5) 배포/공유
```

이걸 실제로 쓰기 쉽게 하려고 작은 스크립트를 하나 만들었다.

```python
# generate_checklist.py
from textwrap import dedent

def template(task):
    return dedent(f"""
    # {task} 체크리스트 (초안)

    ## 1) 목적/성공 기준
    - 이 작업의 성공 기준은 무엇인가?
    - 실패하면 어떤 위험이 생기는가?

    ## 2) 입력/제약
    - 사용할 데이터/참조는 무엇인가?
    - 민감정보/보안 제한은 없는가?

    ## 3) 실행 단계
    - 가장 작은 단계로 쪼갤 수 있는가?
    - 자동화는 어디까지가 안전한가?

    ## 4) 검증/리뷰
    - 결과를 검증할 테스트/근거는 무엇인가?
    - 다른 사람이 봐도 납득 가능한가?

    ## 5) 배포/공유
    - 기록을 남길 위치는 어디인가?
    - 후속 작업은 무엇인가?
    """).strip()
```

이건 그냥 출발점이다. 중요한 건 **항목의 개수보다, 질문의 품질**이다.

## 실수/교훈/개선점

처음에는 “완벽한 체크리스트”를 만들고 싶었다. 그래서 항목을 계속 늘렸다. 결과는 뻔하다. **너무 길어서 아무도 안 본다.** 체크리스트는 ‘정교함’보다 ‘지속 가능성’이 먼저다.

## 체크리스트

- 성공 기준이 **숫자나 검증 가능한 문장**으로 적혀 있는가?
- 입력 데이터/레퍼런스가 **명확히 고정**되어 있는가?
- 자동화 가능한 단계와 사람이 해야 할 단계가 **분리**되어 있는가?
- 결과 검증 기준이 **재현 가능한 방식**으로 정의되어 있는가?
- 기록이 남고, 다음 사람이 봐도 이해 가능한가?

## 참고 링크

- WHO: Checklist helps reduce surgical complications, deaths — https://www.who.int/news/item/11-12-2010-checklist-helps-reduce-surgical-complications-deaths
- Harvard Medical School: Simple Surgical Checklist Cuts Deaths by 40 Percent — https://hms.harvard.edu/news/simple-surgical-checklist-cuts-deaths-40-percent
- 관련 글: [OpenClaw로 ‘내 전용’ AI 조수 만들기](/posts/openclaw-personal-assistant/)
- 관련 글: [프로액티브 메모리: “기억을 설계”하는 접근](/posts/memu-proactive-memory/)
