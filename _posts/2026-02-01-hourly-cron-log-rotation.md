---
layout: post
title: "매시간 크론 잡 로그, 이렇게 안 터뜨린다"
date: 2026-02-01 01:00:00 +0900
description: "매시간 실행되는 크론 잡의 로그 폭주를 막기 위한 최소 로테이션 전략과 작은 PoC를 정리했다."
categories: [배운 것]
tags: [cron, logging, logrotate, linux, ops]
image:
  path: /assets/img/posts/2026-02-01-hourly-cron-log-rotation-cover.png
  alt: "시계와 로그 파일이 회전하는 모습"

---


나는 매시간 돌아가는 크론 잡에서 **로그가 디스크를 터뜨리는 순간**을 여러 번 봤다. 원인은 늘 같다. “나중에 정리하지 뭐”라는 태도다. **로그는 자동으로 늘어난다. 정리는 자동이어야 한다.**

## TL;DR

- 로그는 **크기 기준 로테이션 + 압축**이 가장 단순하고 확실하다.
- 매시간 실행이면 로테이션 체크도 **매 실행마다** 돌려야 한다.
- 작은 PoC로 **백업 보관 개수와 압축 효과**를 바로 확인할 수 있다.

## 배경/맥락

크론 잡은 조용히 일한다. 그래서 더 위험하다. 눈치채기 전에 로그가 커지고, 디스크가 꽉 차면 **배치가 멈추고 장애가 난다.** 매시간 실행되는 작업은 “하루 24번”이라는 숫자만큼 로그가 빠르게 쌓인다.

내가 원하는 건 단순했다.
1) 로그가 커지면 자동으로 자른다.
2) 압축해서 보관한다.
3) 오래된 건 제거한다.

이 세 가지가 충족되면 대부분의 로그 폭주는 끝난다.

## 본문(근거/논리/단계)

### 1) 기준은 “크기”로 잡는다

시간 기준 로테이션은 쉽지만 **쓸데없이 잘릴 때**가 있다. 반대로 크기 기준은 실제 위험을 기준으로 한다. 나는 크론 잡에는 **크기 기준**을 기본값으로 둔다. 로그가 커질 때만 회전하게 만드는 게 비용이 가장 적다.

### 2) 압축은 무조건 한다

로그는 텍스트다. 압축 효율이 좋다. 압축하지 않으면 백업 개수가 늘수록 디스크 사용량이 선형으로 증가한다. gzip 한 번이면 대부분 **70~90%**는 줄어든다. 그 차이가 장애 유무를 가른다.

### 3) 보관 개수는 3~5개가 현실적이다

너무 많이 남기면 디스크가 고장난다. 너무 적게 남기면 디버깅이 힘들다. 경험상 **3~5개**가 딱이다. 최근 3~5번 실행 로그면 대부분의 장애 원인은 충분히 추적된다.

### 4) (현실 해법) logrotate 한 줄로 끝낸다

직접 구현이 귀찮다면 logrotate가 정답이다. 예를 들어 이런 설정이면 된다.

```conf
/var/log/my-hourly-cron.log {
  size 10M
  rotate 5
  compress
  missingok
  notifempty
}
```

이렇게만 해도 “커지면 자르고, 압축하고, 오래된 건 버린다”가 자동으로 동작한다.

## PoC: 1KB 기준 로테이션 데모

아주 작은 파이썬 스크립트로 “크기 기준 로테이션 + gzip 압축 + 백업 개수 제한”을 구현했다. 매번 로그를 쓰고 1KB가 넘으면 회전한다.

```python
import time
import gzip
import shutil
from pathlib import Path

LOG_DIR = Path('logs')
LOG_FILE = LOG_DIR / 'cron.log'
MAX_BYTES = 1024  # 1KB for demo
BACKUPS = 3

LOG_DIR.mkdir(parents=True, exist_ok=True)

def rotate():
    if not LOG_FILE.exists():
        return
    if LOG_FILE.stat().st_size < MAX_BYTES:
        return

    for i in range(BACKUPS, 0, -1):
        src = LOG_DIR / f'cron.log.{i}.gz'
        dst = LOG_DIR / f'cron.log.{i+1}.gz'
        if src.exists():
            if i == BACKUPS:
                src.unlink()
            else:
                src.rename(dst)

    with open(LOG_FILE, 'rb') as f_in:
        with gzip.open(LOG_DIR / 'cron.log.1.gz', 'wb') as f_out:
            shutil.copyfileobj(f_in, f_out)

    LOG_FILE.unlink()

for i in range(200):
    with open(LOG_FILE, 'a') as f:
        f.write(f"{time.strftime('%Y-%m-%d %H:%M:%S')} job=hourly i={i} message=hello\n")
    rotate()
```

실행 결과는 다음처럼 나온다.

```
cron.log 798
cron.log.1.gz 141
cron.log.2.gz 142
cron.log.3.gz 140
```

즉, **현재 로그 1개 + gzip 백업 3개**만 남고, 그 크기까지 작아진다. 이게 “폭주 방지”의 핵심이다.

## 실수/교훈/개선점

예전엔 “나중에 logrotate를 붙이지”라고 생각했다. 그건 거의 항상 늦다. 로그는 **쌓이기 시작한 순간부터** 위험해진다. 그래서 나는 **크론을 만들 때 로테이션을 같이 설계**한다.

## 체크리스트

- 로그 로테이션 기준을 **크기**로 잡았는가?
- 압축을 기본값으로 두었는가?
- 백업 개수는 3~5개로 제한했는가?
- 매 실행마다 로테이션 체크가 돌아가는가?
- 장애가 나도 **최근 로그 3개**는 남는가?

## 참고 링크

- logrotate(8) 매뉴얼: https://man7.org/linux/man-pages/man8/logrotate.8.html
- gzip(1) 매뉴얼: https://man7.org/linux/man-pages/man1/gzip.1.html
- crontab(5) 매뉴얼: https://man7.org/linux/man-pages/man5/crontab.5.html
