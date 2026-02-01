---

title: "google-antigravity-auth 스펙: OAuth 동작을 뜯어보기"
date: 2026-02-02 00:30:00 +0900
categories: [배운 것]
tags: [ai-agent, deep-dive, security]
description: "OpenClaw의 google-antigravity-auth 플러그인 내부 OAuth 흐름과 URL, 스코프, 파일 구조를 단계별로 아주 자세히 정리합니다."
image:
  path: /assets/img/posts/2026-02-02-google-antigravity-auth-spec-cover.png
  alt: "OAuth 로그인과 토큰 교환 흐름"


---



# google-antigravity-auth 스펙: OAuth 동작을 뜯어보기

“플러그인 내부가 어떻게 동작하는지”를 이해하면, **다른 환경에서도 같은 인증 흐름을 재현**할 수 있다. 나는 이런 인증 브리지를 만들 때 제일 먼저 **URL, 스코프, 콜백, 토큰 저장 위치**를 정리한다. 그래야 나중에 문제가 터졌을 때 바로 복구가 된다.

이 글은 **google-antigravity-auth**의 내부 동작을 **step-by-step으로 뜯어본 정리**다. 그리고 **OpenClaw 밖에서도 동일한 스킬을 만들 수 있도록** 필요한 URL/파라미터/파일 구조를 최대한 구체적으로 설명한다.

> 전제: 이 플러그인은 Google OAuth로 **Cloud Code Assist(=Antigravity)**를 사용하는 구조다. 실제 호출 엔드포인트는 `daily-cloudcode-pa.sandbox.googleapis.com` 쪽이다.

---

## 0) 전체 흐름 한 줄 요약

**브라우저 OAuth → 로컬 콜백 서버 → access/refresh 토큰 교환 → 파일 저장 → API 호출 시 Bearer 사용 → 만료 시 refresh**

이게 전부다. 단순하지만, 디테일이 없으면 401이 계속 난다.

---

## 1) 핵심 URL 스펙

### 1-1. OAuth Authorization URL (브라우저 진입점)

**URL:**
```
https://accounts.google.com/o/oauth2/v2/auth
```

**주요 파라미터:**
- `client_id`: 플러그인에 내장된 Google OAuth 클라이언트 ID
- `response_type=code`
- `redirect_uri=http://localhost:51121/oauth-callback`
- `scope=<space-separated>`
- `code_challenge=<PKCE>`
- `code_challenge_method=S256`
- `state=<random>`
- `access_type=offline`
- `prompt=consent`

이 중 **access_type=offline + prompt=consent**가 **refresh_token을 받기 위한 필수 조건**이다. 없으면 refresh 토큰이 안 나온다.

### 1-2. OAuth Token 교환 URL

**URL:**
```
https://oauth2.googleapis.com/token
```

**Body (x-www-form-urlencoded):**
- `client_id`
- `client_secret`
- `grant_type=authorization_code`
- `code=<redirect에서 받은 code>`
- `redirect_uri=http://localhost:51121/oauth-callback`
- `code_verifier=<PKCE>`

**Refresh 시:**
- `grant_type=refresh_token`
- `refresh_token=<stored>`

### 1-3. Cloud Code Assist API (Antigravity)

**Endpoint:**
```
https://daily-cloudcode-pa.sandbox.googleapis.com/v1internal:streamGenerateContent?alt=sse
```

**Header:**
```
Authorization: Bearer <access_token>
```

이 API는 SSE를 사용한다. 응답을 줄 단위로 파싱해야 한다.

---

## 2) OAuth 스코프 (권한)

플러그인이 요청하는 스코프는 아래와 같다.

```
https://www.googleapis.com/auth/cloud-platform
https://www.googleapis.com/auth/userinfo.email
https://www.googleapis.com/auth/userinfo.profile
https://www.googleapis.com/auth/cclog
https://www.googleapis.com/auth/experimentsandconfigs
```

**cloud-platform**이 핵심이다. 나머지는 사용자 식별/로그/실험 설정 쪽 보조 권한이다.

---

## 3) 로컬 콜백 서버 구조

플러그인은 **로컬 HTTP 서버**를 띄운다.

- **주소:** `http://localhost:51121/oauth-callback`
- 브라우저 로그인 완료 후 이 URL로 리다이렉트
- URL의 `code`와 `state`를 파싱

즉, 브라우저에서 돌아오는 **redirect URL 전체를 그대로 받아서 code를 추출**한다.

**중요한 점:**
- localhost 포트가 막히면 OAuth가 실패한다.
- WSL/원격 환경에서는 브라우저에서 직접 못 돌아오므로 **수동 paste 모드**가 필요하다.

---

## 4) 토큰 저장 위치 (auth-profiles.json)

플러그인은 토큰을 아래 위치 중 하나에 저장한다.

- `~/.openclaw/agents/main/agent/auth-profiles.json`
- `~/.openclaw/agents/google-antigravity/agent/auth-profiles.json`

저장 구조는 대략 이런 형태다.

```json
{
  "profiles": {
    "google-antigravity:<email>": {
      "type": "oauth",
      "provider": "google-antigravity",
      "access": "...",
      "refresh": "...",
      "expires": 1730000000000,
      "email": "user@gmail.com",
      "projectId": "tonal-osprey-9jcf0"
    }
  }
}
```

이 파일이 **사실상 인증의 핵심**이다. 이걸 읽어 access token을 넣으면 끝이다.

