# Spec: Test Looper — Add Custom Model Selection Dropdown

**Issue:** [#3](https://github.com/VIVAAN-DHAWAN/ai-study-companion/issues/3)
**Date:** 2026-05-24
**Author:** Looper Planner (opencode)

---

## Problem

The AI Study Companion currently has no way for users to choose which LLM model powers their chat experience. A hardcoded or invisible model selection frustrates users who want to switch between providers (OpenAI, Anthropic, Google, etc.) or between model tiers (fast vs. capable, cheap vs. high-quality). Without a visible, interactive model selector the app feels opaque and inflexible.

## Goals

1. **Visually polished model selection dropdown** — modern, minimal, fits the study companion aesthetic, works on desktop and mobile.
2. **Multiple provider support** — dropdown lists models from at least two providers (e.g., OpenAI GPT-4o, GPT-4o-mini, Anthropic Claude Sonnet, Haiku, etc.).
3. **Selection persists** — chosen model survives page reloads (localStorage or similar).
4. **Integrates with chat** — selected model is sent alongside each chat message so the backend or client-side SDK routes to the correct API.
5. **Zero visual regression** — dropdown does not break existing layout or feel out of place.

## Non-Goals

- Building the full chat backend / API routes (scoped to the UI only, though the spec sketches the integration contract).
- Implementing streaming chat responses.
- Authentication or API-key management UI.
- Usage quotas, rate-limit displays, or cost tracking.

## Approach

### 1. Component: `<ModelSelect />`

Create `src/components/model-select.tsx` as a client component.

- **Trigger:** A button showing the currently selected model name + a small chevron icon.
- **Dropdown panel:** Absolute-positioned card with a list of model options grouped by provider. Each option shows the model name and a subtle provider label.
- **Interaction:** Click to open, click an option to select + close, click outside to close. Keyboard navigation (arrow keys, Enter, Escape).
- **Styling:** Tailwind CSS v4, matches the app's design tokens (checking globals.css for any CSS variables). Uses `@tailwindcss/postcss` conventions.

**States handled:**
- Default — shows current selection
- Open — dropdown visible, user can pick
- Disabled — while a chat message is being sent, prevent switching
- Empty / error — if model list fails to load, show fallback

### 2. State Management

```ts
// src/lib/model-config.ts (types and constants)

type ModelProvider = "openai" | "anthropic" | "google";

interface ModelOption {
  id: string;         // e.g. "gpt-4o"
  label: string;      // e.g. "GPT-4o"
  provider: ModelProvider;
  description?: string;
}

const AVAILABLE_MODELS: ModelOption[] = [
  { id: "gpt-4o",        label: "GPT-4o",         provider: "openai",   description: "Best all-rounder" },
  { id: "gpt-4o-mini",   label: "GPT-4o mini",     provider: "openai",   description: "Fast & cheap" },
  { id: "claude-sonnet", label: "Claude Sonnet",   provider: "anthropic",description: "Balanced" },
  { id: "claude-haiku",  label: "Claude Haiku",    provider: "anthropic",description: "Lightning fast" },
  { id: "gemini-2.0-flash", label: "Gemini Flash", provider: "google",  description: "Fast & capable" },
];

const DEFAULT_MODEL = "gpt-4o-mini";
```

Selection is managed via React state (useState / useReducer) in the chat page, persisted to `localStorage` under a `model-preference` key.

### 3. Integration Contract

The dropdown exposes an `onModelChange(modelId: string)` callback. The parent (chat page or `ChatShell` component) stores the selection and passes it down to:
- The message input component (for display context)
- The API call function (as a header or body field)

```ts
// Hypothetical usage in the chat page
const [activeModel, setActiveModel] = useState(DEFAULT_MODEL);

<ModelSelect
  selected={activeModel}
  onSelect={setActiveModel}
  disabled={isSending}
/>

// Later, when sending:
POST /api/chat
Body: { message: "...", model: activeModel }
```

### 4. UI Placement

The dropdown sits inline with the chat input bar — either as a left-aligned button before the text input, or as a compact badge inside the input bar. A reference layout:

```
┌──────────────────────────────────────────────┐
│  [GPT-4o ▼]  │  Type your question...  [Send] │
└──────────────────────────────────────────────┘
```

On mobile, the same pattern stacks or the dropdown remains accessible via tap.

### 5. Files to Create / Modify

| File | Action | Purpose |
|------|--------|---------|
| `src/lib/model-config.ts` | Create | Model list, types, constants |
| `src/components/model-select.tsx` | Create | Client-side dropdown component |
| `src/components/model-select.test.tsx` | Create | Tests (if testing infra set up) |
| `src/app/page.tsx` | Modify | Integrate ModelSelect into the chat page |
| `specs/2026-05-24-3-test-looper-add-custom.md` | Create | This spec |

## Dependencies

No new npm packages required. The dropdown is built entirely with:
- React 19 `useState`, `useRef`, `useEffect`, `useCallback`
- Tailwind CSS v4 utility classes
- Native HTML `<button>` and list elements for a11y

## Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Scope creep into full chat backend | Medium | High | Clearly define UI-only scope; stub API contract |
| Model list becomes stale / needs dynamic fetching | Low | Medium | Start with static list; refactor to dynamic later |
| Accessible dropdown is harder to get right than a `<select>` | Medium | Low | Use ARIA `listbox` pattern; test with keyboard |
| Persisted selection references a model that no longer exists | Low | Low | Fall back to `DEFAULT_MODEL` on load if stale |

## Validation

1. **Visual:** Dropdown renders correctly at 375px, 768px, and 1440px widths.
2. **Functional:** Each model option is selectable via click and keyboard (Enter/Space). Selection persists after page reload.
3. **Integration:** The selected `modelId` is passed correctly to the chat input's API call (inspect network tab).
4. **Edge cases:** Rapid clicking does not break state. Disabling during "sending" prevents switching. Long model names do not overflow.
5. **A11y:** Screen reader announces the model button and selected option. ARIA attributes on the dropdown panel.
