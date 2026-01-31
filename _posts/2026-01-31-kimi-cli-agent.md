---
layout: post
title: "Kimi Code CLI: 터미널 에이전트의 기준선 정리"
date: 2026-01-31 16:00:00 +0900
description: "Kimi Code CLI의 설치·로그인·쉘 모드·ACP/MCP·VS Code 연동 포인트를 한 번에 정리했다. 중복 없이 이 글 하나만 유지한다."
categories: [learned]
tags: [kimi-cli, agent, ai, cli, mcp, acp]
---

나는 요즘 “터미널 안에서 끝나는 에이전트”가 생산성의 핵심이라고 본다. Kimi Code CLI는 그 포지션을 정확히 찌른다. 이 글은 **중복 없이 한 번만** 정리한 최종본이다.

## TL;DR
- Kimi Code CLI는 “코딩 도구”가 아니라 **터미널 안에서 실행되는 에이전트 쉘**에 가깝다.
- `Ctrl‑X` 쉘 모드, `kimi acp`(IDE 연동), `kimi mcp`(툴 확장) 이 세 축이 핵심이다.
- 설치보다 중요한 건 **/login → ACP 연결 → MCP 확장** 순서다.

## 배경/맥락
CLI 에이전트가 늘어나는 이유는 단순하다. 개발자의 실제 워크플로우는 여전히 터미널 중심이다. 검색, 빌드, 테스트, 리포트 생성까지 터미널에서 끝나는 순간이 많다. Kimi Code CLI는 “터미널 자체를 에이전트의 실행 환경으로 재정의”하려는 의도가 보인다.

## 설치 & 첫 실행
공식 가이드는 스크립트/`uv` 설치를 제공한다. 개인적으로는 버전 관리가 쉬운 `uv`가 더 안전하다.

```bash
uv tool install --python 3.13 kimi-cli
kimi --version
```

첫 실행 후 **/login**은 필수다. ACP 연동은 로그인 상태를 전제로 한다.

```bash
cd your-project
kimi
# 프롬프트 안에서 /login
```

## 핵심 기능 1: 쉘 모드 전환 (Ctrl‑X)
Kimi는 `Ctrl‑X`로 에이전트 모드 ↔ 쉘 모드를 오간다. 이게 왜 중요하냐면, **대화형과 명령형을 같은 맥락에서 처리**할 수 있기 때문이다. 에이전트가 모든 것을 대화로 감싸면 속도가 느려지고, 반대로 쉘만 쓰면 컨텍스트가 끊긴다. 이 전환 키 하나가 체감 속도를 바꾼다.

## 핵심 기능 2: ACP(IDE 연동)
Kimi CLI는 **Agent Client Protocol(ACP)** 를 지원한다. 핵심은 “CLI는 엔진, IDE는 클라이언트” 구조다.

```bash
kimi acp
```

이 구조가 좋은 이유는 간단하다. 에이전트를 IDE에 종속시키지 않고, **갈아 끼우기 쉬운 엔진**으로 만든다. 팀/조직 관점에서 유지보수가 훨씬 쉽다.

## 핵심 기능 3: MCP(툴 확장)
MCP는 에이전트가 외부 도구에 접근하는 표준 통로다. Kimi는 `kimi mcp` 명령군으로 MCP 서버를 추가할 수 있다. 이건 곧 **사내 API, 문서 시스템, 업무 툴을 연결할 수 있다는 뜻**이다.

현업에서 살아남는 에이전트 CLI의 기준은 결국 “확장 경로”다. MCP가 있는 CLI는 기본 기능보다 훨씬 오래 살아남는다.

## VS Code 확장: 온보딩을 위한 현실적인 접점
VS Code 확장은 기본 사용자층 확보용으로 보인다. CLI를 먼저 쓰고, 필요하면 확장으로 넘어가는 흐름이다. 팀 도입 관점에서는 “공식 확장 존재” 자체가 신뢰도를 만든다.

## 내가 보는 결론
Kimi Code CLI는 “모델 성능”보다 **상호작용 설계**에서 경쟁력이 있다. 대화/명령 전환, ACP 기반 IDE 연동, MCP 확장 경로. 이 세 가지가 터미널 에이전트의 기준선을 올린다.

## 체크리스트
- [ ] `/login`을 완료했고 세션이 유지되는가?
- [ ] `Ctrl‑X` 쉘 모드 전환이 실제로 워크플로우에 도움이 되는가?
- [ ] `kimi acp`로 IDE 연동을 붙였는가?
- [ ] MCP 서버 1개라도 연결해 실제 호출을 확인했는가?

## 참고 링크
- Kimi Code CLI README: https://github.com/MoonshotAI/kimi-cli
- Getting Started: https://moonshotai.github.io/kimi-cli/en/guides/getting-started.html
- MCP 문서: https://moonshotai.github.io/kimi-cli/en/customization/mcp.html
- ACP 프로토콜: https://github.com/agentclientprotocol/agent-client-protocol
- VS Code 확장: https://marketplace.visualstudio.com/items?itemName=moonshot-ai.kimi-code
