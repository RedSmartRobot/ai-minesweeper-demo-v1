# One-Shot Prompt: Minesweeper AI Agent Demo

Use this prompt to recreate the full project in a single LLM run.

---

## Prompt

Build a complete **Minesweeper AI Agent demonstration** as a set of standalone HTML/JS files — no build step, no npm, no server required beyond static file hosting. The demo is intended for a live lecture on AI agents for non-specialist audiences (e.g. law students). Every design decision should serve that pedagogical purpose: things should be visible, legible, and explainable.

---

### Files to produce

1. `minesweeper_agent.html` — the main demo page
2. `connect.html` — the AI connection configuration page
3. `worker.js` — a Cloudflare Worker proxy script (ES module format)
4. `explainer.html` — a user-facing explainer page
5. `readme.md` — GitHub readme

---

### 1. `minesweeper_agent.html` — Main demo

**Visual design:** Dark terminal/retro aesthetic. Use fonts `VT323` (headings), `Share Tech Mono` (UI/code), `Rajdhani` (body) from Google Fonts. CSS variables for all colours. A scanline overlay (`repeating-linear-gradient`) on `body::before`. Colour palette: dark navy background (`#0a0e1a`), panel (`#111827`), border (`#1e3a5f`), accent cyan (`#00d4ff`), green (`#39ff14`), orange (`#ff9900`), red (`#ff2244`).

**Layout:** Three-column CSS grid: left panel (320px), centre (auto), right (1fr). Collapses to single column below 1000px. Include a slim status bar between header and main layout.

**Header:** Title "⬛ MINESWEEPER AI AGENT ⬛" in VT323. Subtitle row with links: "? WHAT IS THIS" → `explainer.html`, "⚙ CHANGE CONNECTION" in the status bar.

**Left column panels:**
- **Board State Sent to AI** — a `<div>` showing the ASCII board representation that was last sent to the AI. Green monospace text on near-black background. Updates every turn.
- **Agent Memory** — collapsible panel. Shows text from `localStorage` key `ms_agent_memory`. Has a CLEAR MEMORY button.
- **System Prompt** — collapsible panel. Contains an editable `<textarea>` pre-filled with the system prompt (see below). Caption: "Editable — changes take effect on next AI call."

**Centre column:** The Minesweeper board.
- 8×8 grid using chess notation: columns A–H (left to right), rows 1–8 (top to bottom). So squares are A1–H8.
- Rendered as a 9×9 CSS grid (9 columns × 9 rows, first row/col are labels). Each cell 36×36px.
- Cell states and styling:
  - `unrevealed` — dark blue-grey background, border, hover highlight
  - `revealed` — darker background, no border glow; number colour classes n1–n8 (distinct colours per digit)
  - `flagged` — orange border glow, 🚩 emoji
  - `wrong-flag` (game over, flagged but not a mine) — dark reddish background, ❌ emoji
  - `mine-hit` (the square that killed the agent) — red background, 💥 emoji
  - `mine-shown` (unrevealed mines shown on loss) — dark, 💣 emoji
  - AI action highlights: `ai-target` (cyan glow), `ai-guess` (yellow glow), `ai-flag` (orange glow)
- Status bar above board: mines left, flags placed, moves made.
- Buttons below board: NEW GAME, AI STEP, AUTO PLAY (toggles to STOP AUTO), MANUAL checkbox.
- A thinking indicator (animated dots) shown while waiting for AI response.
- Game-over banner at bottom of board panel: green "✓ FIELD CLEARED" on win, red "✗ MINE HIT" on loss.

**Right column:** AI Communication Log.
- Shows last 4 log entries. Each entry has a label (AGENT → AI / AI → AGENT / SYSTEM) and content.
- Entry types and styles: `outgoing` (green left border, monospace, small font), `incoming` (cyan left border), `system-msg` (orange left border), `error-msg` (red left border).
- Incoming entries show action badges inline: open actions in cyan, flag actions in orange, guess actions in yellow.
- "VIEW FULL CONVERSATION HISTORY" button opens a modal with the complete history.

**Minesweeper game logic:**
- Mines are placed on the first AI step (not at game start), always avoiding the first-clicked square and its 8 neighbours.
- `openCell(r, c)` — reveals a cell; if mine returns `'mine'`; if 0 neighbours, flood-fills recursively.
- `toggleFlag(r, c)` — flags/unflags unrevealed cells.
- On NEW GAME, automatically open cell B2 (row index 1, col index 1) as the first move — this is always safe, saves one AI call, and gives the AI a starting foothold.

