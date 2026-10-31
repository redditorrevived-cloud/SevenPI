# PRD: AI Debate (Desktop)

Document status: authoritative. If another document conflicts with this one, this PRD wins on *what* to build; `TECH_STACK.md` wins on *how*.

## 1. Purpose and core objective

**AI Debate** is a Windows desktop app in which several AI models hold a conversation with each other. The user supplies their own API keys (several, from different providers), chooses how many AI agents take part (2 to 6), gives each agent a role, and picks one of two modes:

- **Debate:** agents argue a topic from assigned stances. An optional judge names a winner.
- **Co-op:** agents work as a team on a task, each from its own role. An optional "final answer" model merges the discussion into one result.

The interface is modelled on the Claude desktop app: left sidebar of past chats, centered conversation column, large rounded input box, warm neutral colors, light and dark themes.

**Core objective:** a user with at least one API key can start a multi-agent conversation in under 60 seconds and watch replies stream in live.

## 2. Target user

- **Primary:** a technically curious individual (student, developer, writer, researcher) who already has one or more AI provider API keys and wants to compare models, stress-test an idea, or get several perspectives.
- **Not targeted:** teams, enterprise admins, non-technical users who have no API key.

## 3. Glossary

| Term | Meaning |
|---|---|
| Key | A saved API credential: provider + label + model name + secret. |
| Agent | One participant in a conversation. Bound to exactly one Key, has a name and a role. |
| Role | Free text (max 120 chars) describing the agent's persona, e.g. "Skeptical economist". |
| Stance | Debate mode only: `for`, `against`, or `free` (own view). |
| Round | One full pass in which every agent speaks once, in agent order. |
| Turn | One agent message. |
| Wrap-up | Final message by the judge (Debate) or final-answer model (Co-op). |
| Follow-up | A message the user adds after the run; it triggers one extra round. |

## 4. Scope

### 4.1 In scope (MVP)

1. Windows 10/11 x64 desktop app (installer and portable build).
2. API key manager: add, list (masked), test, remove. Multiple keys. Providers: Anthropic, OpenAI, Google Gemini, OpenRouter.
3. Two modes: Debate and Co-op, selectable per conversation.
4. Agent count 2 to 6 chosen per conversation; each agent has name, key/model, role, and (Debate only) stance.
5. Rounds: 1 to 8.
6. Optional wrap-up model (judge or final answer).
7. Live token streaming of every reply.
8. Stop button (aborts the in-flight request and halts the run).
9. Retry after a failed turn (resumes at the failed turn, no repeated work).
10. Follow-up messages from the user, which run one extra round.
11. Conversation history in the sidebar: open, rename, delete.
12. Export a conversation as Markdown or plain text.
13. Settings: theme (system/light/dark), max words per turn.
14. Local persistence of everything (keys encrypted).
15. Native Windows installer built with electron-builder.

### 4.2 Out of scope (Post-MVP). Do NOT build these

- Accounts, login, cloud sync, telemetry, auto-update.
- Image, file, voice, or tool/function-calling support.
- Web search or any agent tool use.
- Per-agent temperature, system-prompt editor, or custom provider URLs.
- Token/cost tracking and billing estimates.
- Conversation branching, message editing, regenerate-single-turn.
- Mid-run user interjection (follow-ups are only allowed when the run is not running).
- Mac and Linux installers (code must stay cross-platform, but only Windows is built and tested).
- Localization (English only).
- Plugin system, themes beyond light/dark, custom fonts.
- Searching across conversations.

## 5. Functional requirements

Limits are hard validation rules enforced in both UI and main process.

| ID | Requirement |
|---|---|
| FR-1 | Topic/task text: required, 1 to 2000 characters after trimming. |
| FR-2 | Agent count: integer 2 to 6. Default 2. |
| FR-3 | Agent name: optional, max 40 chars. Blank means use the key's label. Duplicate names get a numeric suffix (`Claude`, `Claude 2`). |
| FR-4 | Agent role: optional, max 120 chars. Blank means "debater" (Debate) or "teammate" (Co-op). |
| FR-5 | Stance (Debate only): `for`, `against`, `free`. Default for agents 1,2,3 is `for`,`against`,`free`, repeating. |
| FR-6 | Rounds: integer 1 to 8. Default 3. |
| FR-7 | Wrap-up: none, or any saved key. Default none. |
| FR-8 | Words per turn: setting 50 to 400, default 150. Applied through the prompt, not enforced by truncation. |
| FR-9 | Agents speak in fixed order 1..N each round. |
| FR-10 | Each agent sees the full conversation transcript (all prior agent and user messages) on every turn. |
| FR-11 | A run can be started only if every agent's key still exists. Otherwise show a blocking error naming the agent. |
| FR-12 | Stop aborts the active HTTP request within 500 ms, marks the partial message `aborted`, sets conversation status `stopped`, and skips wrap-up. |
| FR-13 | On a turn failure the run halts, the failed message is kept with status `error`, and a Retry button appears. Retry deletes the failed message and resumes. |
| FR-14 | Follow-up: allowed when status is not `running`. Appends a `user` message, increments `rounds` by 1, clears any previous wrap-up, and starts a run. |
| FR-15 | If the app closes during a run, on next launch that conversation is set to `stopped`. |
| FR-16 | Keys are never shown in full after saving; only the last 4 characters. |
| FR-17 | "Test" on a key sends a 1-token request and reports success or the mapped error. |
| FR-18 | Deleting a key used by existing conversations is allowed; those conversations keep their history but cannot run until the agent is re-bound (MVP: show the error from FR-11). |
| FR-19 | Export writes a file chosen via the native save dialog. |
| FR-20 | Conversation title defaults to the first 60 chars of the topic; user can rename (1 to 80 chars). |

