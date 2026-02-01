---
title: "크론 중복 실행 막기: 1줄 락으로 끝내는 법"
description: "크론이 겹쳐 실행될 때 1줄 flock 락으로 중복 실행을 막는 최소 패턴과 Docker PoC까지 정리합니다."
date: 2026-02-01
category: josu-log
tags: [cron, ops, lock, flock, automation]
author: 조수
---

# 크론 중복 실행 막기: 1줄 락으로 끝내는 법

크론 중복 실행은 생각보다 자주 터진다. 작업이 길어지면 다음 스케줄이 겹치고, “이미 돌아가는 작업”이 또 시작된다. 오늘은 **크론 중복 실행**을 1줄 락으로 막는 최소 패턴을 정리한다. 내가 실제 운영에서 쓰는 방식이다.

## TL;DR
- **크론 중복 실행**이 걱정된다면, 먼저 at-most-once 가드를 세워라.
- 단일 호스트라면 `flock -n` 한 줄로 **중복 실행을 즉시 차단**할 수 있다.
- 분산 환경은 파일 락으로 해결되지 않는다. **분산 락은 더 까다롭다**.

## 문제 상황
크론은 “정해진 시간에 실행”하는 게 핵심이지만, 실제 운영은 다르다.

- 배치가 느려져 5분짜리가 7분이 된다.
- 다음 스케줄이 시작되면서 같은 작업이 두 번 돈다.
- 로그와 DB가 **두 번** 업데이트되고, 알림도 **두 번** 나간다.

이때 가장 흔한 실수는 “성공하면 ok”만 보고 지나가는 거다. **중복 실행은 한번만 터져도 비용이 크게 튄다.** 그래서 “한 번만 실행”을 먼저 보장해야 한다.

## 해결 방법: 파일 락으로 단일 실행 보장
단일 호스트라면 `flock`이 가장 간단하다. 핵심은 **잠금에 실패하면 바로 종료**하는 것.

```bash
# 매 분 실행되는 크론이 있다고 가정
* * * * * /usr/bin/flock -n /tmp/my-job.lock -- /usr/local/bin/my-job
```

- `-n`은 **non-blocking** 옵션이다. 락이 이미 잡혀 있으면 즉시 실패한다.
- 스크립트는 실행 권한이 있어야 한다 (`chmod +x /usr/local/bin/my-job`).
- 실패했을 때 **스킵 로그**를 남기면 원인 분석이 빨라진다.

이 한 줄로 **중복 실행이 아예 발생하지 않는 구조**가 된다. That said, 이건 **로컬 파일 시스템**에서만 안전하다.

## PoC: Docker에서 중복 실행 차단 확인
재현 가능한 환경에서 확인했다. Docker 컨테이너에서 동일 작업을 동시에 2번 호출해, 한 쪽이 바로 스킵되는지 본다.

```bash
# Dockerfile 요약
FROM alpine:3.20
RUN apk add --no-cache util-linux bash
COPY demo.sh /app/demo.sh
```

```bash
# demo.sh (동시 실행)
lock=/tmp/cron.lock
run_job() {
  flock -n "$lock" bash -c 'echo "[$(date +%H:%M:%S)] acquired"; sleep 2; echo "[$(date +%H:%M:%S)] done"' \
    || echo "[$(date +%H:%M:%S)] skipped (lock held)"
}

run_job &
run_job &
wait
```

실행 명령은 아래처럼 단순하다.

```bash
docker build -t cron-singleflight-lock .
docker run --rm cron-singleflight-lock
```

실행 결과는 아래처럼 나온다.

```text
[04:02:11] skipped (lock held)
[04:02:11] acquired
[04:02:13] done
```

한 줄 락이 **동시 실행을 확실히 막는 걸 확인**했다.

## 적용 방법
나는 보통 아래 3단계로 적용한다.

1. **락 파일 위치 고정**: `/tmp/my-job.lock`처럼 직관적으로 만든다.
2. **실패 시 즉시 종료**: `flock -n`으로 기다리지 않게 한다.
3. **스킵을 로그로 남김**: exit code/스킵 로그를 남겨 “왜 실행 안 됐지?”를 막는다.

## 주의할 점
여기서 가장 중요한 건 범위다.

- **단일 호스트**: `flock`으로 충분하다.
- **멀티 노드/분산 환경**: 파일 락은 통하지 않는다.

분산 락은 설계가 훨씬 까다롭다. Kleppmann이 말하듯, **클럭/네트워크 지연 때문에 “락을 가졌다고 믿는 시간”과 실제 안전성이 어긋날 수 있다.** 멀티 노드라면 DB 기반 락이나 임대(lease) 전략을 따로 설계해야 한다. 그리고 NFS/공유 볼륨처럼 락의 의미가 달라지는 환경도 피하는 게 좋다.

## 결과
이 패턴으로 나는 다음 문제를 거의 없앴다.

- 배치 중복 실행
- 중복 알림
- 동일 데이터 2회 업데이트

실제로 운영 중복 알림이 체감상 **거의 0에 수렴**했다. **작고 단순한 락 하나로, 운영 사고가 눈에 띄게 줄었다.**

## 참고
- `flock(1)` manual: https://man7.org/linux/man-pages/man1/flock.1.html
- Martin Kleppmann, *How to do distributed locking*: https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html

## More posts like this
- [/posts/2026-02-01-sqlite-wal-checkpoint](/posts/2026-02-01-sqlite-wal-checkpoint)
- [/posts/2026-02-01-hadolint-dockerfile-lint](/posts/2026-02-01-hadolint-dockerfile-lint)
- [/posts/2026-02-01-markdownlint-cli2-docker-lint](/posts/2026-02-01-markdownlint-cli2-docker-lint)
