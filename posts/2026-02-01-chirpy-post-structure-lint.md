---
title: "Chirpy 포스트 구조 점검: 30초 린터"
date: 2026-02-01 17:00:00 +0900
categories: [blog, workflow]
tags: [chirpy, lint, markdown, workflow]
description: "Chirpy 포스트에 필수 섹션이 빠졌는지 30초 만에 점검하는 간단 린터와 사용법을 정리했습니다."
---

# Chirpy 포스트 구조 점검: 30초 린터

## TL;DR
- Chirpy 포스트는 구조가 어긋나면 품질이 빠르게 무너집니다.
- 40줄짜리 파이썬 린터로 필수 섹션 누락을 즉시 잡을 수 있습니다.
- CI에 붙이기 전에 로컬에서 먼저 돌려보세요.

## 배경/맥락
저는 글을 자주 쓰다 보니, 초안 단계에서 구조가 흐트러질 때가 많습니다. 특히 TL;DR, 체크리스트, 참고 링크 같은 부분은 마지막에 넣으려다 빠지기 쉽습니다. 이런 실수는 결국 글 품질과 일관성에 바로 영향이 갑니다. 그래서 아예 **구조 누락을 기계적으로 잡는 작은 린터**를 만들었습니다.

Chirpy 포스트는 파일명과 프론트 매터 규칙이 엄격합니다. 이 규칙을 놓치면 배포 이후 수정 비용이 올라갑니다. (Chirpy 공식 문서도 이 부분을 강조합니다.)

## 본문
### 어떤 구조를 검사할까
이번 린터는 아주 단순합니다. 제목(H1)과 아래 섹션이 있는지 먼저 확인합니다.
- TL;DR
- 배경/맥락
- 본문
- 실수/교훈/개선점 (선택)
- 체크리스트
- 참고 링크

저는 기존에 작성한 체크리스트 글과도 연결해서 흐름이 이어지도록 했습니다. 내부 링크를 걸어두면 글 묶음이 더 잘 보입니다. 예: [AI 워크플로 체크리스트](/posts/checklist-for-ai-workflows/), [지터 포함 재시도 패턴](/posts/exponential-backoff-jitter/)

### 40줄 린터 코드
아래 코드는 마크다운 파일 하나를 받아 구조 누락을 알려줍니다. 실전에서는 이걸 CI나 훅으로 붙이면 됩니다.

```python
import re
import sys
from pathlib import Path

REQUIRED = ["TL;DR", "배경/맥락", "본문", "체크리스트", "참고 링크"]
OPTIONAL = ["실수/교훈/개선점"]


def extract_headings(text):
    return [m.group(2).strip() for m in re.finditer(r"^(#{1,3})\s+(.+)$", text, re.M)]


def lint(path: Path):
    text = path.read_text(encoding="utf-8")
    headings = extract_headings(text)
    missing = [h for h in REQUIRED if h not in headings]
    optional_missing = [h for h in OPTIONAL if h not in headings]

    print(f"file: {path}")
    print("missing required:", ", ".join(missing) or "none")
    print("missing optional:", ", ".join(optional_missing) or "none")

    return 1 if missing else 0


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("usage: python lint.py <post.md>")
        raise SystemExit(2)
    raise SystemExit(lint(Path(sys.argv[1])))
```

### 로컬에서 30초 돌려보기
저는 도커로 실행했습니다. 로컬 환경을 오염시키지 않는 게 마음이 편하거든요.

```bash
docker run --rm -v "$PWD":/work -w /work python:3.12-slim \
  python lint.py posts/2026-02-01-chirpy-post-structure-lint.md
```

이 정도면 글 쓰기 전에 한 번만 돌려도 실수가 줄어듭니다. 그리고 비슷한 패턴은 [GitHub 블로그 운영 이유](/posts/why-github-blog/) 글에서도 이어집니다.

## 실수/교훈/개선점
처음에는 프론트 매터만 검사하려고 했습니다. 그런데 정작 빠지는 건 본문 구조였습니다. *틀을 체크하지 않으면, 글은 매번 제멋대로 흘러갑니다.* 이건 제가 매번 겪은 문제라서, 결국 린터로 고정했습니다.

## 체크리스트
- 새 글 파일명과 날짜 형식이 맞는가
- TL;DR과 체크리스트가 빠지지 않았는가
- 내부 링크 2개 이상 포함했는가
- 코드 블록은 언어가 지정됐는가
- 참고 링크가 실제로 유효한가

## 참고 링크
- Chirpy: Writing a New Post — https://chirpy.cotes.page/posts/write-a-new-post/
- Markdownlint 소개 — https://github.com/DavidAnson/markdownlint
