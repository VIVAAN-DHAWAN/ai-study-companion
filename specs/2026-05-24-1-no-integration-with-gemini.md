# Planning Spec: Gemini Integration

**Issue:** [#1 - no integration with gemini](https://github.com/VIVAAN-DHAWAN/ai-study-companion/issues/1)
**Date:** 2026-05-24
**Status:** Draft

---

## Problem

The AI Study Companion has no integration with Google's Gemini models. The codebase is a clean `create-next-app` scaffold — no AI provider is configured, no API routes exist, and there is no chat UI.

The user reports that CLI-based setup fails with errors and that model names `3.5-flash` and `3.1-pro` do not work. This is caused by:

1. **No SDK installed** — the `@google/generative-ai` package is not in `package.json`.
2. **Wrong model IDs** — valid Gemini model IDs are `gemini-1.5-flash` and `gemini-2.0-pro` (or `gemini-1.5-pro`). The strings `3.5-flash` and `3.1-pro` are not valid API model identifiers.
3. **No setup documentation** — the README does not explain how to obtain an API key or configure the app.
4. **No error feedback** — when the API key is missing or a model call fails, there is no UI to surface the error.

## Goals

1. Install and configure `@google/generative-ai` SDK.
2. Implement a server-side API route (Next.js App Router) that proxies requests to the Gemini API, keeping the API key secure.
3. Build a client-side chat interface with a model switcher supporting `gemini-1.5-flash` (fast, cheap) and `gemini-2.0-pro` (higher quality).
4. Provide clear `.env.local` setup instructions in README so CLI-based setup works without errors.
5. Surface clear error messages for missing API keys, invalid keys, rate limits, and unavailable models.

## Non-goals

- Streaming responses (can be added later as a follow-up).
- Authentication / user accounts.
- Persistent chat history beyond in-memory session state.
- Integration with other AI providers (OpenAI, Anthropic) — Gemini only for now.

## Approach

### 1. Dependencies & Environment

- Add `@google/generative-ai` to `package.json` dependencies (pin a specific version for reproducibility).
- Create `.env.local.example` with `GEMINI_API_KEY=` as a template.
- Update `.gitignore` — `.env*` is already ignored, no change needed.
- Document the setup flow in README:
  1. Get a Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey).
  2. Copy `.env.local.example` to `.env.local` and paste the key.
  3. Run `npm install && npm run dev`.

### 2. Model Configuration (`src/config/models.ts`)

Create a single source of truth for model IDs:

```ts
export const GEMINI_MODELS = {
  flash: "gemini-1.5-flash",
  pro: "gemini-2.0-pro",
} as const;

export type GeminiModel = (typeof GEMINI_MODELS)[keyof typeof GEMINI_MODELS];
```

This maps the user-facing labels ("Flash", "Pro") to the actual API model strings. If Google releases new model versions, only this file needs to change.

### 3. API Route (`src/app/api/chat/route.ts`)

Server-side route handler using Next.js App Router conventions:

- `POST /api/chat` with JSON body `{ message: string, model: GeminiModel }`.
- Reads `GEMINI_API_KEY` from `process.env` (server-side only — key never reaches the browser).
- Instantiates `GoogleGenerativeAI` and calls `getGenerativeModel({ model }).generateContent()`.
- Returns `{ reply: string }` on success.
- Returns `{ error: string }` with an appropriate HTTP status code on failure:
  - `401` if `GEMINI_API_KEY` is missing.
  - `403` if the API key is invalid (auth error from SDK).
  - `429` if rate-limited.
  - `400` if the model ID is invalid.
  - `500` for unexpected errors.
- Validate the request body — reject empty `message` with 400, reject unknown `model` values against the config map with 400, and verify `Content-Type: application/json`.

> **Note:** Each API call is stateless — only the current `message` is sent to Gemini. The UI maintains the message list locally for display but does not send conversation history to the API.

### 4. Chat UI (`src/components/ChatPanel.tsx`)

A client component with:

- A scrollable message list displaying user messages and model replies.
- A text input with send button.
- A model selector dropdown (Flash / Pro).
- Error banner that appears inline when an API call fails.
- Loading state with a disabled input and spinner while waiting for a response.

Wired into `src/app/page.tsx`, replacing the default scaffold content.

## Risks

| Risk | Mitigation |
|------|-----------|
| API key leaked to client | All Gemini API calls go through the server-side API route. The key is never in client bundles. |
| User types wrong model ID | The UI offers a fixed dropdown, not a free-text field. The model config map is the single source of truth. |
| User has no API key | README documents how to get a key. The UI detects a missing key at runtime and shows a clear "set up your API key" message. |
| Rate limiting / quota exceeded | API route catches 429 responses and returns a user-friendly message instead of crashing. |
| SDK breaking changes | Pin `@google/generative-ai` to a specific version in `package.json`. Review on upgrade. |
| Next.js 16 API changes | Route handler uses standard Web API `Request`/`Response` objects (stable across Next.js versions). |

## Validation

1. **Setup from scratch works:** a developer following the README can get the app running with Gemini in < 5 minutes.
2. **Both models respond:** send `"Hello, what model are you?"` using both `gemini-1.5-flash` and `gemini-2.0-pro`. Both return correct replies identifying themselves.
3. **Error handling:**
   - Remove `GEMINI_API_KEY` from `.env.local` → UI shows "API key not configured" message.
   - Set `GEMINI_API_KEY=invalid` → UI shows authentication error.
   - Send an empty message → client-side validation rejects before the API call.
4. **`npm run build` succeeds** with no TypeScript or lint errors.
5. **No API key in client bundle:** `rg GEMINI_API_KEY .next/static/` returns no results after a build.

## Implementation Order

1. Add `@google/generative-ai` dependency (`npm install @google/generative-ai`).
2. Create `src/config/models.ts` with model map.
3. Create `src/app/api/chat/route.ts`.
4. Create `src/components/ChatPanel.tsx`.
5. Update `src/app/page.tsx` to use ChatPanel.
6. Create `.env.local.example`.
7. Update `README.md` with setup instructions.
8. Verify with `npm run build`.

---

*Closes #1 when merged.*
