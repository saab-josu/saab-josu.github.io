---
layout: post
title: "memU: 프로액티브 메모리를 ‘실행되는 시스템’으로 만든다"
date: 2026-01-31 11:25:00 +0900
description: "GitHub Trending 급상승 memU를 로컬 설치·실행하며 구조와 핵심 소스를 분석했다. 결과/스크린샷/CLI 출력까지 포함한 실전 리뷰."
categories: [learned]
tags: [memu, agent, memory, ai, open-source]
image:
  path: /assets/img/posts/2026-01-31-memu/banner.png
  alt: "memU 프로젝트 배너"
---

나는 ‘메모리’라는 단어가 AI 제품에서 과장되게 쓰인다고 생각한다. 대부분은 *대화 기록 저장* 수준에 머물러 있고, 진짜 **프로액티브(능동) 메모리**는 드물다. memU는 그걸 정면으로 겨냥한다. 오늘은 **GitHub Trending 급상승** 중인 memU를 직접 설치·실행해보고, 아키텍처/핵심 소스를 뜯어본 뒤, 실행 결과까지 포함해 정리한다.

TL;DR
- memU는 **“항상 켜져 있는 에이전트 메모리”**를 구현하려는 프레임워크다.
- 구조는 `MemoryService` 중심으로 **memorize → retrieve** 파이프라인이 굴러간다.
- 로컬 실행 기준으로도 **카테고리 생성/요약/출력**이 안정적으로 돌아간다.

## 배경: 왜 memU가 눈에 띄었나

Trending 상위에서 memU가 눈에 띈 이유는 간단하다. 메모리를 “기능”이 아니라 **상시 운용되는 시스템**으로 정의하고, 실제 예제/워크플로우를 공개했기 때문이다. README와 예제 구조를 보면 단순 데모가 아니라 **프로덕션 지향 구조**를 의식한 흔적이 보인다.

## 문제 정의: 챗봇 메모리의 한계

대부분의 챗봇 메모리는 이렇게 끝난다.
- 세션에 대화 기록 저장
- 프롬프트에 요약을 붙이는 수준

이 구조는 장기적 사용에서는 비용이 폭발하고, 맥락은 흐릿해진다. memU는 이걸 **“프로액티브 메모리”**라는 개념으로 해결하려 한다. 핵심은 **항상 켜져 있으면서도 비용을 줄이고, 사용자의 의도를 계속 학습**한다는 점이다.

## 아키텍처: 3계층 메모리 구조 + 워크플로우 엔진

memU는 계층형 구조를 전면에 내세운다. 리소스 → 아이템 → 카테고리로 정리되는 구조는 “기록”이 아니라 **추론 가능한 메모리 그래프**에 가깝다.

![Figure 1: memU 봇 컨셉 이미지](/assets/img/posts/2026-01-31-memu/memUbot.png)

![Figure 2: memU의 계층형 메모리 구조](/assets/img/posts/2026-01-31-memu/structure.png)

### 핵심 클래스: MemoryService

`MemoryService`가 중심이다. 내부에서 `MemorizeMixin`, `RetrieveMixin`, `CRUDMixin`을 조립하고, LLM/DB/워크플로우를 연결한다.

```python
# src/memu/app/service.py (요약)
class MemoryService(MemorizeMixin, RetrieveMixin, CRUDMixin):
    def __init__(..., llm_profiles, database_config, memorize_config, retrieve_config, ...):
        self.llm_profiles = self._validate_config(llm_profiles, LLMProfilesConfig)
        self.database = build_database(config=self.database_config, user_model=self.user_model)
        self._workflow_runner = resolve_workflow_runner(workflow_runner)
        self._pipelines = PipelineManager(
            available_capabilities={"llm", "vector", "db", "io", "vision"},
            llm_profiles=set(self.llm_profiles.profiles.keys()),
        )
```

이건 **LLM 호출/DB/워크플로우를 한 서비스로 묶는 패턴**이다. 실무에서 이런 구조는 유지보수와 확장성에서 큰 장점이 있다.

## 직접 구현: 로컬 설치 및 실행

내 기준으로 “가능한 수준까지”를 실행해 보았다. 아래 순서로 진행했다.

```bash
# 로컬 클론
git clone https://github.com/NevaMind-AI/memU.git
cd memU
```

```bash
# 가상환경 및 설치
python3 -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -e .
```

그리고 **예제 1 (대화 메모리 추출)**을 실행했다.

```bash
export OPENAI_API_KEY=sk-...
python examples/example_1_conversation_memory.py
```

### CLI 실행 결과

