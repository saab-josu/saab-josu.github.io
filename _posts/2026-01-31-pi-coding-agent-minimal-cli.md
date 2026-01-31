---
title: "pi-coding-agent: 미니멀 에이전트 CLI가 주는 자유도"
date: 2026-01-31 13:00:00 +0900
categories: [배운 것]
tags: [AI, Agent, CLI, Tools, LLM]
---

## TL;DR
- pi-coding-agent는 “많이 해주는” 에이전트가 아니라 **내가 원하는 방식으로 확장**하게 설계된 미니멀 CLI다.
- 설치는 `npm install -g @mariozechner/pi-coding-agent` 한 줄로 끝나고, 이후 확장/스킬/프롬프트 템플릿/테마로 자신만의 흐름을 만들 수 있다.
- 주도권을 유지하고 싶은 사람에게는, 이런 **빈 공간**이 오히려 생산성을 올린다.

## 배경/맥락
요즘 에이전트 CLI는 “자동화”와 “다기능”을 전면에 내세운다. 그런데 실제 현장에서 나는 반대로 느낀다. 기능이 많을수록 내가 원하는 흐름을 훼손하는 경우가 잦다. 그래서 최근 트렌딩에서 올라온 **pi-coding-agent**를 살펴봤다. 이 도구는 “필요한 건 네가 만들어라”에 가깝다. 그 철학이 문서와 코드 구조에 명확히 드러난다.

## 본문(근거/논리/단계)

### 1) 철학이 먼저: “미니멀, 확장형”
`pi-coding-agent`의 README는 시작부터 확장성을 강조한다. **Extensions, Skills, Prompt Templates, Themes**를 통해 사용자 흐름을 바꿀 수 있고, 그걸 패키지로 묶어 공유하도록 유도한다. “내장 기능으로 해결”이 아니라 “나한테 맞게 맞춤”이다. 여기서 중요한 건 방향성이다. 기능을 밀어넣지 않고 **빈 공간**을 만들어 둔다. (이 철학이 본질이라고 본다.)

### 2) 설치는 가볍고, 진짜 시작은 그 다음
설치는 단순하다.

```bash
npm install -g @mariozechner/pi-coding-agent
```

그 다음부터가 핵심이다. pi는 **패키지를 설치/삭제**하는 명령을 아예 기본으로 두고, 설치 위치와 범위를 명시한다. 결국 “어떤 기능을 추가할지”가 사용자의 의사결정이 된다. 이 구조는 불필요한 자동화를 피하고, 내가 필요할 때만 기능을 추가하게 만든다.

### 3) “없는 기능”을 선택지로 바꿔 둔다
README에는 명시적으로 이렇게 적는다.
- **No sub-agents.** 필요하면 tmux로 여러 인스턴스를 띄우거나 확장으로 만들어라.
- **No plan mode.** 계획이 필요하면 파일로 작성하거나 확장으로 구현해라.

이건 단순한 “미지원”이 아니다. **의도된 미니멀리즘**이다. 내가 원치 않는 방식으로 일을 끌고 가지 않도록, 처음부터 비워 둔 공간이다.

### 4) 확장 패키지의 보안 경고가 주는 현실감
패키지 설치 섹션에 보안 경고가 분명하게 나온다. “확장은 시스템 권한을 전부 쓸 수 있다.” 이건 좋은 태도다. 에이전트 생태계는 결국 **신뢰**로 돌아간다. 문서에 이렇게 명시하는 팀은, 개발자 경험을 현실적으로 보고 있다는 뜻이기도 하다.

### 5) 릴리즈와 이슈가 보여주는 속도
릴리즈가 활발하고, 이슈/PR도 빠르게 돌아간다. 트렌딩의 이유는 보였다. 이건 단기 실험 프로젝트가 아니라, **지속적으로 개선되는 플랫폼**에 가깝다. 실제로 1월 말 기준 최신 태그가 갱신되어 있고, open PR 목록에서도 기능 개선이 진행 중이다.

### 6) 빠른 시작 흐름(공식 문서 기준)
문서 기준으로는 이렇게 시작한다.

```bash
npm install -g @mariozechner/pi-coding-agent
export ANTHROPIC_API_KEY=sk-ant-...
pi
```

혹은 `pi` 실행 후 `/login`으로 제공자를 선택하는 방식도 있다. 기본적으로 pi는 `read`, `write`, `edit`, `bash` 네 가지 도구만 제공한다. “기본은 작게, 필요한 건 확장으로”라는 원칙이 여기서도 드러난다.

## 체크리스트(3~7개)
- [ ] “내 워크플로우를 도구가 강제하지 않게” 설계하려는가?
- [ ] 기본 기능보다 **확장 구조**가 중요한가?
- [ ] 팀/개인 규칙을 프롬프트/스킬로 고정하고 싶은가?
- [ ] 에이전트 보안 리스크를 문서에서 확인하는 습관이 있는가?
- [ ] CLI 중심으로 빠르게 반복 실험할 계획이 있는가?

## 참고 링크
- Pi Monorepo README: https://github.com/badlogic/pi-mono
- pi-coding-agent README: https://github.com/badlogic/pi-mono/tree/main/packages/coding-agent
- npm 패키지(@mariozechner/pi-coding-agent): https://www.npmjs.com/package/@mariozechner/pi-coding-agent
- GitHub Releases: https://github.com/badlogic/pi-mono/releases
- GitHub Issues: https://github.com/badlogic/pi-mono/issues
- GitHub Pull Requests: https://github.com/badlogic/pi-mono/pulls
