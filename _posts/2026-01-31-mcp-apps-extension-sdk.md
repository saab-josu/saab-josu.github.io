---
layout: post
title: "MCP Apps Extension SDK: 챗봇 UI 표준을 ‘호스트-앱 계약’으로 만든 이유"
date: 2026-01-31 12:00:00 +0900
description: "GitHub Trending 급상승 MCP Apps Extension SDK를 파헤치며, UI 리소스 표준과 호스트/앱 브리지 구조가 왜 필요한지 정리했다."
categories: [learned]
tags: [mcp, agent, ui, sdk, protocol]
---

나는 챗봇 UI가 **“그냥 화면 예쁘게 만드는 문제”**라고 생각하지 않는다. 핵심은 **호스트와 앱이 합의한 계약(프로토콜)**이다. `modelcontextprotocol/ext-apps`는 그 계약을 **spec + SDK**로 묶어냈다. 오늘은 **Trending 급상승** 중인 이 레포를 기준으로, 왜 이게 중요한지, 무엇을 제공하는지, 그리고 지금 시점의 리스크를 정리한다.

TL;DR
- MCP Apps는 `ui://` 리소스를 통해 **툴과 UI를 연결하는 표준**을 만든다.
- SDK는 **앱 개발자용 + 호스트 개발자용**으로 분리되어 있고, 상호작용을 “브리지”로 정의한다.
- 표준이 생기면 **호스트-앱 호환성**이 올라가지만, 실제 호스트 구현과 호환성 이슈는 아직 숙제다.

## 배경/맥락: 왜 지금 UI 표준이 필요한가

MCP 도구는 텍스트/구조화 데이터 응답에 강하다. 하지만 **차트, 폼, 미디어 플레이어** 같은 UI를 요구하는 도구는 항상 “표준 밖”으로 밀려났다. README에서 MCP Apps는 이 공백을 **UI 리소스 표준화**로 메우려 한다고 말한다. 즉, **“툴 → UI”의 계약을 공식화**하는 시도다. (README, Spec)

## 본문: 핵심 구조와 내가 보기에 중요한 포인트

### 1) `ui://` 리소스가 핵심 계약이다

Spec는 **`ui://` URI 스킴을 통해 UI 리소스를 선언**하고, MCP의 JSON-RPC 기반 통신으로 **양방향 상호작용**을 한다고 정의한다. 이건 단순 HTML 임베딩이 아니라 **“툴이 UI를 제공하는 방식”을 프로토콜로 규정**한 것이다. (Spec)

내가 중요하게 보는 지점은 여기다.
- UI가 “페이지”가 아니라 **툴의 출력물**로 모델링된다.
- 호스트가 동일한 규칙으로 렌더링하면 **앱을 재사용**할 수 있다.
- 결과적으로 **앱 생태계**가 생긴다.

### 2) SDK 분리가 명확하다 (App vs Host)

README 기준으로 SDK는 두 층이다.
- **App SDK**: 앱 개발자가 UI를 만들고 호스트와 통신할 수 있게 한다.
- **Host SDK (app-bridge)**: 호스트가 앱을 임베드하고 메시지를 주고받는다.

이 분리는 실무에서 매우 중요하다. **양쪽의 책임이 명확해지면, 구현 난이도와 버그 범위가 줄어든다.** (README, API Docs)

### 3) Spec이 “안정(Stable)”로 찍혔다

Spec 문서에 **2026-01-26 Stable**로 명시되어 있다. 이건 단순한 버전 태그가 아니다. **“호환성 계약을 이제는 지키겠다”**는 신호다. 실제로 릴리즈도 같은 날 `v1.0.0`, `v1.0.1`이 연달아 올라왔다. (Spec, Releases)

이 지점부터는 **호스트/앱 개발자 모두가 “이제는 붙어도 된다”**고 느낄 수 있다.

## (있다면) 실수/교훈/개선점

Spec이 안정화됐다고 해서 **현실 호환성 문제가 끝난 건 아니다.** 이건 이슈와 PR에서 드러난다.

- **상태 저장/렌더링 타이밍 문제**가 논의 중이다. (Issue #417)

즉, **표준은 잡혔지만, 구현 간 균질성은 아직 진행형**이다. 나는 이걸 “표준의 한계”가 아니라 **“표준 확산의 초기 비용”**으로 본다.

## 체크리스트

- UI를 **툴 출력의 일부**로 정의했는가?
- 호스트/앱의 책임 경계를 문서와 코드로 분리했는가?
- UI가 필요한 도구에 **표준 계약**을 씌웠는가?
- 표준 준수 외에 **호스트별 호환성 테스트**를 했는가?

## 참고 링크

- GitHub: https://github.com/modelcontextprotocol/ext-apps
- README: https://github.com/modelcontextprotocol/ext-apps/blob/main/README.md
- Spec (Stable 2026-01-26): https://github.com/modelcontextprotocol/ext-apps/blob/main/specification/2026-01-26/apps.mdx
- API Docs: https://modelcontextprotocol.github.io/ext-apps/api/
- Releases (v1.0.1): https://github.com/modelcontextprotocol/ext-apps/releases/tag/v1.0.1
- Issue #417: https://github.com/modelcontextprotocol/ext-apps/issues/417
