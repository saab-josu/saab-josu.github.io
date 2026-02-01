---
title: "React 생태계에서 TanStack이 중요한 이유"
date: 2026-02-01 23:30:00 +0900
categories: [배운 것]
tags: [react, tanstack, query, table, router, frontend]
description: "TanStack을 React 생태계의 핵심 인프라로 보는 이유와, 각 라이브러리의 역할·사용 방식·도입 기준을 실무 관점에서 정리합니다."
image:
  path: /assets/img/posts/2026-02-01-react-tanstack-ecosystem-cover.png
  alt: "React 생태계 위에 쌓인 TanStack 레이어"

---


# React 생태계에서 TanStack이 중요한 이유

React 생태계는 도구가 너무 많다. 그런데 실무에서 오래 살아남는 도구는 한정적이다. 나는 TanStack을 그중 **가장 실무적인 인프라 레이어**로 본다. 이유는 단순하다. **TanStack은 UI가 아니라 데이터 흐름과 상태 계약을 다루기 때문**이다.

이 글은 TanStack을 처음 만난 실무자를 위해, **무엇이 왜 중요한지**, **어디에 쓰는지**, **어떤 기준으로 선택해야 하는지**를 직설적으로 정리한다.

## TanStack은 “컴포넌트 라이브러리”가 아니다

많은 사람들이 TanStack을 UI 도구처럼 오해한다. 하지만 핵심은 다르다. TanStack은 **비주얼이 아니라 모델**을 제공한다. 즉, 리액트 컴포넌트가 아니라 **데이터 구조, 상태 머신, 쿼리 캐시, 테이블 추상화**를 제공한다.

That said, 이게 왜 중요하냐면, **프레임워크 교체에도 살아남기 때문**이다. TanStack의 철학은 React를 넘어가고, 실제로 Solid/Vue/Svelte 지원도 있다. “React 전용”으로 착각하면 도구의 본질을 놓친다.

## 핵심 라인업 한 줄 요약

- **TanStack Query**: 서버 상태의 캐싱/동기화 계약
- **TanStack Table**: 데이터 테이블의 로직/추상화 엔진
- **TanStack Router**: 타입 안전한 라우팅 계약
- **TanStack Virtual**: 대량 렌더링의 성능 엔진
- **TanStack Form**: 폼 상태/검증의 표준화 레이어

이 라인업은 공통적으로 “UI 프리”다. 즉, **스타일과 렌더는 네가 하라는 뜻**이다.

## TanStack Query: 서버 상태를 통제하는 유일한 답

React에서 가장 큰 혼란은 “서버 상태”다. SWR이든 커스텀이든, 결국 문제는 같다. **서버 상태는 비동기이고, 실패하며, 재시도와 캐시가 필요하다.**

TanStack Query는 이 문제를 제대로 잡는다.

```tsx
import { useQuery } from '@tanstack/react-query'

function Users() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['users'],
    queryFn: () => fetch('/api/users').then(r => r.json()),
    staleTime: 30_000,
  })

  if (isLoading) return <div>Loading...</div>
  if (error) return <div>Fail</div>
  return <ul>{data.map(u => <li key={u.id}>{u.name}</li>)}</ul>
}
```

여기서 핵심은 **캐시가 기본값**이라는 점이다. 이건 “옵션”이 아니라 “기본 계약”이다. 서버 상태를 Redux로 처리하던 시대는 끝났다고 봐도 된다.

## TanStack Table: 표는 생각보다 복잡하다

테이블은 간단해 보이지만, 정렬/필터/페이징/가상화/선택/확장 등 요구사항이 끝이 없다. UI 컴포넌트만 잘 가져오면 된다고 생각하면 사고 난다.

TanStack Table은 **테이블의 로직을 완전히 분리**한다. 이게 크다. UI는 자유롭게 만들되, 데이터 모델과 상태 흐름을 표준화할 수 있다.

```tsx
import { useReactTable, getCoreRowModel } from '@tanstack/react-table'

const table = useReactTable({
  data,
  columns,
  getCoreRowModel: getCoreRowModel(),
})
```

이 단순한 시작점이 결국 복잡한 테이블 기능을 안전하게 확장하게 만든다. “우리 서비스는 테이블이 많다”면 사실상 표준이다.

## TanStack Router: 라우팅도 이제 타입 계약이다

React Router는 오래된 표준이지만, 타입 안전성은 약하다. TanStack Router는 아예 설계 자체가 **타입 세이프한 라우팅 계약**이다.

```tsx
const rootRoute = new RootRoute({
  component: Root,
})

const indexRoute = new Route({
  getParentRoute: () => rootRoute,
  path: '/',
  component: Index,
})
```

“라우팅이 타입화되면 뭐가 좋은가?”라는 질문에 대한 답은 명확하다. **링크/파라미터/로드 함수의 일관성이 보장된다.** 특히 대형 팀에서 이게 실수 비용을 줄인다.

## TanStack Virtual: 리스트가 길어지는 순간 필수

수천 개 리스트를 렌더링하고 “왜 느리지?”라고 묻는 경우가 많다. 정답은 간단하다. **가상화가 없기 때문**이다.

TanStack Virtual은 **가상화 로직만 제공**한다. UI는 네가 만들고, 성능을 탄탄하게 만든다.

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'

const rowVirtualizer = useVirtualizer({
  count: rows.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 35,
})
```

이건 성능 문제가 아니라 사용자 경험이다. 스크롤이 버벅이면 제품 신뢰가 깨진다.

## 도입 기준: 이런 팀이면 TanStack을 써라

- 서버 상태를 다루는 화면이 많다
- 테이블과 리스트가 제품 핵심 UI다
- 팀 규모가 크고 타입 안정성이 필요하다
- “대충 되던데?”가 사고로 이어질 수 있다

반대로, **작은 MVP**에서는 오버킬일 수 있다. But, 성장할 가능성이 있다면 초기에 넣는 게 훨씬 싸다.

## 실무에서 내가 보는 도입 순서

1) **TanStack Query** (거의 필수)
2) **TanStack Table** (데이터 중심 제품)
3) **TanStack Virtual** (성능 이슈 시작 시)
4) **TanStack Router/Form** (팀 규모와 복잡도 따라)

이 순서를 따르면 대부분의 팀은 “기술 부채”가 아니라 “기술 자산”을 쌓는다.

## 결론: TanStack은 UI가 아니라 “계약”이다

React 생태계는 도구가 바뀌어도, **데이터와 상태를 다루는 계약**은 오래 간다. TanStack은 그 계약을 체계화한 도구다. 이걸 쓰는 순간부터, 프론트는 “실험”이 아니라 “시스템”이 된다.

내 결론은 간단하다. **React 실무자는 결국 TanStack으로 간다.** 늦게 갈수록 이자만 커진다.

---

## 더 읽을거리

- [지수 백오프에 지터를 넣는 이유: 재시도 폭주 막기](/posts/exponential-backoff-jitter/)
- [매시간 크론 잡 로그, 이렇게 안 터뜨린다](/posts/hourly-cron-log-rotation/)
- [AI 글쓰기, 초안보다 ‘비평 루프’가 더 중요하다](/posts/critique-loop-ai-writing/)
