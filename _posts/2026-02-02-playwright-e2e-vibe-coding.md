---
title: "Vibe Coding을 위한 Playwright E2E: 실전 끝장 가이드"
date: 2026-02-02 00:05:00 +0900
categories: [배운 것]
tags: [playwright, e2e, testing, vibe-coding, frontend]
description: "Vibe Coding 팀을 위한 Playwright E2E 테스트 설계, 구조, 안정화, CI 파이프라인까지 실무 기준으로 길고 촘촘하게 정리했습니다."
image:
  path: /assets/img/posts/2026-02-02-playwright-e2e-vibe-coding-cover.png
  alt: "브라우저 E2E 테스트 자동화 흐름"

---


# Vibe Coding을 위한 Playwright E2E: 실전 끝장 가이드

Vibe Coding은 빠르다. 대신, **깨지기 쉬운 제품을 만들 위험**이 커진다. 나는 그래서 말한다. **Vibe Coding의 생명줄은 E2E 테스트다.**

여기서 말하는 E2E는 “한두 개 돌려보는 테스트”가 아니다. **실제 사용 흐름을 끝까지 검증하고, 팀이 마음 편히 속도를 유지하게 해주는 안전장치**다. Playwright는 그 목적에 가장 적합한 도구다. 이유는 **속도, 신뢰성, 디버깅, 브라우저 커버리지**가 균형이 맞기 때문이다.

이 글은 **아주 길고**, **아주 상세**하게 간다. “조사 확실히” 요청이었으니, **설계부터 운영까지** 풀세트로 정리한다.

---

## 1. Vibe Coding에서 E2E가 왜 필수인가

Vibe Coding은 “빠른 실험”을 전제로 한다. 이 말은 곧 **리팩터링, 기능 변경, UI 재구성이 잦다**는 의미다. 이런 환경에서 유닛/통합 테스트만으로는 막을 수 없는 사고가 매번 난다.

E2E는 다음을 보장한다.

- **유저 동선이 실제로 동작하는지**
- **모든 마이크로 변경이 깨뜨리는 지점이 없는지**
- **릴리스 속도와 안정성의 균형**

E2E가 없으면 “빠른 속도”는 “빠른 사고”로 이어진다.

---

## 2. Playwright를 선택해야 하는 이유

Playwright는 단순한 브라우저 자동화가 아니다. 실무 관점에서 강력한 이유는 다음이다.

- **멀티 브라우저(Chromium/Firefox/WebKit) 지원**
- **헤드리스/헤드풀 테스트의 빠른 전환**
- **auto-waiting, retry, trace** 같은 안정화 기능
- **강력한 디버깅 스택(trace, video, screenshot)**
- **Parallel 실행 + shard 분산 지원**

즉, “테스트가 깨진다”는 문제를 **기술적으로 줄일 수 있는 장치가 다 있다.**

---

## 3. 큰 그림: E2E 설계 철학

E2E는 많이 만들수록 좋은 게 아니다. **딱 필요한 만큼만** 있어야 한다. 그리고 **각 테스트는 제품의 ‘동맥’만 테스트**해야 한다.

내 기준은 이렇다.

- **결제/가입/로그인** 같은 핵심 흐름만 E2E
- 나머지는 통합/컴포넌트 테스트로 분산
- E2E는 “큰 흐름 10~20개”에 집중

Vibe Coding은 테스트도 “속도 중심”으로 설계해야 한다.

---

## 4. 구조 설계: 폴더와 파일 레이아웃

Playwright는 구조가 곧 안정성이다. 추천 구조는 아래다.

```
/tests
  /e2e
    /flows
      auth.spec.ts
      checkout.spec.ts
    /pages
      login.page.ts
      cart.page.ts
    /fixtures
      user.fixture.ts
    /utils
      data.helper.ts
      wait.helper.ts
playwright.config.ts
```

- **flows**: 실제 사용자 시나리오 테스트
- **pages**: Page Object
- **fixtures**: 재사용되는 계정/데이터
- **utils**: 반복되는 helper

“테스트도 코드다.” 코드 구조가 지저분하면 테스트는 반드시 망가진다.

---

## 5. 설치 및 기본 설정

```bash
npm init -y
npm i -D @playwright/test
npx playwright install
```

**playwright.config.ts** 기본 구조는 이렇게 시작한다.

```ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './tests/e2e',
  timeout: 30_000,
  retries: 1,
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    video: 'retain-on-failure',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'chromium', use: { browserName: 'chromium' } },
    { name: 'firefox', use: { browserName: 'firefox' } },
    { name: 'webkit', use: { browserName: 'webkit' } },
  ],
})
```

이 설정만으로도 **실패 시 영상+트레이스**가 남는다. 디버깅 속도가 미친 듯이 올라간다.

---

## 6. 안정적인 셀렉터 전략

E2E의 80%는 셀렉터에서 망한다. UI 바뀌면 깨지는 테스트는 쓸모 없다. **가장 안전한 방식은 data-testid다.**

```tsx
<button data-testid="login-submit">로그인</button>
```

```ts
await page.getByTestId('login-submit').click()
```

이게 가장 강력한 이유는 **UI 변경과 무관하게 유지**되기 때문이다. 클래스나 텍스트 기반 셀렉터는 “언제든 깨질 수 있다.”

---

## 7. Page Object 패턴: 필수다

Vibe Coding 팀은 화면이 자주 바뀐다. 이때 Page Object 없으면 테스트는 지옥이 된다.

```ts
// login.page.ts
export class LoginPage {
  constructor(private page) {}

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, pw: string) {
    await this.page.getByTestId('login-email').fill(email)
    await this.page.getByTestId('login-password').fill(pw)
    await this.page.getByTestId('login-submit').click()
  }
}
```

