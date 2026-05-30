# Collective Intelligence — AI Debate Arena

A browser-based **AI debate arena**. Two AI debaters argue opposite sides of a single binary motion, while a panel of **AI judges — each with their own background, bias and decision criteria** — votes after every round. The result is a small experiment in *collective intelligence*: many opinionated, imperfect judges aggregating into a verdict.

Everything runs **on your device**. It uses Chrome's built-in **Prompt API** when available, and automatically falls back to Google **Gemma** running locally via WebGPU when it isn't. No API keys, no server, no data leaving your machine.

**[▶ Live Demo](https://woodyhoko.github.io/collective-intelligence)** · **[📖 Docs](https://woodyhoko.github.io/collective-intelligence/docs.html)**

---

## 1. What it does

- **Binary motions.** You enter one statement (e.g. *"This house believes remote work is better than office work"*). The app frames it as a clean two-sided debate:
  - 🔵 **Proposition · FOR** — argues the motion is true.
  - 🔴 **Opposition · AGAINST** — argues the motion is false.

  Both sides are listed explicitly so it's always clear who represents what.
- **A debate plays out.** The two debaters take turns (six statements), each rebutting the last.
- **A panel of judges votes every round.** Each judge weighs the transcript through *their own* personality and criteria, then casts a PRO/CON vote. Live tallies animate as a bar graph, and a round-by-round **transcript** records every statement and the vote split it produced.

---

## 2. The judge panel

The panel is fully editable — this is where the "collective intelligence" comes from.

| Action | How |
|---|---|
| **Add a judge** | Open *Add Judge*, then either roll one or generate from a role (below). |
| **Fully random** | One click invents a complete judge — name, role, background, bias and criteria — from a hand-tuned pool. No model needed. |
| **Assisted from a role** | Type a job/role (e.g. *"marine biologist"*) and the on-device model writes a vivid, opinionated judge around it. |
| **Customize before adding** | Whichever way you generate, every field is editable in the form *before* the judge joins the panel. |
| **Peek** | The 👁 button on any judge opens their full background, bias and criteria. |
| **Edit** | Peek → *Edit* to revise an existing judge. |
| **Reduce the panel** | The ✕ button removes a judge. Add as many or as few as you like (one minimum to debate). |

Because each judge interprets the same transcript differently, a 3-judge panel and a 9-judge panel can reach opposite verdicts on the same motion — which is the whole point.

---

## 3. Dual on-device backend

The app never calls a cloud API. At startup it probes for a backend and picks one automatically:

```javascript
const AI = {
  backend: null,
  async detect() {
    if (typeof window.LanguageModel !== 'undefined') {
      const avail = await window.LanguageModel.availability();
      if (avail && avail !== 'unavailable') { this.backend = 'chrome'; return 'chrome'; }
    }
    this.backend = 'gemma';            // fall back to on-device Gemma
    return 'gemma';
  }
};
```

### Backend A — Chrome built-in Prompt API

When the browser exposes `window.LanguageModel` (Chrome's built-in AI), the app uses it directly. It's instant, needs no download, and supports **structured output** via `responseConstraint`, which keeps judge votes strictly `PRO` / `CON`:

```javascript
const session = await window.LanguageModel.create();
const schema = { type: "object", properties: { vote: { enum: ["PRO", "CON"] } }, required: ["vote"] };
const raw = await session.prompt(prompt, { responseConstraint: schema });
```

### Backend B — On-device Gemma (fallback)

When there's no built-in AI, the app loads **Gemma** (`litert-community/gemma-4-E2B-it`) through MediaPipe Tasks GenAI on WebGPU — the same on-device approach as [match-maker-game](https://github.com/woodyhoko/match-maker-game). It's an **opt-in** one-time **≈ 2 GB** download, cached in the Origin Private File System (OPFS) so later visits skip it:

```javascript
import { FilesetResolver, LlmInference } from '@mediapipe/tasks-genai';

const fileset = await FilesetResolver.forGenAiTasks(
  'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-genai/wasm'
);
AI.gemmaLlm = await LlmInference.createFromOptions(fileset, {
  baseOptions: { modelAssetBuffer: tracked.getReader() },
  maxTokens: 1024, topK: 16, temperature: 0.85
});
```

A single `AI.run(prompt, { schema })` call hides the difference — Chrome honours the schema natively, while the Gemma path wraps the prompt in Gemma's turn format and parses the JSON/keyword out of the response.

---

## 4. Prompt design

**Debaters** receive their side, the motion, and the running transcript, and are asked for one concise rebutting argument:

```
You are the Proposition (FOR the motion).
Motion: "<motion>".
Debate so far: <transcript>
Give ONE concise, persuasive argument (1-2 sentences) for your side.
If your opponent just spoke, rebut them directly.
```

**Judges** receive their own persona plus the transcript and must return a constrained vote:

```
You are a debate judge.
Background: <background>   What sways you: <bias>   Your criteria: <criteria>
Motion: "<motion>".  PRO = arguing FOR, CON = arguing AGAINST.
Transcript so far: <transcript>
Through the lens of YOUR background, who is more convincing right now?
Answer with ONLY: {"vote": "PRO"} or {"vote": "CON"}.
```

On Chrome the `enum` schema guarantees a valid vote; on Gemma the response is parsed for `PRO`/`CON` with a keyword fallback, so a judge never silently drops out of a round.

---

## 5. Privacy model

| Data | Where it goes |
|---|---|
| The motion & judge profiles | Stay in browser memory only |
| LLM inference | Runs on-device (built-in model or Gemma in the WASM/WebGPU sandbox) |
| Gemma weights | Downloaded once from HuggingFace; cached locally in OPFS |
| Debate transcript & votes | Never transmitted anywhere |

There is **no telemetry, no logging, and no server** in the loop.

---

## 6. Stack

| Layer | Technology |
|---|---|
| Primary AI | [Chrome built-in Prompt API](https://developer.chrome.com/docs/ai/prompt-api) (`window.LanguageModel`) |
| Fallback AI | [Gemma](https://ai.google.dev/gemma) E2B (INT4 LiteRT build) on MediaPipe Tasks GenAI (WASM + WebGPU) |
| Model hosting | [litert-community on HuggingFace](https://huggingface.co/litert-community/gemma-4-E2B-it-litert-lm) (token-free) |
| Model cache | Origin Private File System (OPFS) |
| Styling | Custom CSS — glassmorphism, Outfit + Playfair Display, Font Awesome |
| Build | None — a single `index.html`, ES module imports |

---

## 7. Run locally

```bash
# Serve over http (WebGPU/OPFS are blocked on file://)
python -m http.server 8000
# open http://localhost:8000
```

- If your Chrome exposes the **built-in Prompt API**, the app uses it instantly — no download.
- Otherwise it offers the **opt-in Gemma download** (≈ 2 GB, one-time, cached in OPFS). Needs Chrome or Edge with WebGPU.

> The Gemma path streams weights once and persists them in OPFS, so only the first run pays the download cost.

---

## 8. Project layout

| File | Purpose |
|---|---|
| `index.html` | The whole app — UI, dual AI backend, judge engine, debate loop |
| `docs.html` | Renders this README as a styled docs page (linked from the app's **📖 Docs** button) |
| `README.md` | This document |