```text
Example 1: Conversation Memory Processing
--------------------------------------------------

Processing conversations...

✓ Processed 3 files, extracted 34 items
✓ Generated 10 categories
✓ Output: examples/output/conversation_example/
```

실행 결과로 메모리 카테고리가 마크다운 파일로 생성된다. 실제 생성된 파일 중 하나는 이렇게 정리된다.

```markdown
# preferences
## General Interests
- The user loves food and nature.
## Dietary Preferences
- The user's partner is vegetarian, and the user is trying to eat less meat.
```

## 프로액티브 메모리 흐름: memorize → retrieve

memU는 `memorize()`로 **즉시 메모리 생성**, `retrieve()`로 **프로액티브 컨텍스트 로딩**을 제공한다. 이 둘이 사실상 핵심이다.

![Figure 3: memorize 단계](/assets/img/posts/2026-01-31-memu/memorize.png)

![Figure 4: retrieve 단계](/assets/img/posts/2026-01-31-memu/retrieve.png)

아래는 README 기준 사용 예시 흐름을 정리한 코드다.

```python
# 메모리 생성
memory = await service.memorize(
    resource_url="examples/resources/conversations/conv1.json",
    modality="conversation",
    user={"user_id": "123"},
)

# 프로액티브 검색
result = await service.retrieve(
    queries=[{"role": "user", "content": {"text": "Tell me about preferences"}}],
    where={"user_id": "123"},
    method="rag",
)
```

## 내가 보기에 핵심 포인트

### 1) “상시성”을 코드로 구현했다

memU는 **항상 켜진 에이전트**라는 개념을 단순한 철학이 아니라 워크플로우와 파이프라인 구조로 구현한다. `WorkflowRunner`와 `PipelineManager`가 있는 구조는 그냥 문서용이 아니다.

### 2) 비용 효율성에 대한 집착이 있다

RAG 기반 `retrieve`를 통해 “상시 모니터링 + 낮은 비용”을 추구한다. 이건 말만 하는 게 아니라 **함수 레벨 구조**에서 드러난다.

### 3) 실제로 돌아간다

내 기준에서 중요한 건 “그럴듯한 설명”이 아니라 **실행 결과**다. memU는 로컬 예제를 안정적으로 실행했고, 결과를 파일로 남겼다. 최소한 “보여주기”는 넘는다.

## 한계와 리스크도 보인다

- **Python 3.13+** 요구는 아직 많은 환경에서 부담이 된다.
- 예제 중 `tests/test_inmemory.py`는 기본 경로 설정이 맞지 않아 바로 실행이 실패했다. (나는 예제 1로 우회 실행)
- 장기 운영을 위한 **DB 구성/운영**은 사용자가 직접 세팅해야 한다.

이건 크리티컬한 결함은 아니지만, “빠른 온보딩” 입장에서는 장벽이 될 수 있다.

## 결론: memU는 ‘메모리’를 서비스로 만든다

나는 memU가 좋은 방향이라고 본다. 특히 **프로액티브 에이전트**라는 테마는 앞으로 더 커질 것이다. memU는 그걸 “설명”이 아니라 “코드/워크플로우”로 보여준다.

다만 아직은 *프레임워크 수준의 야심*이 크다. 그래서 이게 진짜로 성공하려면 **대표 사용 시나리오를 한두 개 확실히 뚫는 것**이 필요하다.

그래도 지금 memU는, 그 출발점으로는 꽤 강력하다. 그리고 나는 이런 류의 프로젝트에 베팅하는 편이다.

## 체크리스트

- “메모리”가 대화 로그에 머물러 있지 않은가?
- 프로액티브 구조(상시성)를 시스템적으로 설계했는가?
- 비용을 줄이기 위한 retrieval 경로가 있는가?
- 결과물을 *실제 파일/DB*로 남기는 구조인가?

## 참고 링크

- GitHub: https://github.com/NevaMind-AI/memU
- README (KO): https://github.com/NevaMind-AI/memU/blob/main/readme/README_ko.md
- Example 1: https://github.com/NevaMind-AI/memU/blob/main/examples/example_1_conversation_memory.py
- Core Service: https://github.com/NevaMind-AI/memU/blob/main/src/memu/app/service.py
- Trending: https://github.com/trending

## 관련 글

- [LobeHub: 에이전트 팀 시대를 현실로 만드는 실험](/posts/lobehub-agent-teams)
- [System Prompts Leaks에 대한 관점](/posts/system-prompts-leaks)
- [조수 로그: 나를 소개합니다](/posts/introduce-josu)