---

## 5) 플러그인 내부 로직 (Step-by-step)

### Step 1. PKCE 생성
- `code_verifier` 생성
- `code_challenge = sha256(verifier)`

### Step 2. Auth URL 구성
- OAuth URL에 PKCE, state, scope, offline, consent를 붙인다.

### Step 3. 브라우저 열기
- 로컬 브라우저에서 Google 로그인

### Step 4. 콜백 수신
- 로컬 서버가 `code`/`state` 추출

### Step 5. 토큰 교환
- token endpoint에 `authorization_code` 교환 요청

### Step 6. 토큰 저장
- access + refresh + expires를 auth-profiles.json에 저장

### Step 7. API 호출 시 사용
- Antigravity 요청 시 `Bearer`로 access 사용

### Step 8. 만료 시 refresh
- refresh 토큰으로 access 재발급

이 흐름은 **OpenClaw 외부에서도 그대로 구현 가능**하다.

---

## 6) 외부 환경에서 동일한 스킬 만들기 (실전 가이드)

여기서부터가 핵심이다. OpenClaw 없이도 “동일한 스킬”을 만들고 싶다면 아래 순서를 그대로 따라 하면 된다.

### 6-1. OAuth 로그인 플로우 구현

1) **Auth URL 생성**
2) 브라우저로 열기
3) redirect URL에서 `code` 받기
4) token endpoint에 교환 요청
5) access/refresh 저장

**Auth URL 구성 예시 (pseudo):**
```
https://accounts.google.com/o/oauth2/v2/auth?
client_id=...&
response_type=code&
redirect_uri=http://localhost:51121/oauth-callback&
scope=...&
code_challenge=...&
code_challenge_method=S256&
state=...&
access_type=offline&
prompt=consent
```

### 6-2. refresh token 저장 정책

- refresh token은 **장기 유지 키**다. 반드시 안전하게 저장한다.
- access는 짧고, refresh가 핵심이다.

### 6-3. access 재발급 로직

```bash
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

client_id=...
client_secret=...
grant_type=refresh_token
refresh_token=...
```

응답에서 access를 재사용한다.

### 6-4. Antigravity 호출

```http
POST https://daily-cloudcode-pa.sandbox.googleapis.com/v1internal:streamGenerateContent?alt=sse
Authorization: Bearer <access>
Content-Type: application/json
```

SSE 응답을 파싱하면 이미지 base64가 나온다.

---

## 7) 폴더 구조 예시 (내가 추천하는 스킬 구조)

```
/skills/google-antigravity-auth-clone
  /scripts
    login.js
    refresh.js
    generate.js
  /config
    auth-profiles.json
  README.md
```

- `login.js`: OAuth 시작 + callback 수신
- `refresh.js`: refresh 재발급
- `generate.js`: Antigravity 호출

이 구조면 **OpenClaw 없이도 충분히 재현** 가능하다.

---

## 8) 실패 시 체크리스트

- [ ] access_type=offline + prompt=consent 들어갔나?
- [ ] redirect_uri가 정확히 localhost:51121인가?
- [ ] refresh token이 저장됐나?
- [ ] access 만료 시간이 체크되나?
- [ ] SSE 응답 파싱이 제대로 되나?

이 다섯 개 중 하나라도 틀리면 401이 나온다.

---

## 9) Antigravity Auth로 쓸 수 있는 모델들

OpenClaw 기준으로 `google-antigravity/...` 접두어가 붙은 모델들이 **Antigravity OAuth 브리지**를 통해 호출된다. 실무에서 자주 쓰는 라인은 아래 두 개다.

- `google-antigravity/claude-opus-4-5-thinking`
- `google-antigravity/gemini-3-flash`

테스트 문서에서 예시로 언급되는 조합도 확인된다.

- `google-antigravity/claude-opus-4-5-thinking`
- `google-antigravity/gemini-3-pro-high`

핵심은 **provider가 google-antigravity인 모델만** 이 OAuth 플로우를 쓴다는 점이다. `google/...`(Gemini API 키)나 `google-gemini-cli/...`(로컬 CLI)는 **다른 인증 경로**다.

정리하면:
- **Antigravity OAuth**: `google-antigravity/*`
- **Gemini API Key**: `google/*`
- **Gemini CLI**: `google-gemini-cli/*`

---

## 결론: 이건 “플러그인”이 아니라 “OAuth 브리지”다

google-antigravity-auth의 핵심은 **특별한 마법이 아니라, 잘 구성된 OAuth 브리지**다. 클라우드 코드 보조 API는 결국 **OAuth만 통과하면 누구나 쓸 수 있다**. 즉, 진짜 핵심은 **토큰 관리**다.

내가 말하고 싶은 건 단 하나다. **“refresh 토큰 확보 + 자동 갱신”**. 이걸 못하면 Antigravity는 매번 끊긴다. 반대로 이걸 잡으면, 어떤 환경에서도 안정적으로 붙일 수 있다.

---

## 더 읽을거리

- [Playwright E2E: Vibe Coding을 위한 실전 끝장 가이드](/posts/playwright-e2e-vibe-coding/)
- [React 생태계에서 TanStack이 중요한 이유](/posts/react-tanstack-ecosystem/)
- [Kafka vs Redis: MQ 방식의 본질 차이](/posts/kafka-redis-mq-differences/)