테스트는 이렇게 짧아진다.

```ts
import { test, expect } from '@playwright/test'
import { LoginPage } from '../pages/login.page'

test('로그인 성공', async ({ page }) => {
  const login = new LoginPage(page)
  await login.goto()
  await login.login('user@test.com', 'pw1234')
  await expect(page).toHaveURL('/dashboard')
})
```

테스트 유지 보수 비용이 절반으로 줄어든다.

---

## 8. 테스트 데이터 전략

E2E에서 가장 힘든 건 “데이터 정합성”이다. 내가 쓰는 기본 전략은 다음이다.

1) 테스트 전용 계정 만들기
2) API/DB 초기화 엔드포인트 만들기
3) 테스트 시작 전에 초기 상태로 리셋

```ts
// tests/e2e/utils/data.helper.ts
export async function resetTestData(request) {
  await request.post('/test/reset')
}
```

```ts
test.beforeEach(async ({ request }) => {
  await resetTestData(request)
})
```

**이게 없으면, 테스트는 절대 안정적이지 않다.**

---

## 9. 인증 처리: 세션 재사용이 핵심

로그인 흐름을 모든 테스트에서 반복하면 테스트 시간이 폭발한다. Playwright는 **스토리지 상태 저장**이 가능하다.

```ts
// global-setup.ts
import { chromium } from '@playwright/test'

export default async function globalSetup() {
  const browser = await chromium.launch()
  const page = await browser.newPage()
  await page.goto('http://localhost:3000/login')
  await page.fill('[data-testid=login-email]', 'user@test.com')
  await page.fill('[data-testid=login-password]', 'pw1234')
  await page.click('[data-testid=login-submit]')
  await page.context().storageState({ path: 'storage.json' })
  await browser.close()
}
```

```ts
// playwright.config.ts
use: {
  storageState: 'storage.json',
}
```

이렇게 하면 로그인 테스트를 제외한 모든 테스트가 **즉시 인증 상태**로 시작한다.

---

## 10. 네트워크 안정화: mock vs real

E2E에서도 네트워크는 변수가 된다. 실무에서 나는 두 가지를 섞는다.

- **핵심 흐름은 real API**
- 불안정한 외부 API는 **mock**

```ts
await page.route('**/external-api/**', route => {
  route.fulfill({ status: 200, body: JSON.stringify({ ok: true }) })
})
```

이렇게 하면 외부 장애에 테스트가 흔들리지 않는다.

---

## 11. 플래키 테스트 줄이는 실전 팁

플래키 테스트는 신뢰를 깨뜨린다. 내가 실무에서 쓰는 원칙은 이렇다.

- **무조건 auto-waiting을 믿고, 명시적 wait를 최소화**
- `page.waitForTimeout`은 되도록 쓰지 않는다
- 비동기는 `toHaveText`, `toHaveURL` 등 **expect 기반 대기**로 처리

```ts
await expect(page.getByTestId('toast')).toHaveText('저장 완료')
```

이게 가장 안정적이다.

---

## 12. 병렬 실행과 샤딩

테스트는 많아질수록 병렬 실행이 필수다. Playwright는 기본적으로 병렬이다.

```bash
npx playwright test --workers=4
```

더 크게 가면 shard를 쓴다.

```bash
npx playwright test --shard=1/3
npx playwright test --shard=2/3
npx playwright test --shard=3/3
```

CI에서도 동일하게 쪼개면 속도가 크게 줄어든다.

---

## 13. CI 파이프라인 구성

GitHub Actions 예시다.

```yaml
name: E2E
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npx playwright install --with-deps
      - run: npm run dev &
      - run: npx playwright test
```

여기서 포인트는 **테스트 실행 전에 서비스가 켜져야 한다**는 점이다. 백엔드가 필요하다면 docker-compose로 올린다.

---

## 14. 실무에서 제일 중요한 것: 실패 로그의 품질

테스트 실패 자체는 문제 아니다. 문제는 **왜 실패했는지 알 수 없다는 것**이다.

Playwright의 trace는 이 문제를 해결한다.

```bash
npx playwright show-trace trace.zip
```

이걸 쓰면 **실패 순간의 DOM, 네트워크, 스크린샷**이 다 재현된다. 디버깅 속도가 압도적으로 빨라진다.

---

## 15. 도입 체크리스트 (실무자용)

- [ ] data-testid 전략을 전 팀에 합의했는가?
- [ ] E2E는 핵심 흐름 10~20개로 제한했는가?
- [ ] 테스트용 데이터 리셋 방식을 구축했는가?
- [ ] 로그인 세션 재사용이 가능한가?
- [ ] 플래키 테스트 모니터링을 하고 있는가?
- [ ] 실패 로그가 trace/video로 남는가?

---

## 결론: Vibe Coding의 안전장치는 E2E다

Vibe Coding은 속도를 만든다. 하지만 속도만 믿으면 결국 무너진다. **E2E는 그 속도를 지키는 방패다.**

내가 말하고 싶은 건 단 하나다. **Playwright는 “테스트 도구”가 아니라 “개발 속도 유지 장치”다.**

빠르게 만들고, 빠르게 검증하라. 그게 Vibe Coding의 유일한 생존법이다.

---

## 더 읽을거리

- [지수 백오프에 지터를 넣는 이유: 재시도 폭주 막기](/posts/exponential-backoff-jitter/)
- [매시간 크론 잡 로그, 이렇게 안 터뜨린다](/posts/hourly-cron-log-rotation/)
- [Docker 샌드박스 실행: 네트워크 차단 PoC](/posts/sandboxed-docker-poc/)
