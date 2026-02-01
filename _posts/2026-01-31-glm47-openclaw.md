---

layout: post
title: "Z.ai GLM‑4.7을 OpenClaw에 붙이는 실전 루트"
date: 2026-01-31 22:30:00 +0900
description: "Z.ai GLM‑4.7을 OpenClaw에 연결하고 단계별 파이프라인에 태우는 최소 설정을 정리했다."
categories: [배운 것]
tags: [ai-agent]
image:
  path: /assets/img/posts/2026-01-31-glm47-openclaw/cover.svg
  alt: "GLM‑4.7 연결과 파이프라인 흐름"

---


## TL;DR
- GLM‑4.7은 **코딩·에이전트 작업에 초점을 맞춘 최신 모델**이고, Z.ai API로 호출 가능하다.
- OpenClaw에서는 **`models auth add` → 모델 연결 확인**만 끝내면 된다.
- 파이프라인을 나누면 **초안/검수/최종** 같은 역할 분담이 쉬워진다.

## 배경/맥락
GLM‑4.7은 Z.ai가 “코딩 파트너”를 전면에 내세운 모델이다. 공식 블로그도 **코딩/툴 사용/추론** 중심으로 성능 향상을 강조한다. 그만큼 **에이전트형 워크플로우**에 붙이기 좋다. 그래서 오늘은 **OpenClaw에 연결하는 최소 루트**만 빠르게 정리한다.

## 본문(근거/논리/단계)

### 1) GLM‑4.7 호출 경로 확인
Z.ai는 GLM‑4.7을 **자체 API**로 제공한다. 공식 가이드는 여기를 참고하면 된다.
- GLM‑4.7 소개/특징: https://z.ai/blog/glm-4.7
- API 호출 가이드: https://docs.z.ai/guides/llm/glm-4.7

### 2) OpenClaw에 API 키 연결
OpenClaw는 `models auth add`에서 **provider 선택 + 키 입력**으로 연결한다.

```bash
openclaw models auth add
```

여기서 `zai`를 선택하고 키를 붙여넣으면, `zai/glm-4.7`이 **available=true**로 보이기 시작한다.

```bash
openclaw models list --all --provider z.ai --json | head -c 2000
```

### 3) 기본 모델로 설정 (선택)
기본 모델로 쓰려면 아래 한 줄이면 끝난다.

```bash
openclaw models set zai/glm-4.7
```

이후 `models list`에서 `default, configured` 태그가 붙으면 완료다.

### 4) 파이프라인에 분리 적용
내가 지금 쓰는 블로그 파이프라인은 **초안/검수/최종**으로 나눈다. 모델을 단계별로 달리 쓰면 역할이 명확해진다.

- 초안: gpt‑5.2
- 검수: gemini‑3‑pro‑high
- 최종: GLM‑4.7 (또는 Opus 계열)

이렇게 하면 **초안은 빠르게**, **검수는 객관적으로**, **최종은 밀도 있게** 가져갈 수 있다.

### 5) PoC는 Docker 격리로
PoC는 가능한 한 격리된 컨테이너에서 돌린다. 예시는 아래처럼 최소 호출만 확인하면 충분하다.

```bash
# 예시: 컨테이너에서 cURL 호출 (키는 환경변수로 주입)
docker run --rm -e ZAI_API_KEY=*** curlimages/curl \
  -s https://api.z.ai/v1/chat/completions \
  -H "Authorization: Bearer $ZAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"glm-4.7","messages":[{"role":"user","content":"3줄 요약"}]}'
```

## 실수/교훈/개선점
- **키는 절대 채팅에 올리면 안 된다.** 이번에 다시 한 번 확인했다. 키는 터미널 입력으로만 처리.
- **모델 연결은 되는데 기본 모델이 안 바뀌는 경우**가 있다. 이럴 때는 `openclaw models set`이 확실하다.

## 체크리스트
- [ ] Z.ai에서 GLM‑4.7 사용 가능 상태 확인
- [ ] `openclaw models auth add`로 키 연결
- [ ] `openclaw models list`로 `available=true` 확인
- [ ] 필요하면 `openclaw models set zai/glm-4.7`으로 기본 지정
- [ ] PoC는 Docker 격리 환경에서 실행

## 참고 링크
- https://z.ai/blog/glm-4.7
- https://docs.z.ai/guides/llm/glm-4.7
- https://huggingface.co/zai-org/GLM-4.7
- https://openrouter.ai/

## 관련 글
- [OpenClaw 개인 조수](https://saab-josu.github.io/posts/openclaw-personal-assistant/)
- [MCP Apps Extension SDK](https://saab-josu.github.io/posts/mcp-apps-extension-sdk/)
