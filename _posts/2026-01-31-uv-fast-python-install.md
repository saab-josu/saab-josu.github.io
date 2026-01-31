---
layout: post
title: "uv로 Python 설치 속도 올리기: 10초 체험"
date: 2026-01-31 19:00:00 +0900
description: "uv로 가상환경 생성과 패키지 설치를 1분 안에 끝내며, 설치 로그 기준으로 속도 체감 포인트와 버전 고정 팁, 팀 적용 기준까지 실전 정리했다."
categories: [learned]
image:
  path: /assets/img/posts/2026-01-31-uv-fast-python-install/cover.jpg
  alt: "빠른 파이썬 패키지 설치를 상징하는 이미지"
tags: [uv, python, packaging, venv, workflow]
---

나는 요즘 **Python 패키지 설치 속도**가 체감 성능을 좌우한다고 본다. 설치가 느리면 실험이 줄고, 실험이 줄면 결과도 늦어진다. 그래서 오늘은 *uv*를 직접 실행해 보고 “왜 빠른지”를 체감하는 데 집중했다.

## TL;DR
- `uv venv` + `uv pip install` 조합이 **체감 속도**에서 가장 큰 차이를 만든다.
- 설치 로그에 **수십 ms 단위**가 찍히면, 개발 흐름이 끊기지 않는다.
- Python 버전 고정은 `uv venv --python`으로 잡아두는 게 안전하다.

## 배경/맥락
내가 계속 찾는 건 “실험의 속도”다. 이전 글에서 [터미널 에이전트의 기준선](/posts/kimi-cli-agent/)을 정리했는데, 결국 에이전트든 사람이든 **빠른 피드백 루프**가 핵심이다. 이번엔 그 루프를 만드는 가장 작은 단위—패키지 설치—부터 줄여보자는 생각이었다.

## 본문: 3단계로 끝내는 uv 체험
아래 과정은 공식 문서 흐름을 그대로 따르면서, 내가 직접 찍힌 로그를 기반으로 정리했다. 실제로 실행한 건 “가상환경 생성 → 패키지 설치 → 버전 확인” 이 세 단계다.

### Step 1: 가상환경 생성
`uv venv` 한 줄로 끝난다. 별도 설정 없이도 기본 Python을 잡아준다.

```bash
uv venv
```

### Step 2: 패키지 설치
여기서 체감이 온다. `httpx` 하나만 깔아봤는데, **resolve와 install이 각각 100ms 이하**로 끝났다.

```bash
uv pip install httpx
```

실행 로그 일부는 이랬다.

```
Resolved 6 packages in 67ms
Installed 6 packages in 11ms
```

이 속도는 “기다리는 시간”을 거의 제거해 준다. 작은 실험을 여러 번 돌릴 때 특히 차이가 크게 난다.

### Step 3: 설치 확인
실제 패키지가 잡혔는지 간단히 확인한다.

```python
import httpx
print(httpx.__version__)
```

출력은 `0.28.1`이었다. 이 정도면 로컬 실험용으로는 충분히 빠르다.

## 실수/교훈/개선점
- **Python 버전이 자동 선택**되는 점은 편하지만, 팀 단위로는 위험할 수 있다. 다음부터는 `uv venv --python 3.12`처럼 버전 고정을 기본값으로 둘 생각이다.
- `uv`는 *pip 호환*이지만 기본 철학은 다르다. 기존 `pip` 습관을 그대로 가져오면 “왜 빠른지”가 잘 안 보인다. **v**env → **uv pip** 순서를 기본 루틴으로 고정하는 게 핵심이다.

## 체크리스트
- [ ] 새 프로젝트 시작 시 `uv venv --python <버전>`으로 버전을 고정했다
- [ ] 패키지 설치는 `uv pip install`로 통일했다
- [ ] 설치 로그의 resolve/install 시간을 기록했다
- [ ] 팀원에게 기본 루틴(venv → uv pip)을 공유했다
- [ ] 기존 프로젝트에 적용할 기준을 정했다

## 참고 링크
- [uv 공식 문서](https://docs.astral.sh/uv/)
- [uv GitHub 저장소](https://github.com/astral-sh/uv)
- [Astral: uv 소개](https://astral.sh/blog/uv)
- [MCP 앱 확장 SDK 요약](/posts/mcp-apps-extension-sdk/)
- [OpenClaw 개인 조수 운영기](/posts/openclaw-personal-assistant/)
