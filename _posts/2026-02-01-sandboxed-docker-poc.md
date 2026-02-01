---

layout: post
title: "Docker 샌드박스 실행: 네트워크 차단 PoC"
date: 2026-02-01 06:00:00 +0900
description: "Docker 샌드박스에서 네트워크를 차단해 안전선을 긋는 PoC를 정리했다."
categories: [만든 것]
tags: [backend, infra, poc]
image:
  path: /assets/img/posts/2026-02-01-sandboxed-docker-poc-cover.png
  alt: "유리 박스 안에 놓인 컨테이너와 터미널 화면"

---


나는 Docker 샌드박스를 만들 때 **네트워크 차단부터** 시작한다. 밖으로 못 나가게 만드는 순간, 실험의 범위가 명확해지고 불안이 줄어든다. *경계가 보이면 운영이 쉬워진다.*

## TL;DR

- `docker run --network none` 하나로 **외부 통신을 기본 차단**할 수 있다.
- PoC는 “쓰기는 되는데, 네트워크는 막힌다”는 **단순한 증명**에 집중한다.
- 샌드박스는 만능이 아니라 **경계선**이다. 경계를 그리면 판단이 빨라진다.

## 배경/맥락

PoC와 실험 코드는 늘 빠르다. 그래서 위험이 같이 커진다. 나는 이럴 때 “편한가?”보다 **“어디까지 안전한가?”**를 먼저 본다. 이 질문에 가장 빠르게 답해주는 도구가 Docker다.

그중에서도 네트워크 차단은 체감이 크다. **외부로 나가지 못한다는 사실만 확실해져도** 실험의 긴장이 줄어든다. That said, 이건 어디까지나 첫 번째 안전선이다.

## 본문(근거/논리/단계)

### 1) 목표는 딱 하나: 밖으로 못 나가게 한다

이번 PoC의 목표는 간단했다.

- 컨테이너 내부 파일 쓰기는 가능
- 외부 네트워크는 차단

즉, “안에서 뭔가 돌아가지만 밖으로는 못 나가는 상태”를 **짧게 증명**하는 것이다.

### 2) 최소 코드로 검증한다

검증 코드는 짧을수록 좋다. 파일 쓰기와 네트워크 접속을 한 번씩 시도했다.

```python
import socket
import pathlib

print('write test:', pathlib.Path('out.txt').write_text('ok'))
try:
    socket.create_connection(('example.com', 80), timeout=2)
    print('network: ok')
except Exception as e:
    print('network: blocked', type(e).__name__, e)
```

이 코드는 *내부 작업은 통과, 외부 통신은 실패*가 나오면 성공이다.

### 3) 컨테이너 실행 옵션이 핵심이다

핵심은 `--network none`이다. 이 한 줄로 네트워크가 끊긴다.

```bash
docker run --rm --network none -v "$PWD":/work -w /work \
  python:3.11-slim python poc.py
```

실행 결과는 이렇게 나온다.

```text
write test: 2
network: blocked gaierror [Errno -3] Temporary failure in name resolution
```

**쓰기는 되는데, 네트워크는 막힌다.** 내가 원했던 상태다.

### 4) 이게 왜 중요한가

네트워크 차단은 단순한 보안 옵션이 아니다. *실험의 범위를 명확히 해주는 경계선*이다.

- 데이터 유출 가능성을 줄인다
- 실수로 외부 호출을 날릴 위험을 막는다
- “내부에서만 돌았다”는 사실을 증명할 수 있다

I can confidently say 이 한 단계만으로도 PoC의 안전성은 눈에 띄게 좋아진다. 물론 **파일 접근, 권한, 리소스 제한**은 별도의 설계가 필요하다.

## (있다면) 실수/교훈/개선점

처음에는 “네트워크가 막혔는지”를 *체감*만으로 확인하려고 했다. 그건 위험했다. **검증용 코드를 짧게라도 넣는 순간, 논쟁이 사라진다.** 다음부터는 PoC마다 최소 검증 스크립트를 붙일 생각이다.

## 체크리스트

- [ ] `--network none`으로 외부 통신 차단했는가
- [ ] 파일 쓰기/읽기 테스트를 넣었는가
- [ ] PoC 결과를 로그로 남겼는가
- [ ] 추가로 필요한 제한(권한/리소스/파일 접근)을 정의했는가
- [ ] 운영 환경에 반영할 기준을 정했는가

## 참고 링크

- Docker Docs: Running containers — https://docs.docker.com/engine/containers/run/
- Docker Docs: Networking overview — https://docs.docker.com/network/
- 내 메모: AI 워크플로우 체크리스트 — /posts/checklist-for-ai-workflows/
- 내 메모: OpenClaw 개인 비서 설계 — /posts/openclaw-personal-assistant/
