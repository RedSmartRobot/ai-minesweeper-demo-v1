# Minesweeper AI Agent — Demo

An interactive demonstration of an **AI agent** playing Minesweeper, built as a teaching tool for a lecture on the technical aspects of AI for non-specialist audiences.

The demo shows the full perception–action loop of an agent in real time: the agent reads the board, sends it to a large language model, receives moves and reasoning in plain text, and executes them. Every step of the communication is visible on screen.

![Screenshot placeholder — replace with actual screenshot](screenshot.png)

---

## What it demonstrates

- The **perception–action loop** — the core concept of AI agency
- How an LLM receives structured input (ASCII board state) and produces structured output (move instructions)
- The role of the **system prompt** as the agent's "constitution" — editable live in the demo
- **Agent memory** — the AI writes natural-language notes after each game, which are fed back to it in subsequent games
- The difference between **stateless AI calls** and the appearance of continuity (the agent re-sends the full history each time)
- **Uncertainty** — the agent marks probabilistic moves explicitly as `(guess)`
- **Multi-model support** — the same agent code works with Llama, Gemini, Claude, GPT, and DeepSeek; switching models visibly changes reasoning style

---

## Files

| File | Purpose |
|---|---|
| `minesweeper_agent.html` | Main demo page — the game board and all panels |
| `connect.html` | Connection configuration — choose AI provider and model |
| `worker.js` | Cloudflare Worker proxy — keeps API keys off the client |
| `explainer.html` | User-facing explainer — three-part guide to AI agents |
| `setup_guide.html` | Step-by-step instructions for deploying the proxy |
| `one_shot_prompt.md` | Prompt for recreating this project from scratch with an LLM |

---

## How to run

All files are static HTML/JS. No build step, no npm, no server required.

1. Download all files into the same folder.
2. Open `connect.html` in a browser and configure your AI connection (see below).
3. Open `minesweeper_agent.html` to play.

For local testing, you may need to serve the files via a local HTTP server rather than opening directly from disk, because some browsers block `localStorage` access on `file://` URLs:

```bash
python3 -m http.server 8000
# then open http://localhost:8000/minesweeper_agent.html
```

---

## AI Connection Options

The demo supports three ways to connect to an AI, configured via `connect.html`:

### Option 1 — Cloudflare Worker proxy (recommended)

You deploy a small proxy (`worker.js`) to Cloudflare Workers (free tier: 100,000 requests/day). The proxy holds your API keys and routes requests to Groq or Google Gemini. The demo page never sees the keys.

**Supported models via proxy:**
- **Llama 4 Scout** (Groq) — recommended default; fast, free, 14,400 req/day renewable
- **Llama 3.3 70B** (Groq)
- **Gemini 2.5 Flash** (Google) — generous daily free tier
- **Gemini 2.5 Pro** (Google) — tighter free-tier limits

See `setup_guide.html` for step-by-step instructions on deploying the proxy and obtaining free API keys. Neither Groq nor Google require a credit card for their free tiers.

### Option 2 — Your own Groq key (direct)

Enter your personal Groq API key in `connect.html`. The key is stored only in your browser's `localStorage` and sent directly to Groq — no proxy needed. Suitable when you want to run the demo from your own account.

### Option 3 — Puter.js (fallback)

[Puter.js](https://puter.com) provides browser-side access to major AI models through a user's Puter account. No API key management required. Supports Claude, GPT, Gemini, DeepSeek, and others. Note: Puter's free-tier AI credits are non-renewable.

---

## Architecture

```
Browser (minesweeper_agent.html)
  │
  │  reads settings from localStorage (set by connect.html)
  │
  ├─── proxy tier ──→  Cloudflare Worker (worker.js)
  │                         ├──→  Groq API  (Llama models)
  │                         └──→  Gemini API (Gemini models)
  │
  ├─── user-groq tier ──→  Groq API directly
  │
  └─── puter tier ──→  Puter.js  ──→  Claude / GPT / Gemini / DeepSeek
```

**Key design decisions:**

- **No persistent connection.** Every AI call is a fresh HTTP request. The agent re-sends the full conversation history and the system prompt each time. The AI has no memory of its own — continuity is engineered by the agent.
- **System prompt is injected per call.** Editing the prompt in the UI takes effect on the very next step, mid-game.
- **Memory lives in `localStorage`.** The AI writes its own memory entries in natural language after each game. They persist across page reloads but are local to the browser.
- **Actions come before reasoning in the response.** The prompt instructs the AI to write `ACTIONS:` first, then `REASONING:`. This ensures moves are captured even if the model truncates its response.
- **Invalid move filter in code, not just in the prompt.** LLMs reliably attempt to re-open already-revealed squares despite prompt instructions. The agent validates every move before executing it.
- **Mines placed on first click.** The first cell opened is always safe (and its 8 neighbours). The demo opens B2 automatically on game start to give the AI a starting foothold and save one API call.

---

## Prompt Engineering Notes

The system prompt is visible and editable in the demo UI, which is itself a teaching point. Key decisions in the prompt design:

- **Actions-first format** avoids wasted API calls when responses are truncated by token limits (notably Gemini on the free tier).
- **Explicit board legend** (`? F . 1–8 *`) is necessary — without it, models guess the notation and fail.
- **Negative instruction for revealed squares** (`NEVER attempt to open a square that is already revealed`) reduces but does not eliminate the problem; the code filter is the real fix.
- **Memory instruction with exact format** (`MEMORY: <text>`) makes the memory entry machine-parseable by regex.
- The prompt is short enough to stay well within free-tier token budgets.

---

## Gotchas Encountered During Development

These are documented here because they are not obvious and cost significant debugging time:

1. **Puter.js `systemPrompt` option is silently ignored.** Pass the system message as `{role:'system', content:'...'}` as the first element of the messages array instead.

2. **Puter.js response shape varies by model.** Claude returns `response.message.content` as an array of `{type, text}` blocks; most other models return it as a plain string. The `extractText()` function handles all cases.

3. **Gemini ignores `max_tokens`.** The Gemini OpenAI-compatible endpoint requires `max_completion_tokens` instead. This must be handled in the proxy worker by swapping the parameter before forwarding.

4. **Puter sign-out triggers immediate re-authentication** if you make an AI test call right after. Use `puter.auth.isSignedIn()` (a local check) to update the UI instead.

5. **Cloudflare Worker 405 errors** from the demo usually mean the Worker URL is missing the `https://` prefix, or the worker code was not deployed (saved but not deployed is a common mistake in the Cloudflare dashboard).

---

## Extending the Demo

Some natural next steps not implemented here:

- **Origin check in the worker** — restrict the proxy to requests from your specific domain to prevent quota abuse by others who know the worker URL.
- **State compression** — send a denser board representation to the AI to reduce token usage (meaningful mainly for very long games or models with tight context limits).
- **Different environments** — the agent architecture is environment-agnostic; replacing the Minesweeper board with a contract review task, a negotiation simulator, or a text adventure would illustrate the same concepts in a law-relevant context.
- **Two-agent mode** — run two AI models simultaneously and compare their moves turn by turn.

---

## Credits

Built iteratively with Claude (Anthropic) as a teaching demonstration for a lecture on AI agents. The project evolved through approximately 40 exchanges covering architecture, debugging, UI design, prompt engineering, and API integration.

---

## License

MIT — free to use, adapt, and redistribute with attribution.
