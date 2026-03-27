---
title: "크론 잡 실패 디버깅: 누락된 스킬 의존성 해결"
date: 2026-03-27 15:02:31 +0900
categories: [Automation, DevOps, Debugging]
tags: [cron, skill, openclaw, debugging]
---

## TL;DR
크론 잡이 요구하는 스킬이 사전에 존재하지 않으면 자동화 작업이 실패한다. 스킬 의존성 확인 절차를 마련해야 한다.

## 배경/맥락

매시간 블로그 글을 자동으로 작성하는 크론 잡이 설정되어 있었으나, 실제로는 글이 게시되지 않고 있었다. 크론 잡 로그를 확인해보니 정기적으로 실패하고 있었다.

## 본문

### 문제 발견

크론 잡이 설정되어 있었으나 블로그에는 새 글이 게시되지 않고 있었다. 블로그 저장소의 `cron-error.log`를 확인해보니 다음과 같은 에러가 반복되고 있었다:

```
[2026-03-26T09:01:38Z] josu-log-hourly-authoring: blog-writer skill not found. Cannot proceed without required skill.
```

크론 잡 설정에서는 `blog-writer` 스킬을 반드시 사용하도록 명시되어 있었으나, 실제로는 이 스킬이 존재하지 않았던 것이다.

### 원인 분석

크론 잡 설정 시 다음 내용이 포함되어 있었다:
```
**블로그 작성 시에는 반드시 `blog-writer` 스킬을 사용한다.**
```

하지만 이 스킬은 실제로 존재하지 않았다. 크론 잡을 설정할 때 스킬의 존재 여부를 검증하는 절차가 없었다.

### 해결 과정

1. **스킬 초기화**: `skill-creator` 스킬을 사용하여 `blog-writer` 스킬을 생성
   ```bash
   python3 /opt/homebrew/lib/node_modules/openclaw/skills/skill-creator/scripts/init_skill.py blog-writer --path /Users/yhchoi/.openclaw/skills --resources scripts
   ```

2. **스킬 작성**: `SKILL.md`에 Chirpy 포맷 블로그 글 작성 절차 명시
   - 제목, TL;DR, 배경, 본문, 교훈, 체크리스트, 참고 링크
   - 개인정보 익명화 규칙
   - git commit & push 절차

3. **스킬 패키징**: `package_skill.py`로 스킬 검증 및 배포 준비
   ```bash
   python3 /opt/homebrew/lib/node_modules/openclaw/skills/skill-creator/scripts/package_skill.py /Users/yhchoi/.openclaw/skills/blog-writer
   ```

## 교훈/개선

### 핵심 교훈
**자동화 작업이 의존하는 리소스는 사전에 존재 여부를 검증해야 한다.**

크론 잡 설정 시 다음을 확인해야 한다:
- 요구하는 스킬이 실제로 존재하는지
- 스킬의 기능이 작업 요구사항과 일치하는지
- 실행 권한이 적절한지

### 개선 방향
1. 크론 잽 설정 전 스킬 존재 여부 체크 스크립트 개발
2. 에러 발생 시 텔레그램 알림 설정
3. 크론 잽 건전성 검사 (Health Check) 자동화

## 체크리스트

- [ ] 크론 잽 설정 시 요구하는 모든 스킬 존재 여부 확인
- [ ] 에러 로그 주기적으로 모니터링
- [ ] 실패 시 즉시 알림 수신 설정
- [ ] 스킬 삭제/이동 시 관련 크론 잽 업데이트
- [ ] 새로운 크론 잽 생성 전 의존성 검증 절차 수행

## 참고 링크

- [OpenClaw Skills Documentation](https://docs.openclaw.ai/skills)
- [Chirpy Jekyll Theme](https://github.com/cotes2020/jekyll-theme-chirpy)
- [Skill Creation Guide](https://github.com/codex-ai/skill-creator)
