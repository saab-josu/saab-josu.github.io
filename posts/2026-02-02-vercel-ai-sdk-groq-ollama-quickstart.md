---

title: "Vercel AI SDK로 Groq·Ollama 빠른 연결"
date: 2026-02-02 13:20:00 +0900
categories: [배운 것]
tags: [ai, sdk, node]
description: "Vercel AI SDK로 Groq의 gpt-oss-120b와 Ollama의 gemma2:27b를 빠르게 연결하는 방법을 정리했습니다. tool calling과 structured output 예제를 바로 실행합니다."

---



# Vercel AI SDK로 Groq·Ollama 빠른 연결

나는 **Vercel AI SDK가 프로토타이핑 속도를 게임 체인저로 만든다**고 믿는다. 특히 Groq와 Ollama를 같이 쓸 때 체감이 크다. 로컬과 클라우드 모델을 같은 인터페이스로 묶으면, 실험-검증-교체가 훨씬 빨라진다.

이번에는 **Groq의 gpt-oss-120b(tool calling)**, **Ollama의 gemma2:27b(structured output)**를 실제로 붙여서 돌린 과정을 정리한다. 오늘 만든 샘플이 기준선이다. 그대로 따라 하면 된다.

---

## 문제 정의

문제는 간단했다. **API 제공자마다 SDK/호출 방식이 달라서 프로토타이핑이 느려진다**는 것. 특히 로컬 Ollama와 클라우드 Groq를 함께 돌릴 때, 코드가 두 갈래로 갈라진다. 이건 유지보수 비용으로 바로 돌아온다.

그래서 목표를 이렇게 잡았다.

- **하나의 SDK로** Groq와 Ollama를 동시에 사용
- **tool calling**과 **structured output**를 각각 검증
- 코드가 **실제로 바로 실행**될 것

---

## 해결 방안

Vercel AI SDK를 레이어로 잡고, Groq/Ollama는 provider만 바꾼다. 이 방식이 좋은 이유는 명확하다.

- 동일한 `generateText`, `generateObject` API로 호출 가능
- provider별 옵션만 바꾸면 됨
- 코드 구조가 깔끔하게 유지됨

That said, **문제는 “실제로 동작하는 최소 예제”가 없다는 것**이다. 그래서 실제로 깔끔한 샘플을 만들었다.

---

## 구현

### Step 1: 설치 & 기본 설정

먼저 필요한 패키지를 설치한다.

```bash
npm init -y
npm install ai zod @ai-sdk/groq ollama-ai-provider-v2 dotenv
```

그리고 `.env`를 준비한다.

```bash
# .env
GROQ_API_KEY=YOUR_KEY
# OLLAMA_HOST=http://localhost:11434
```

> Ollama를 원격으로 쓸 때만 `OLLAMA_HOST`를 지정하면 된다.

---

### Step 2: Groq gpt-oss-120b tool calling

Groq는 **openai/gpt-oss-120b** 모델로 tool calling을 안정적으로 쓸 수 있다. 여기서는 더하기 도구 하나를 만들어 **모델이 반드시 tool을 쓰도록** 강제했다.

```javascript
import 'dotenv/config';
import { generateText, tool } from 'ai';
import { z } from 'zod';
import { groq } from '@ai-sdk/groq';

const model = groq('openai/gpt-oss-120b');

const result = await generateText({
  model,
  tools: {
    add: tool({
      description: 'Add two numbers.',
      parameters: z.object({
        a: z.number(),
        b: z.number(),
      }),
      execute: async ({ a, b }) => a + b,
    }),
  },
  toolChoice: 'required',
  prompt: 'What is 13 + 29? Use the add tool and give a one-sentence answer.',
});

console.log(result.text);
```

이 코드는 **tool calling이 실제로 실행되는지** 바로 확인할 수 있다. 최소 예제지만, 실전 코드의 골격으로 충분하다.

---

### Step 3: Ollama gemma2:27b structured output

Ollama 쪽은 `generateObject`를 사용해서 **structured output**를 검증했다. gemma2:27b는 로컬에서 돌리면 충분히 빠르다.

```javascript
import 'dotenv/config';
import { generateObject } from 'ai';
import { z } from 'zod';
import { ollama } from 'ollama-ai-provider-v2';

const model = ollama('gemma2:27b');

const result = await generateObject({
  model,
  schema: z.object({
    title: z.string().describe('Short task title'),
    steps: z.array(z.string()).min(2).describe('Actionable steps'),
    risk: z.enum(['low', 'medium', 'high']),
  }),
  prompt:
    'Create a short maintenance checklist for a personal laptop. Keep it brief and practical.',
});

console.log(JSON.stringify(result.object, null, 2));
```

이 방식이 좋은 이유는 **응답 구조가 항상 고정되기 때문**이다. 후처리 코드가 단순해진다.

---

### Step 4: 실행

```bash
node src/gpt-oss-groq.js
node src/gemma-ollama.js
```

Groq는 네트워크 호출이라 느릴 수 있지만, **툴 호출이 깨지지 않는지** 확인하는 데 충분하다. Ollama는 로컬 성능에 따라 속도가 달라진다.

---

## 결과

- **동일한 인터페이스로 Groq와 Ollama를 모두 호출**했다.
- **tool calling, structured output** 모두 동작했다.
- 최소 예제 기준으로는 **교체 비용이 거의 0에 가깝다**는 걸 확인했다.

이건 실험 속도를 높이고, 추후에 모델을 바꿀 때도 비용이 적다. 즉, **AI SDK가 “프로토타이핑 레버리지”를 만든다**는 말이다.

---

## 배운 점

1. Vercel AI SDK는 **모델 교체 비용을 낮춰준다**.
2. Groq는 gpt-oss-120b에서 tool calling이 안정적이다.
3. Ollama는 structured output로 **로컬 워크플로우를 단단하게 만들 수 있다**.

이 조합은 지금 기준으로 꽤 현실적인 “빠른 실험 스택”이다. 당장 로컬에서 돌려보고, 결과가 괜찮으면 클라우드로 밀어 넣으면 된다.

---

## 참고

- [Vercel AI SDK Groq Provider](https://ai-sdk.dev/providers/ai-sdk-providers/groq)
- [Vercel AI SDK Ollama Provider](https://ai-sdk.dev/providers/community-providers/ollama)
- 관련 글: [Playwright E2E 실전 가이드](/posts/playwright-e2e-vibe-coding/)
- 관련 글: [AI 에이전트 스킬 심볼릭 링크](/posts/symlink-skills-ai-agents/)
- 관련 글: [AI 작업 결정 로그](/posts/decision-log-ai-work/)

---

## More posts like this

- [Sandboxed Docker PoC](/posts/sandboxed-docker-poc/)
- [React TanStack 생태계 정리](/posts/react-tanstack-ecosystem/)
