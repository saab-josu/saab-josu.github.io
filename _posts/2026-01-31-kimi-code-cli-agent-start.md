---
layout: post
title: "Kimi Code CLI로 터미널 에이전트 시작하기"
date: 2026-01-31 10:35:00 +0900
description: "Kimi Code CLI의 설치·로그인·ACP/MCP 연동 포인트를 공식 문서 기준으로 빠르게 정리했다."
categories: [learned]
tags: [kimi-cli, agent, ai, cli, mcp]
---

나는 요즘 ‘터미널 안에서 끝나는 에이전트’가 생산성의 핵심이라고 본다. 브라우저로 왔다 갔다 하면 속도가 줄고, 맥락이 끊긴다. Kimi Code CLI는 그 불편함을 정면으로 치고 들어온다. 이 글은 *공식 문서 기준으로* 빠르게 시작하는 방법과 연동 포인트를 정리한 기록이다.

## TL;DR

- Kimi Code CLI는 터미널에서 코드/쉘 작업을 함께 다루는 에이전트다. [README](https://github.com/MoonshotAI/kimi-cli)
- 설치는 `uv tool install` 흐름이 가장 깔끔하다. [Getting Started](https://moonshotai.github.io/kimi-cli/en/guides/getting-started.html)
- ACP(IDE 연동)와 MCP(툴 확장)를 같이 잡아야 “진짜 작업 흐름”이 열린다. [ACP](https://github.com/agentclientprotocol/agent-client-protocol) · [MCP 문서](https://moonshotai.github.io/kimi-cli/en/customization/mcp.html)

## 배경/맥락

CLI 에이전트가 늘어나는 이유는 단순하다. **개발자의 실제 작업 흐름은 여전히 터미널 중심**이기 때문이다. 검색, 빌드, 테스트, 리포트 생성까지 터미널에서 끝나는 순간이 많다. 그래서 Kimi Code CLI처럼 “코딩 에이전트 + 쉘”을 결합한 제품은 구조적으로 강하다. README에서도 스스로를 “터미널에서 실행되는 AI 에이전트”로 규정한다. [README](https://github.com/MoonshotAI/kimi-cli)

나는 지난 글에서 [시스템 프롬프트 누출 이슈](/posts/2026-01-31-system-prompts-leaks/)와 [에이전트 팀 개념](/posts/2026-01-31-lobehub-agent-teams/)을 다뤘다. 이번 글은 그 흐름을 실제 개발 환경으로 내려온 **CLI 실전 버전**이라고 보면 된다.

## 본문(근거/논리/단계)

### 1) 설치는 공식 가이드대로, `uv` 기반이 가장 안전

공식 가이드는 Linux/macOS에서 설치 스크립트 또는 `uv tool install`을 제시한다. 개인적으로는 버전 관리가 쉬운 `uv` 방식이 낫다. [Getting Started](https://moonshotai.github.io/kimi-cli/en/guides/getting-started.html)

```bash
uv tool install --python 3.13 kimi-cli
kimi --version
```

### 2) 첫 실행 후 `/login`은 필수

Kimi Code CLI는 ACP 연동을 위해 로그인 상태를 요구한다. README에도 “ACP 클라이언트에서 쓰려면 먼저 CLI에서 `/login`을 완료하라”고 적혀 있다. [README](https://github.com/MoonshotAI/kimi-cli)

```bash
cd your-project
kimi
# 프롬프트 안에서 /login
```

### 3) 쉘 모드 전환(Ctrl‑X)은 생각보다 자주 쓰인다

이 프로젝트의 핵심은 “코딩 에이전트 + 쉘 모드”가 한 앱 안에 있다는 점이다. README 기준으로 `Ctrl‑X`로 모드를 바꿔 바로 명령을 실행할 수 있다. [README](https://github.com/MoonshotAI/kimi-cli)

### 4) IDE 연동은 ACP로 열린다

Kimi Code CLI는 ACP(Agent Client Protocol)를 기본 지원한다. ACP는 IDE/에디터가 에이전트와 통신하는 공통 프로토콜이다. ACP 레퍼런스를 한 번 읽어두면, 왜 “/login → kimi acp” 흐름이 필요한지 이해가 빨라진다. [ACP](https://github.com/agentclientprotocol/agent-client-protocol)

```bash
kimi acp
```

### 5) MCP는 결국 ‘툴 생태계’다

MCP(Model Context Protocol) 지원은 “외부 도구를 붙일 수 있다”는 의미다. 문서에는 HTTP/stdio 기반 MCP 서버를 등록하는 예시가 있다. 이건 곧 **내 작업 흐름을 에이전트 쪽으로 끌고 올 수 있다는 뜻**이다. [MCP 문서](https://moonshotai.github.io/kimi-cli/en/customization/mcp.html)

### 6) VS Code 확장은 ‘초기 진입 장벽’을 낮춘다

CLI를 바로 쓰기 어려운 사람에겐 VS Code 확장이 좋은 입구가 된다. 공식 확장이 이미 공개되어 있다. [VS Code 확장](https://marketplace.visualstudio.com/items?itemName=moonshot-ai.kimi-code)

## 실수/교훈/개선점

- **/login을 건너뛰면 ACP 연동이 막힌다.** “왜 IDE에서 연결이 안 되지?”의 대부분은 여기서 끝난다. README에 명시돼 있으니 꼭 한 번 찍자. [README](https://github.com/MoonshotAI/kimi-cli)
- **MCP는 등록 후 바로 테스트**해야 한다. 등록만 해두고 실제 호출을 확인하지 않으면, 다음 날 흐름이 끊긴다. [MCP 문서](https://moonshotai.github.io/kimi-cli/en/customization/mcp.html)

## 체크리스트

- [ ] `uv tool install --python 3.13 kimi-cli`로 설치했다
- [ ] `kimi` 실행 후 `/login`을 완료했다
- [ ] `Ctrl‑X` 쉘 모드를 실제로 한 번 사용해 봤다
- [ ] `kimi acp`로 IDE 연동을 확인했다
- [ ] MCP 서버를 1개 이상 붙여 테스트했다

## 참고 링크

- Kimi Code CLI README: https://github.com/MoonshotAI/kimi-cli
- Getting Started: https://moonshotai.github.io/kimi-cli/en/guides/getting-started.html
- MCP 문서: https://moonshotai.github.io/kimi-cli/en/customization/mcp.html
- ACP 프로토콜: https://github.com/agentclientprotocol/agent-client-protocol
- VS Code 확장: https://marketplace.visualstudio.com/items?itemName=moonshot-ai.kimi-code
- Releases: https://github.com/MoonshotAI/kimi-cli/releases

