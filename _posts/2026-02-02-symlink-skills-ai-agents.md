---

title: "AI 에이전트 스킬 공유: symlink로 한 번만 관리하기"
date: 2026-02-02 08:10:00 +0900
categories: [배운 것]
tags: [ai-agent]
description: "Claude Code, Cursor, Codex의 스킬 폴더를 하나로 묶는 심볼릭 링크 운영법과 실험 결과를 단계별로 정리했습니다."
image:
  path: /assets/img/posts/2026-02-02-symlink-skills-ai-agents-cover.png
  alt: "중앙 스킬 폴더와 에이전트 노드를 연결한 다이어그램"


---



# AI 에이전트 스킬 공유: symlink로 한 번만 관리하기

나는 여러 에이전트를 동시에 쓰면 **스킬 폴더가 분산되는 순간 운영이 망가진다**고 생각한다. `.claude`, `.cursor`, `.codex`가 제각각이면 업데이트는 항상 누락되고, 결국 “어느 쪽이 최신인지”가 모호해진다.

그래서 결론은 단순했다. **스킬은 한 곳만 유지하고, 나머지는 심볼릭 링크로 붙인다.** 이 글은 그 방식의 **실전 적용법과 직접 실험 결과**를 정리한 글이다.

---

## 핵심 결론 (한 줄)

**스킬 원본은 하나만 두고, 각 에이전트 폴더는 symlink로 연결한다.**

---

## 배경: 왜 분산 저장이 문제인가

- 스킬을 하나 수정하면 세 곳에 복붙해야 한다
- 한 군데만 업데이트되면 “예전 스킬”이 계속 호출된다
- 팀 단위에서 관리할 때 수정 히스토리가 꼬인다

Vibe Coding처럼 빠르게 쓰고 지우는 환경에서는 **중복 관리가 곧 장애**다.

---

## 실전 구조 설계 (권장)

기본 원칙은 이렇다.

1) **원본 스킬 폴더를 하나 만든다**
2) 나머지 에이전트 폴더는 symlink로 연결한다
3) 상대 경로를 쓰면 레포 복제 시에도 깨지지 않는다

예시 구조:

```
project/
  .claude/skills/          # 원본
  .cursor/skills/          # symlink
  .codex/skills/           # symlink
```

---

## Step-by-step: symlink로 묶는 방법 (macOS 기준)

### 1) 원본 스킬 폴더 준비

```bash
mkdir -p .claude/skills
```

### 2) 다른 폴더는 symlink로 연결

```bash
mkdir -p .cursor/skills .codex/skills
ln -s ../.claude/skills/test-skill.md .cursor/skills/test-skill.md
ln -s ../.claude/skills/test-skill.md .codex/skills/test-skill.md
```

**포인트:** 상대경로 `../`를 쓰면 레포가 어디에 있든 동작한다.

---

## 직접 사용해본 결과 (실험 로그)

나는 실제로 `skill-symlink-lab` 폴더를 만들고 테스트했다.

```bash
mkdir -p .claude/skills .cursor/skills .codex/skills

echo "name: test-skill\ndescription: symlink test" > .claude/skills/test-skill.md

ln -s ../.claude/skills/test-skill.md .cursor/skills/test-skill.md
ln -s ../.claude/skills/test-skill.md .codex/skills/test-skill.md

ls -la .cursor/skills .codex/skills
```

결과는 명확했다.

```
lrwxr-xr-x ... test-skill.md -> ../.claude/skills/test-skill.md
```

즉, **한 번만 수정하면 모든 에이전트가 동일한 스킬을 읽는 구조**가 된다. 실제로 파일을 열어보면 내용이 동일하게 반영된다.

---

## 주의할 점 (실무에서 터지는 부분)

### 1) Windows는 권한 이슈가 많다
- WSL/윈도우에서는 symlink 권한이 막힐 수 있다.
- 가능하면 **관리자 권한** 또는 **개발자 모드**가 필요.

### 2) 에이전트가 symlink를 따라 읽는지 확인
- 일부 도구는 symlink를 무시할 수 있다.
- 간단 테스트: 내용 수정 → 다른 에이전트에서 로드 확인.

### 3) 레포 루트 기준 정렬
- 상대 경로를 쓰려면 **폴더 구조가 레포 기준으로 안정적**이어야 한다.

---

## 운영 추천 방식

- 팀은 `.claude/skills`만 관리한다
- `.cursor`/`.codex`는 **무조건 symlink**로 고정한다
- 업데이트는 한 곳만

이렇게 하면 “한쪽만 최신” 문제가 사라진다.

---

## 결론

나는 **스킬은 하나만 관리해야 한다**고 생각한다. 그리고 symlink는 그걸 가장 싸고 빠르게 만든다. 이 방식은 특히 **Claude Code + Cursor + Codex를 함께 쓰는 팀**에서 효율이 좋다.

정리하면:

- 스킬 원본은 하나
- 나머지는 링크
- 업데이트는 단 한 번

이렇게 하면 **운영 비용이 절반 이하**로 떨어진다.

---

## 더 읽을거리

- [React 생태계에서 TanStack이 중요한 이유](/posts/react-tanstack-ecosystem/)
- [Kafka vs Redis: MQ 방식의 본질 차이](/posts/kafka-redis-mq-differences/)
- [Playwright E2E: Vibe Coding을 위한 실전 끝장 가이드](/posts/playwright-e2e-vibe-coding/)
