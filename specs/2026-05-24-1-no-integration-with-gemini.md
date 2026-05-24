# Planning Spec: Gemini Integration

**Issue:** [#1 - no integration with gemini](https://github.com/VIVAAN-DHAWAN/ai-study-companion/issues/1)
**Date:** 2026-05-24
**Status:** Draft

---

## Problem

The AI Study Companion has no integration with Google's Gemini models. The current codebase is a scaffolding of `create-next-app` — no AI provider is configured. Attempts to set up Gemini via CLI produce errors, and neither `gemini-1.5-flash` nor `gemini-2.0-pro` (or the models referenced as 3.5 flash / 3.1 pro) work.

## Goals

1. Add the `@google/generative-ai` SDK dependency.
2. Implement a client-side chat or query interface that uses Gemini models.
3. Support both `gemini-1.5-flash` (fast, cheap) and `gemini-2.0-pro` (higher quality) with an easy model switcher.
4. Provide a `.env.local` setup flow with clear instructions so CLI setup works (no cryptic errors).
5. Store user's API key in a way that is secure and does not leak to the client.

## Non-goals

- Streaming responses (can be added later).
- Authentication / user accounts.
- Persistent chat history beyond session state.
- Integration with other AI providers (OpenAI, Anthropic) — Gemini only.

## Approach

### 1. Environment & Setup

- Add `@google/generative-ai` to `package.json` dependencies.
- Document setup in `README.md`:
  1. Get a Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey).
  2. Create `.env.local` with `GEMINI_API_KEY=your_key`.
  3. Run `npm install && npm run dev`.

### 2. API Route: `/api/chat`

- Create `src/app/api/chat/route.ts` (Next.js App Router route handler).
- Accept `POST` with JSON body: `{ message: string, model: "gemini-1.5-flash" | "gemini-2.0-pro" }`.
- Use `GoogleGenerativeAI` from `@google/generative-ai` to send the prompt.
- Return the model's response text as `{ reply: string }`.
- Handle errors gracefully (invalid key, rate limit, model unavailable).

### 3. UI Component: Chat Interface

- Create `src/components/ChatPanel.tsx` — a simple chat UI (message list + input box + model selector dropdown).
- Wire it into `src/app/page.tsx` or a new `/chat` route.
- Show clear error messages when the API key is missing or a model call fails.

### 4. Model Aliasing

The user reports issues with "3.5 flash" and "3.1 pro". These likely refer to:
- `gemini-1.5-flash` (commonly called Gemini 1.5 Flash)
- `gemini-2.0-pro` (commonly called Gemini 2.0 Pro)

If Google releases models with different version strings, we keep the model IDs in a config map so they can be updated in one place.

## Risks

| Risk | Mitigation |
|------|-----------|
| API key exposed to client | Server-side API route only; key never sent to browser |
| Model version strings change | Centralized model config map |
| Rate limiting / quota | Show user-friendly error; suggest upgrading tier |
| User doesn't have a key / can't get one | Clear setup docs; detect missing key at runtime and show helpful message |
| `@google/generative-ai` SDK breaking changes | Pin the dependency version; test on upgrade |

## Validation

1. **Setup flow works:** following README instructions, a new developer can get the app running with Gemini in under 5 minutes.
2. **Both models respond:** send a test prompt ("Hello, what model are you?") using `gemini-1.5-flash` and `gemini-2.0-pro`. Both return correct replies.
3. **Error handling:** remove the API key — the UI shows a clear "missing API key" message. Send an invalid key — the UI shows an auth error.
4. **`npm run build` succeeds** with no TypeScript or lint errors.

## Implementation Order

1. Add `@google/generative-ai` dependency.
2. Create `.env.local.example` with `GEMINI_API_KEY=`.
3. Create `src/config/models.ts` with model aliases.
4. Create `src/app/api/chat/route.ts`.
5. Create `src/components/ChatPanel.tsx`.
6. Wire ChatPanel into `src/app/page.tsx`.
7. Update `README.md` with setup instructions.
8. Verify build (`npm run build`).

---

*Closes #1 when merged.*