**ASCII board representation** (sent to AI each turn):
```
   A B C D E F G H
  +----------------+
1 |? ? ? ? ? ? ? ?|
2 |? . 1 ? ? ? ? ?|
...
  +----------------+
```
Where: `?` = unrevealed unflagged, `F` = flagged, `.` = revealed 0, `1–8` = revealed number, `*` = mine (game over only).

**AI connection — read from localStorage:** All connection settings are stored in `localStorage` by `connect.html` and read by the demo on every call. Keys:
- `conn_tier` — `'proxy'` | `'user-groq'` | `'puter'`
- `conn_worker_url` — URL of Cloudflare Worker
- `conn_proxy_model` — model ID for proxy/user-groq tier
- `conn_puter_model` — model ID for puter tier
- `conn_user_groq` — user's own Groq API key

**`callAI(messages)` dispatcher:**
- `proxy` tier: POST to worker URL with body `{model, messages, max_tokens:2048}` and header `X-AI-Provider: groq|gemini`. Extract `choices[0].message.content` from response.
- `user-groq` tier: POST directly to `https://api.groq.com/openai/v1/chat/completions` with `Authorization: Bearer <key>`. Same response shape.
- `puter` tier: call `puter.ai.chat(messages, {model})`. Response parsing must handle multiple shapes: plain string, `response.message.content` as array (Claude), `response.message.content` as string (others).

**`executeActions(actions)` — invalid move filter:** Before executing each `OPEN` action, check `board[row][col].revealed` and `board[row][col].flagged`. Skip silently if either is true. Collect skipped squares and show a single system log entry after the loop: "⚠ Skipped invalid moves (already revealed): C3, D4".

**System prompt (DEFAULT_PROMPT):**
```
You are an AI agent playing Minesweeper on an 8×8 board.
NOTATION: Columns A–H (left to right), Rows 1–8 (top to bottom). Squares are written as A1–H8.

RULES:
- Opening a square reveals a mine (game over) or a number 0–8 counting adjacent mines.
- A revealed 0 means all 8 neighbours are safe and open automatically.
- Place flags on squares you believe contain mines.
- You win by opening every non-mine square.

BOARD STATE (sent to you each turn):
  ?  = unrevealed, unflagged
  F  = flagged
  .  = revealed with 0 mine-neighbours
  1–8 = revealed with that many mine-neighbours
  *  = mine (shown only on game over)

STRATEGY:
- If a revealed number equals the count of adjacent unrevealed squares, they are all mines — flag them.
- If a revealed number equals the count of adjacent flags, all remaining adjacent unrevealed squares are safe — open them.
- When pure logic is exhausted, guess probabilistically. Edges and corners are slightly safer as first moves.
- NEVER attempt to open a square that is already revealed. Only open squares marked with ?.
- Never open a flagged square.

RESPONSE FORMAT — ACTIONS MUST COME FIRST:
Start with "ACTIONS:" immediately, one action per line:
  OPEN A3
  FLAG B5
  OPEN C4 (guess)
Mark probabilistic moves with "(guess)". You may issue multiple actions.
After the actions, write a brief reasoning note (1–3 sentences) starting with "REASONING:".

MEMORY INSTRUCTIONS:
After a game ends (win or loss), write a memory entry on its own line in exactly this format:
  MEMORY: <2–4 sentences, first person: result, opening move assessment, patterns noticed, lesson learned>
If new experience contradicts old memory, update your conclusion rather than just appending.
Previous memory entries will be shown to you as:
  [Game 1]: <text>
  [Game 2]: <text>
```

**Agent memory (localStorage):**
- Key: `ms_agent_memory`. Value: plain text, entries formatted as `[Game N]: <text>`.
- After each game end, send a final message asking the AI to write its MEMORY entry. Parse with `/^MEMORY:\s*(.+)$/im`. Append to localStorage.
- On each AI call, append memory to system prompt if non-empty.

**Response parsing:**
- `parseActions(text)` — scan lines after `ACTIONS:`, stop at `REASONING:` or `MEMORY:`. Match `/^(OPEN|FLAG)\s+([A-H][1-8])(.*)?$/i`. Mark `isGuess` if `(guess)` in rest.
- `extractReasoning(text)` — extract text after `REASONING:` label. Fall back to text before `ACTIONS:` for models that ignore the format.
- `parseMemory(text)` — match `/^MEMORY:\s*(.+)$/im`.

**Game-over wrong flag display:** When `gameState !== 'playing'`, in `renderBoard()` check: if `cel.flagged && !cel.mine` → render as `wrong-flag` class with ❌. After `showGameOver()`, count wrong flags and add a system log entry.

**Conversation continuity:** Each game maintains `conversationMessages` array of `{role, content}` pairs. On every AI call, prepend `{role:'system', content: buildSystemPrompt()}` to the messages array (do not store the system message in `conversationMessages` itself — rebuild it fresh each time so prompt edits and memory updates take effect immediately).