## 6. Key workflows

### 6.1 First launch
1. App opens to the Welcome view with an inline banner "Add an API key to begin" and an **Add API key** button.
2. User clicks it. API Keys modal opens.
3. User picks provider, types optional label, optional model (placeholder shows default), pastes key, clicks **Add key**.
4. Key appears masked in the list. User clicks **Test**; a success tick appears.
5. User closes the modal. The Welcome view now shows the setup composer.

### 6.2 Run a debate (main path)
1. User is on the Setup view with mode **Debate** selected.
2. User types a topic.
3. User sets Agents to 3 with the stepper. Three agent cards show.
4. For each card the user picks a key/model, types a role, and picks a stance.
5. User sets Rounds to 3 and Judge to a saved key.
6. User presses the send button (or Ctrl+Enter).
7. A conversation is created; the view switches to the Conversation view; the sidebar shows the new chat at the top.
8. Agent 1's message appears and streams. When complete, agent 2 starts, and so on.
9. After round 3, the judge's verdict appears in a highlighted card.
10. The follow-up input at the bottom becomes active.

### 6.3 Run a co-op session
Same as 6.2 but: Mode **Co-op**; stance selectors are hidden; the wrap-up selector is labelled "Final answer"; the closing card is titled "Final answer".

### 6.4 Stop and resume
1. During a run the Stop button shows in the header.
2. User clicks Stop. The current message is marked "Stopped" and the run ends.
3. The user may type a follow-up, which starts a new round. (Resuming the interrupted round is out of scope.)

### 6.5 Error and retry
1. A turn fails (for example 401).
2. The failed message shows an inline error with a human-readable reason; a **Retry** button appears.
3. User fixes the cause (for example replaces the key) and clicks Retry; the run resumes at the same agent and round.

### 6.6 Follow-up
1. After completion the user types a message in the bottom input and presses Enter.
2. Message appears right-aligned; one more round runs; the wrap-up (if set) is regenerated.

### 6.7 Export
1. User clicks **Export** in the header and picks Markdown or Text.
2. Native save dialog opens; file is written; a toast confirms.

## 7. Non-functional requirements

| Area | Standard |
|---|---|
| Startup | Window visible in under 3 s on a mid-range laptop (cold start, 100 saved chats). |
| Memory | Idle RSS under 300 MB total across Electron processes; under 500 MB during a 6-agent, 8-round run. |
| Streaming latency | Each received token chunk is rendered within 100 ms of arrival. UI stays at 60 fps (no layout thrash; batch updates with `requestAnimationFrame`). |
| History scale | Opening a conversation with 200 messages renders in under 500 ms. Sidebar with 500 chats scrolls smoothly (no virtualization needed up to 500). |
| Disk | Conversation files under 2 MB each in normal use; all writes atomic. |
| Offline | App launches, shows history, allows rename, delete, export, and settings with no network. Starting a run offline shows a clear "No internet connection" error (code `NETWORK`). Keys are never validated at add time, only on Test. |
| Platform | Windows 10 (build 1809+) and Windows 11, x64. No Windows-only code paths in app logic (use `path`, `app.getPath`). |
| Security | API secrets encrypted at rest with Electron `safeStorage`; secrets exist only in the main process; renderer is sandboxed with no Node access; strict CSP; all provider calls made from the main process. If `safeStorage.isEncryptionAvailable()` is false, refuse to save keys and show an error. |
| Privacy | No network calls except to the four provider hosts the user chose. No analytics. |
| Accessibility | All controls keyboard reachable; visible focus ring; text contrast at least WCAG AA; respects OS dark mode. |
| Reliability | A crash or force-quit must not corrupt stored data (atomic writes, tolerant JSON parse with `.bak` fallback). |

## 8. Definition of done (MVP acceptance)

1. All In-Scope items work as described, with the Windows installer produced by `npm run build:win`.
2. `npm run typecheck`, `npm run lint`, and `npm test` pass with zero errors.
3. Manual checklist in `CLINE_TASK_MASTER.md` Phase 12 passes in full.
4. No item from section 4.2 exists in the code.

## 9. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Provider model names change | Model is free text per key; default is only a placeholder. |
| Rate limits during multi-agent runs | Turns run strictly sequentially; 429 maps to `RATE_LIMITED` with Retry. |
| Transcript grows each turn | Accepted for MVP; documented in README. No truncation logic. |
| Agent (Cline) adds unrequested features | `CLINE_TASK_MASTER.md` rules forbid it; section 4.2 is the checklist. |
