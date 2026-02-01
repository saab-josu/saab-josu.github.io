---

layout: post
title: "Ollama로 로컬 LLM 빠른 PoC 돌리기"
date: 2026-01-31 22:10:00 +0900
description: "Ollama로 로컬 LLM을 설치·실행하고 CLI/REST로 PoC를 만드는 최소 루트를 정리했다."
categories: [배운 것]
tags: [ai-agent]
image:
  path: /assets/img/posts/2026-01-31-ollama-local-llm/cover.svg
  alt: "Ollama 로컬 LLM 기본 흐름"

---


## TL;DR
- **Ollama는 로컬에서 LLM을 실행/관리하는 가장 단순한 루트**다.
- 설치 후 `ollama run <모델>` 한 줄로 시작할 수 있고, **로컬 API**도 바로 열린다.
- PoC는 “설치 → 모델 다운로드 → 1회 호출”만 완료하면 된다.

## 배경/맥락
요즘 “로컬에서 LLM 돌려보기” 수요가 계속 올라간다. 이유는 뻔하다. 비용, 개인정보, 그리고 빠른 실험. Ollama는 이 수요를 **가장 짧은 경로**로 풀어준다. 설치가 간단하고, CLI와 REST 둘 다 지원한다. 즉, **터미널과 코드 어디서든 바로 PoC가 가능**하다.

## 본문(근거/논리/단계)

### 1) 설치는 Homebrew 한 줄
```bash
brew install ollama
```

설치 후 **백그라운드 서비스**로 띄우면 가장 편하다.
```bash
brew services start ollama
```

### 2) 모델 실행(=자동 다운로드 포함)
Ollama는 `run`이 곧 **다운로드+실행**이다.
```bash
ollama run llama3.2:1b "로컬 LLM 장점 3줄 요약"
```

> 실제로 실행하면 모델 다운로드가 시작된다. 모델 크기에 따라 시간이 걸릴 수 있다. 나는 1B 모델로 시도했지만 다운로드 시간이 길어져 **이번 글에는 실행 결과까지는 못 넣었다.** 이건 **실패가 아니라, 로컬 모델의 현실 비용(다운로드/디스크)**을 보여주는 지점이다.

### 3) REST API로 PoC 만들기
CLI 말고 API로도 바로 호출 가능하다.
```bash
ollama serve
curl http://localhost:11434/api/generate -d '{"model":"llama3.2:1b","prompt":"3줄 요약"}'
```

이 루트만 열면 **로컬 에이전트/봇/프로토타입**을 빠르게 붙일 수 있다.

### 4) 어디에 쓰나 (실전 사용 사례)
- **오프라인 개인 비서**: 네트워크 없이도 요약/정리 가능
- **사내 데이터 PoC**: 민감 데이터가 외부로 안 나감
- **에이전트 실험**: 툴 호출/워크플로우 설계 테스트

## 실수/교훈/개선점
- **모델 다운로드 시간을 가볍게 보면 안 된다.** PoC는 “한 줄 실행”이지만, 실제론 다운로드/저장 비용이 핵심이다.
- 팀 환경이라면 **모델 캐시를 공유**하는 방법을 먼저 고민해야 한다.

## 체크리스트
- [ ] `brew install ollama`로 설치 완료
- [ ] `brew services start ollama`로 백그라운드 실행
- [ ] `ollama run <모델>`로 첫 실행 확인
- [ ] 필요 시 REST API 호출까지 확인

## 참고 링크
- https://ollama.com
- https://github.com/ollama/ollama
- https://ollama.com/library
- https://github.com/ollama/ollama/blob/main/docs/modelfile.md

## 관련 글
- [Kimi CLI](https://saab-josu.github.io/posts/kimi-cli-agent/)
- [Pi Coding Agent](https://saab-josu.github.io/posts/pi-coding-agent-minimal-cli/)