---

### 2. `connect.html` — Connection configuration page

Three-tab interface (same dark aesthetic as main page):

**Tab 1 — My API Keys (Proxy):** Input field for Cloudflare Worker URL (pre-filled with a placeholder). Model grid showing proxy models.

**Tab 2 — Your Own Groq Key:** Password input for a personal Groq API key. Info text explaining where to get one free (console.groq.com), that it's stored only in localStorage. Model grid showing same proxy models.

**Tab 3 — Puter.js:** Status pill + Sign In/Out button. Puter authentication via `puter.auth.signIn()` / `puter.auth.signOut()`. Test connectivity with a minimal `puter.ai.chat` call. Model grid with Puter models.

**Proxy models (in order):**
1. `meta-llama/llama-4-scout-17b-16e-instruct` — Llama 4 Scout (Groq) — default
2. `llama-3.3-70b-versatile` — Llama 3.3 70B (Groq)
3. `gemini-2.5-flash` — Gemini 2.5 Flash (Google)
4. `gemini-2.5-pro` — Gemini 2.5 Pro (Google)

**Puter models:** Claude Sonnet 4.5, Claude Haiku 4.5, Claude Opus 4.5, GPT-4.1 Nano, Gemini 2.5 Flash (via Google), DeepSeek V3.2.

**Save & Return button** writes all settings to localStorage and redirects to `minesweeper_agent.html`.

---

### 3. `worker.js` — Cloudflare Worker (ES module)

- Handles CORS preflight (`OPTIONS` → 204).
- Reads `X-AI-Provider` header to route to either `https://api.groq.com/openai/v1/chat/completions` or `https://generativelanguage.googleapis.com/v1beta/openai/chat/completions`.
- API keys come from Worker environment variables `GROQ_API_KEY` and `GEMINI_API_KEY`.
- **Critical Gemini fix:** Gemini's OpenAI-compatible endpoint uses `max_completion_tokens`, not `max_tokens`. Parse the request body as JSON, swap the parameter for Gemini calls, then re-stringify. For Groq, keep `max_tokens`.
- Returns upstream response with CORS headers added.
- Optional origin check: validate `Origin` header against an allowlist.

---

### 4. `explainer.html` — User explainer

Light academic aesthetic (`EB Garamond` serif, cream background `#f7f4ef`). Three parts:

**Part 1 — What is an AI Agent?** (~200 words) Define the perception–action loop. Include a simple horizontal flow diagram: `ENVIRONMENT → PERCEPTION → AI REASONING → ACTION → ENVIRONMENT`.

**Part 2 — About This Demo** (~200 words + UI guide grid) Describe how the Minesweeper agent works. 2-column grid of UI element explanations: Board State, Memory, System Prompt, AI Log, AI Step/Auto, Change Connection, Cell Highlights, Manual Mode. Include a note box: "There is no persistent connection to the AI. Every call is stateless; the agent re-sends everything each time."

**Part 3 — Key Concepts Table** Eight rows: Perception–Action Loop, Environment, System Prompt, Stateless AI, Memory, Uncertainty & Guessing, Grounding, Multi-model Support.

---

### Key technical constraints and gotchas

1. **System prompt delivery to Puter.js:** Do NOT use a `systemPrompt` named option — it is silently ignored. Instead prepend `{role:'system', content:'...'}` as the first element of the messages array passed to `puter.ai.chat()`.

2. **Response shape varies by model via Puter:** Claude returns `response.message.content` as an array of `{type, text}` blocks. Most other models return it as a plain string. Handle both, plus a plain string response, plus a fallback to `JSON.stringify`.

3. **Gemini truncation:** Gemini's OpenAI-compatible endpoint ignores `max_tokens`. Must use `max_completion_tokens` instead. Handle this in the worker, not the demo.

4. **Mine placement timing:** Mines must be placed on the *first cell open*, not at game start. This guarantees the first click is always safe (avoid the clicked cell and its 8 neighbours when placing mines).

5. **Actions-first response format:** Put ACTIONS: before REASONING: in the prompt. This ensures moves are parseable even if the model truncates its response mid-reasoning.

6. **Puter sign-out loop:** After `puter.auth.signOut()`, do NOT immediately make an AI test call to check status — this triggers auto sign-in. Use `puter.auth.isSignedIn()` (a local check) instead.

7. **localStorage for settings:** Store connection settings in localStorage so `connect.html` and `minesweeper_agent.html` share state without any backend. Re-read on every AI call so prompt edits take effect immediately.

8. **Invalid move filter:** LLMs reliably attempt to re-open already-revealed squares despite prompt instructions. Always validate in `executeActions` before acting, not just in the prompt.
