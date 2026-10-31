# CLINE_TASK_MASTER: Implementation Roadmap

Build **AI Debate** exactly as specified in `PRD.md`, `TECH_STACK.md`, `DATA_MODELS.md`, and `UI_UX_SPEC.md`. This file is the order of work.

## 0. Rules for the coding agent (read before every task)

1. **One task at a time, in order.** Do not start task N+1 until task N's **Verify** step passes.
2. **Read first.** Before each task, re-read the sections of the spec documents named in the task.
3. **No scope creep.** Do not build anything in `PRD.md` section 4.2. Do not add packages outside `TECH_STACK.md` section 3. Do not create files outside the tree in `TECH_STACK.md` section 7 (except `*.test.ts(x)`).
4. **Stay inside task size.** Each task should touch at most 5 files. If it needs more, stop and split it.
5. **Types first.** Never use `any`. Copy shared types from `DATA_MODELS.md` verbatim.
6. **Fix before moving on.** If Verify fails, fix the cause. Do not skip, comment out, or weaken a check, test, or lint rule.
7. **Log progress.** After each task passes, append one line to `PROGRESS.md`: `- [x] <task id> <task title>`.
8. **Commit.** After each task passes, run `git add -A && git commit -m "<task id>: <title>"`.
9. **If blocked or unsure,** stop and ask the user. Do not guess an API shape; the provider formats are in `TECH_STACK.md` section 5.
10. **Never** print, log, or commit API keys. `.env` files are not used.

Common verification commands: `npm run typecheck`, `npm run lint`, `npm test`, `npm run build`, `npm run dev`.

---

## Phase 0: Prerequisites (human does these)

- Install Node.js 22 LTS and Git. Run `node -v` (must print v22.x) and `git --version`.
- Create an empty folder `ai-debate`, copy the 5 spec files into it, open it in your editor with Cline.
- Tell Cline: "Follow CLINE_TASK_MASTER.md starting at task 1.1."

---

## Phase 1: Setup and scaffolding

### 1.1 Scaffold the project
- **Do:** In the project root run `npm create @quick-start/electron@latest . -- --template react-ts`. If prompted: TypeScript yes, ESLint yes, Prettier yes, Electron updater no, Windows/Mac/Linux builds: skip. If the tool refuses a non-empty folder, scaffold into `../ai-debate-tmp` and move everything except the 5 spec files into the root. Set `"name": "ai-debate"` and `"productName"` in `package.json`.
- **Verify:** `npm install` finishes with no errors, then `npm run dev` opens a window with the template UI. Close it.

### 1.2 Pin versions and engines
- **Do:** Edit `package.json` so versions match `TECH_STACK.md` section 2. Add `"engines": {"node": ">=22 <23"}`. Run `npm install`.
- **Verify:** `npm ls electron react typescript electron-vite` shows the specified majors. `npm run dev` still opens.

### 1.3 Install runtime dependencies
- **Do:** `npm install zustand zod lucide-react react-markdown remark-gfm @radix-ui/react-dialog @radix-ui/react-select @radix-ui/react-tooltip @radix-ui/react-dropdown-menu clsx tailwind-merge`.
- **Verify:** `npm ls zustand zod lucide-react` prints each without `UNMET` or `missing`.

### 1.4 Install dev dependencies
- **Do:** `npm install -D tailwindcss@3 postcss autoprefixer vitest @testing-library/react @testing-library/jest-dom jsdom prettier-plugin-tailwindcss @types/node@22`. Do not install Tailwind 4.
- **Verify:** `npx tailwindcss --help` runs; `npx vitest --version` prints 2.x.

### 1.5 Configure TypeScript strictness and aliases
- **Do:** Edit `tsconfig.node.json` and `tsconfig.web.json` per `TECH_STACK.md` 8.1 and 7.1. Add `src/shared/**/*` to both `include` lists. Add aliases `@shared/*`, `@main/*`, `@renderer/*`. Mirror aliases in `electron.vite.config.ts` `resolve.alias` for main, preload, and renderer.
- **Verify:** `npm run typecheck` passes with zero errors.

### 1.6 Configure Prettier, ESLint, and editorconfig
- **Do:** Create/replace `.prettierrc.json` (8.2), `eslint.config.mjs` (8.3), `.editorconfig` (2 spaces, LF, UTF-8, final newline). Update npm scripts to match `TECH_STACK.md` 8.5.
- **Verify:** `npm run lint` passes; `npm run format` runs; `git diff --stat` shows only formatting changes.

### 1.7 Configure Tailwind and global CSS
- **Do:** Create `tailwind.config.ts` (content globs for `src/renderer/**/*.{ts,tsx,html}`, `darkMode: 'class'`, colors mapped to CSS variables), `postcss.config.js`, and `src/renderer/src/styles/globals.css` with the full token set from `UI_UX_SPEC.md` 8.1 for `:root` and `.dark`, plus base styles (8.2, 8.5). Import the CSS in `main.tsx`. Remove template CSS files and assets.
- **Verify:** `npm run dev` shows the page with background `#faf9f5`. Add `class="dark"` to `<html>` in devtools and confirm the background changes to `#262624`.

### 1.8 Create the folder skeleton
- **Do:** Create every directory in `TECH_STACK.md` section 7 (use empty `.gitkeep` files in directories that stay empty for now). Create `PROGRESS.md` and `.gitignore` (node_modules, dist, out, `*.log`).
- **Verify:** `git status` shows the new tree; `npm run typecheck` still passes.

### 1.9 Secure the window and add CSP
- **Do:** Implement `src/main/window.ts` with the exact `BrowserWindow` options and handlers from `TECH_STACK.md` 4.1. Call it from `src/main/index.ts` with the single-instance lock. Add the CSP meta tag to `src/renderer/index.html`.
- **Verify:** `npm run dev` opens a 1200x800 window; in devtools console run `typeof require` and `typeof process`; both print `undefined`. Opening a second instance focuses the first.

---

## Phase 2: Core data engine (shared + storage)

### 2.1 Shared constants and types
- **Do:** Create `src/shared/constants.ts` (section 2 of `DATA_MODELS.md` plus `ERROR_MESSAGES` mapping each `ErrorCode` to the user text in `TECH_STACK.md` 5.3) and `src/shared/types.ts` (section 3, verbatim).
- **Verify:** `npm run typecheck` passes.

### 2.2 Zod schemas
- **Do:** Create `src/shared/schemas.ts` per `DATA_MODELS.md` section 4, including the two-way compile-time drift checks.
- **Verify:** `npm run typecheck` passes. Temporarily remove one field from a schema and confirm typecheck fails, then restore it.

### 2.3 Pure text helpers with tests
- **Do:** Create `src/shared/text.ts` exporting `makeTitle(topic)`, `dedupeNames(names)`, `buildTranscript(messages)` (rules in `DATA_MODELS.md` 5.1 and `TECH_STACK.md` 6.2), `slugify(title)`. Create `vitest.config.ts` (node environment, include `src/**/*.test.ts`). Write `src/shared/text.test.ts` with at least 8 cases (empty, duplicates, long titles, only complete messages included, user label).
- **Verify:** `npm test` passes all tests.

### 2.4 Atomic JSON file helpers
- **Do:** Create `src/main/storage/atomic.ts` with `readJson(path, schema)`, `writeJsonAtomic(path, data)`, and the per-path write queue exactly as in `DATA_MODELS.md` 1.2. Create `src/main/storage/paths.ts` returning all file paths from a `root` string (injectable for tests).
- **Verify:** Write `atomic.test.ts` using `os.tmpdir()`: write then read; corrupt main file and confirm `.bak` fallback; missing file returns `null`; two parallel writes end with valid JSON. `npm test` passes.

### 2.5 Settings store
- **Do:** Create `src/main/storage/settingsStore.ts` with `getSettings()` (defaults if file missing) and `setSettings(s)` validating with `SettingsSchema`.
- **Verify:** `settingsStore.test.ts` covers default load, save, reload, and rejection of `wordsPerTurn: 10`. `npm test` passes.

### 2.6 Key vault
- **Do:** Create `src/main/storage/keyVault.ts` with `listKeys()` (returns `ApiKeyInfo[]` without `secretEnc`), `addKey(input)`, `removeKey(id)`, `getSecret(id)`, using Electron `safeStorage`. Accept an injectable `crypto` object (`encrypt`, `decrypt`, `isAvailable`) so tests use a fake. Apply empty label/model defaults from `PROVIDERS`. Store `last4`.
- **Verify:** `keyVault.test.ts` with a fake crypto: add, list (no secret in output; JSON.stringify of list must not contain the secret), getSecret roundtrip, remove, and failure when `isAvailable` is false. `npm test` passes.

### 2.7 Conversation store: create, read, list
- **Do:** Create `src/main/storage/conversationStore.ts` with `create(input, keys, settings)` (rules in `DATA_MODELS.md` 5.1), `get(id)`, `list()`, and index maintenance (5.2).
- **Verify:** `conversationStore.test.ts`: create returns deduped names, `coop` forces `stance: 'free'`, title is cut at 60 chars, list is sorted by `updatedAt` desc, missing key id returns `VALIDATION` error. `npm test` passes.

### 2.8 Conversation store: mutate, rename, delete, repair
- **Do:** Add `appendMessage`, `updateMessage`, `setStatus`, `rename`, `remove`, `deleteTrailingFailed`, `rebuildIndex`, and `repairOnStartup` (rules 5.3 to 5.7).
- **Verify:** Extend the test file: rename validation; delete removes file and index entry; startup repair converts `running`→`stopped` and `streaming`→`aborted`; index rebuild from files works. `npm test` passes.

---

## Phase 3: Provider adapters

### 3.1 SSE reader
- **Do:** Create `src/main/providers/sse.ts` per `TECH_STACK.md` 5.2.
- **Verify:** `sse.test.ts` feeds a mocked `Response` whose body is split mid-line across chunks, includes comment lines (`:`), a multi-line `data:` event, and CRLF line endings. `npm test` passes.

### 3.2 Error mapping and shared types
- **Do:** Create `src/main/providers/types.ts` (interfaces from 5) and `errors.ts` with `mapHttpError(status, bodyText)` and `mapThrown(err)` per the table in 5.3, plus an inactivity-timeout helper `withInactivityTimeout(signal, ms)`.
- **Verify:** `errors.test.ts` covers statuses 401, 403, 404, 429, 500, a `TypeError`, and an `AbortError`. `npm test` passes.

### 3.3 Anthropic adapter
- **Do:** Create `src/main/providers/anthropic.ts` implementing `ProviderAdapter` using `fetch`, request format and stream parsing from 5.1.
- **Verify:** `anthropic.test.ts` mocks `globalThis.fetch` with a fake SSE stream (`content_block_delta` events); asserts headers (`x-api-key`, `anthropic-version`), body fields, `onDelta` call sequence, final returned text. Add a 401 case. `npm test` passes.

### 3.4 OpenAI and OpenRouter adapters
- **Do:** Create `openai.ts` and `openrouter.ts`. Share one helper `streamChatCompletions(url, ...)` in `openai.ts` and import it into `openrouter.ts`.
- **Verify:** Tests with a mocked stream including the `[DONE]` sentinel and an empty `delta`; assert the URL for each provider and the Bearer header. `npm test` passes.

### 3.5 Gemini adapter
- **Do:** Create `src/main/providers/gemini.ts` per 5.1 (`x-goog-api-key` header, never put the key in the URL).
- **Verify:** Test with a mocked SSE stream containing multiple `parts`; asserts URL contains `:streamGenerateContent?alt=sse` and does not contain the key. `npm test` passes.

### 3.6 Adapter registry and `test()` methods
- **Do:** Create `src/main/providers/index.ts` with `getAdapter(id)`. Implement each adapter's `test()` as a call with `maxTokens: 1`, user text `Hi`, discarding output.
- **Verify:** `index.test.ts` asserts all four ids resolve and an unknown id throws. `npm run typecheck` and `npm test` pass.

---

## Phase 4: Orchestrator

### 4.1 Prompt builders
- **Do:** Create `src/main/orchestrator/prompts.ts` with pure functions `buildSystem(conv, agent, wordsPerTurn)`, `buildTurnUser(conv, agent, transcript, round, isFirstTurn, lastIsUser)`, `buildWrapUp(conv, transcript)`. Copy the templates verbatim from `TECH_STACK.md` 6.3.
- **Verify:** `prompts.test.ts` snapshot-style tests with literal expected strings for: debate opening, debate rebuttal, debate closing, coop first, coop converge, user-reply phase, judge, final answer, blank role defaults. `npm test` passes.

### 4.2 Progress derivation
- **Do:** Create `src/main/orchestrator/progress.ts` with `nextStep(conv)` returning `{kind:'turn', agentIndex, round} | {kind:'wrapup'} | {kind:'done'}` per 6.1.
- **Verify:** `progress.test.ts`: fresh conversation → agent 0 round 1; after 3 of 3 agents → agent 0 round 2; after all rounds with wrap-up → `wrapup`; with no wrap-up → `done`; wrap-up already complete → `done`; failed trailing message is ignored. `npm test` passes.

### 4.3 Delta batcher
- **Do:** Create `src/main/orchestrator/batcher.ts`: `createBatcher(flush: (text: string) => void, intervalMs = 50)` returning `{push(text), flushNow()}`.
- **Verify:** `batcher.test.ts` using `vi.useFakeTimers()`: 10 pushes within 50 ms produce one flush with concatenated text; `flushNow` flushes immediately. `npm test` passes.

### 4.4 Runner core: one turn
- **Do:** Create `src/main/orchestrator/runner.ts` with `runTurn(deps, conv, step, signal)` that builds prompts, creates the streaming message, calls the adapter, updates the message through `deps.store`, and emits events through `deps.emit`. `deps` is an injectable object (store, getAdapter, getSecret, emit, now, id). Do not import Electron here.
- **Verify:** `runner.test.ts` with fake deps: a successful turn emits `turn-started`, ≥1 `turn-delta`, `turn-completed` and stores `complete`; a thrown `INVALID_KEY` produces `turn-failed` and stores `error`. `npm test` passes.

### 4.5 Runner core: full run, stop, retry, follow-up
- **Do:** Add `startRun(deps, id)`, `stopRun(id)`, `retry(deps, id)`, `followUp(deps, id, text)` implementing `TECH_STACK.md` 6.4, the single-active-run rule, persistence debounce (2 s), startup repair hook, and wrap-up as a final turn of kind `wrapup`. Keep an in-memory `Map<string, AbortController>`.
- **Verify:** Tests: 2 agents × 2 rounds yields 4 agent messages in order then wrap-up; stop mid-stream yields `aborted` + status `stopped` and no wrap-up; failure then retry resumes at the same agent without duplicates; follow-up adds a user message and exactly `n` new agent messages and regenerates wrap-up; second concurrent `startRun` is rejected. `npm test` passes.

---

## Phase 5: IPC and preload

### 5.1 IPC definitions
- **Do:** Create `src/shared/ipc.ts` with `CH`, `ApiSurface`, and the Zod input schemas for each channel (`DATA_MODELS.md` 6). Export `ok(data)` and `fail(code, message)` helpers.
- **Verify:** `npm run typecheck` passes.

### 5.2 Handler registration helper
- **Do:** Create `src/main/ipc/register.ts` with `handle(channel, schema, fn)` that validates input, catches errors, converts them to `Result`, and calls `ipcMain.handle`. Export `registerAll(deps)`.
- **Verify:** `register.test.ts` mocks `ipcMain`: invalid input returns `VALIDATION`; a thrown error becomes `UNKNOWN` with no stack in the message; a thrown `AppError` keeps its code. `npm test` passes.

### 5.3 Keys and settings handlers
- **Do:** Create `src/main/ipc/keys.ts` (list, add, remove, test) and `src/main/ipc/settings.ts` (get, set). `keys:test` uses `getAdapter(...).test` with a 20 s timeout.
- **Verify:** `npm run typecheck`; unit test `keys:add` handler with fakes. `npm test` passes.

### 5.4 Conversation and run handlers
- **Do:** Create `src/main/ipc/conversations.ts` and `src/main/ipc/run.ts`. Run events are forwarded with `win.webContents.send(CH.runEvent, event)` only if the window is not destroyed. `conv:delete` stops an active run first.
- **Verify:** `npm run typecheck` and `npm test` pass.

### 5.5 Export handler
- **Do:** Create `src/main/ipc/exporter.ts` and a pure `renderExport(conv, format)` function per `DATA_MODELS.md` section 7, using `dialog.showSaveDialog`.
- **Verify:** `exporter.test.ts` tests `renderExport` for markdown and text, including that error and aborted messages are omitted and the wrap-up heading is correct per mode. `npm test` passes.

### 5.6 Wire main and preload
- **Do:** In `src/main/index.ts` call `repairOnStartup`, `registerAll`. Create `src/preload/index.ts` building `api: ApiSurface` with `ipcRenderer.invoke` and `onEvent` (subscribe to `CH.runEvent`, return an unsubscribe). Create `src/preload/index.d.ts` declaring `window.api`.
- **Verify:** `npm run dev`; in the renderer devtools console run `await window.api.settings.get()` and confirm it returns `{ok:true,data:{theme:'system',wordsPerTurn:150}}`. `npm run typecheck` passes.

---

## Phase 6: Renderer foundation

### 6.1 Renderer utilities
- **Do:** Create `lib/cn.ts`, `lib/api.ts` (`call(promise)` unwraps `Result` or throws an `AppErrorException`), `lib/format.ts` (`initials`, `relativeTime`). Add tests for `format.ts`.
- **Verify:** `npm test` and `npm run typecheck` pass.

### 6.2 Stores: settings, keys, ui, run
- **Do:** Create `settingsStore.ts`, `keysStore.ts`, `uiStore.ts` (including `SetupDraft` defaults and toasts), `runStore.ts` per `DATA_MODELS.md` 8.
- **Verify:** Store tests (jsdom environment via `// @vitest-environment jsdom`) with a mocked `window.api`: keys load/add/remove; toast push/dismiss; draft defaults (2 agents, 3 rounds). `npm test` passes.

### 6.3 Conversations store and event handling
- **Do:** Create `conversationsStore.ts` with all actions and `applyEvent` rules from `DATA_MODELS.md` 8.2, including the `requestAnimationFrame` batch for deltas.
- **Verify:** Tests simulate the event sequence `run-started`, `turn-started`, three `turn-delta`, `turn-completed`, `run-finished` and assert final message text, status, and `runningId`. Events for another conversation id are ignored except `run-finished`. `npm test` passes.

### 6.4 Hooks
- **Do:** Create `useRunEvents.ts` (subscribe once, forward to `applyEvent`), `useTheme.ts` (class toggle + matchMedia), `useAutoScroll.ts` (80 px rule from `UI_UX_SPEC.md` 2.3), `useShortcuts.ts` (section 5 shortcuts).
- **Verify:** `useAutoScroll` unit test: stays pinned when near bottom, does not scroll when the user scrolled up. `npm test` and `npm run typecheck` pass.

---

## Phase 7: UI primitives and layout

### 7.1 Button, IconButton, Spinner, Badge
- **Do:** Create these in `components/ui/` using the tokens and sizes from `UI_UX_SPEC.md` 3 and 8.
- **Verify:** `npm run dev` with a temporary demo in `App.tsx` showing all variants in light and dark. Remove the demo afterwards. Lint and typecheck pass.

### 7.2 Input, Textarea, Select, SegmentedControl, Stepper
- **Do:** Create these. `Select` wraps Radix Select; `Stepper` clamps to `min`/`max`.
- **Verify:** `Stepper.test.tsx`: clicking + at max does nothing; − at min does nothing. `npm test` passes.

### 7.3 Dialog and Tooltip
- **Do:** Create `ui/Dialog.tsx` and `ui/Tooltip.tsx` with the overlay, sizes, and animation from the spec (including `prefers-reduced-motion`).
- **Verify:** `Dialog.test.tsx`: opens, Esc calls `onOpenChange(false)`. `npm test` passes.

### 7.4 Toasts
- **Do:** Create `ui/Toast.tsx` with `ToastHost` reading `uiStore.toasts`; auto-dismiss 4 s / 7 s for errors.
- **Verify:** Test with fake timers: a success toast disappears after 4 s, an error after 7 s. `npm test` passes.

### 7.5 AppShell and Sidebar
- **Do:** Create `layout/AppShell.tsx`, `Sidebar.tsx`, `SidebarItem.tsx` with New chat, Recents (all four list states), bottom API keys and Settings buttons, hover menu (Rename, Delete), running dot, collapse behaviour from `UI_UX_SPEC.md` section 6.
- **Verify:** `npm run dev`: sidebar shows "No chats yet"; `Ctrl+B` toggles it; resizing below 1100 px collapses it. Typecheck and lint pass.

### 7.6 Header
- **Do:** Create `layout/Header.tsx` with title, mode badge, Stop (visible only when `runningId === activeId`), Export dropdown, and the sidebar toggle.
- **Verify:** `Header.test.tsx` with stores seeded: Stop hidden when idle and visible when running; Export disabled with no complete messages. `npm test` passes.

### 7.7 Rename and Delete dialogs
- **Do:** Create `components/layout/RenameDialog.tsx` and `DeleteDialog.tsx` wired to `conversationsStore`.
- **Verify:** Test: rename rejects an empty title (inline error) and accepts 80 chars; delete calls `remove`. `npm test` passes.

---

## Phase 8: Keys, settings, and setup view

### 8.1 Keys dialog
- **Do:** Create `keys/KeysDialog.tsx`, `KeyRow.tsx`, `AddKeyForm.tsx` with every state in `UI_UX_SPEC.md` 4.5 (Test spinner → check or error, reverts after 5 s). Open it from the sidebar button and `Ctrl+K`.
- **Verify:** `AddKeyForm.test.tsx` (mock `window.api`): empty secret blocks submit; success clears the form; failure shows the inline error. `npm test` passes. Manual: add a dummy key and see it masked.

### 8.2 Settings dialog and theme
- **Do:** Create `settings/SettingsDialog.tsx` (theme segmented control, words per turn input with clamp) and connect `useTheme`.
- **Verify:** `npm run dev`: switching Light/Dark/System changes the UI immediately and persists after restarting the app (`npm run dev` again).

### 8.3 Welcome view
- **Do:** Create `views/WelcomeView.tsx` with loading, empty, and error states (`UI_UX_SPEC.md` 4.1) and route to it when `keys.length === 0`.
- **Verify:** Fresh data folder shows the Welcome view; adding a key switches to the Setup placeholder; success toast shows.

### 8.4 Setup composer and toolbar
- **Do:** Create `setup/SetupView.tsx` (hero, textarea with counter, toolbar), `ModeToggle.tsx`, and wire `uiStore.setupDraft`. Hero text changes by mode.
- **Verify:** `SetupView.test.tsx`: Send disabled when topic is blank; switching mode changes heading and wrap-up label ("Judge" vs "Final answer"); counter appears at 1800 characters. `npm test` passes.

### 8.5 Agent cards
- **Do:** Create `setup/AgentCard.tsx` and the agents grid. Adding/removing agents via the Stepper adds/removes cards (2 to 6); default stances rotate for/against/free; stance select hidden in Co-op; removed-key invalid state.
- **Verify:** Tests: stepper to 4 renders 4 cards; Co-op hides stance selects; agent 3 default stance is `free`. `npm test` passes.

### 8.6 Create conversation from Setup
- **Do:** Implement the Send action: validate, call `conversationsStore.create`, then `window.api.run.start(id)`, switch to the Conversation route, show the loading and error states from `UI_UX_SPEC.md` 4.2, and support `Ctrl+Enter`.
- **Verify:** With a dummy key, pressing Send creates a chat in the sidebar (the run fails with an `INVALID_KEY` error; that is expected here). Typecheck and lint pass.

---

## Phase 9: Conversation view

### 9.1 Message bubble with Markdown
- **Do:** Create `conversation/MessageBubble.tsx` (avatar, name, meta line, `react-markdown` + `remark-gfm`, caret while streaming, copy button, Stopped tag) and `UserMessage.tsx`, styled per `UI_UX_SPEC.md` 2.3 and 8.4.
- **Verify:** `MessageBubble.test.tsx`: meta line shows stance only in Debate; caret only when `status === 'streaming'`; the Markdown `**bold**` renders as `<strong>`. `npm test` passes.

### 9.2 Wrap-up card and error banner
- **Do:** Create `WrapUpCard.tsx` ("Judge" vs "Final answer") and `ErrorBanner.tsx` with Retry.
- **Verify:** Tests for title by mode and that Retry calls `onRetry`. `npm test` passes.

### 9.3 Conversation view assembly
- **Do:** Create `conversation/ConversationView.tsx` and `views/ConversationRoute.tsx`: loading skeleton, message list, thinking line, jump-to-latest pill, auto-scroll via `useAutoScroll`.
- **Verify:** `npm run dev`: opening a chat from the sidebar shows skeletons briefly then messages. Typecheck passes.

### 9.4 Follow-up composer and Stop/Retry wiring
- **Do:** Create `conversation/FollowUpComposer.tsx` (Enter sends, Shift+Enter newline, disabled while running). Wire Stop (`run.stop`), Retry (`run.retry`), and follow-up (`run.followUp`). Hide the dock while running.
- **Verify:** `FollowUpComposer.test.tsx`: Enter calls `onSend` with trimmed text and clears the field; Shift+Enter does not send; blank text does not send. `npm test` passes.

### 9.5 App routing and startup sequence
- **Do:** In `App.tsx` implement the startup sequence from `DATA_MODELS.md` 8.3 and choose Welcome / Setup / Conversation views. Mount `ToastHost` and `useRunEvents`.
- **Verify:** `npm run dev`: quitting and restarting preserves chats and settings; the app opens on the Setup view when keys exist.

### 9.6 Export UI
- **Do:** Wire the Export dropdown to `window.api.exportConversation` with the loading, error, and success states in `UI_UX_SPEC.md` 4.7.
- **Verify:** Manual: export a conversation as `.md` and `.txt` and open both files; headings match `DATA_MODELS.md` section 7.

---

## Phase 10: Integration with real providers

### 10.1 Live test, one provider
- **Do (human + agent):** The user adds one real key in the app and presses **Test**. If it fails, fix the adapter and add a regression unit test.
- **Verify:** Test shows "Works". A 2-agent, 1-round debate using that same key for both agents completes with streamed text.

### 10.2 Live test, mixed providers (human)
- **Do:** The user adds a second key from another provider and runs a 3-agent Co-op session with a final answer.
- **Verify:** Messages arrive in agent order 1,2,3; the Final answer card appears; the export contains all messages.

### 10.3 Failure behaviour
- **Do:** Test by hand: wrong key (expect `INVALID_KEY` banner + Retry), wrong model name (`MODEL_NOT_FOUND`), turning off Wi-Fi mid-run (`NETWORK`). Fix any state bug found and add a unit test for it.
- **Verify:** After fixing the cause and pressing Retry, the run resumes at the failed agent without duplicate messages. Press Stop mid-stream: the message shows "Stopped" within 1 second.

### 10.4 Follow-up and restart safety
- **Do:** Run a debate, add a follow-up, then kill the app with the task manager during a second run and relaunch.
- **Verify:** The interrupted chat shows status Stopped, its partial message is tagged Stopped, and no data file is corrupted (open the chat; it loads).

---

## Phase 11: Polish

### 11.1 Accessibility pass
- **Do:** Ensure every `IconButton` has an `aria-label`, dialogs have titles, focus returns to triggers, tab order is logical, `prefers-reduced-motion` disables transitions.
- **Verify:** Keyboard-only run-through: create a debate and export it without using the mouse. Lint passes.

### 11.2 Resize and theme pass
- **Do:** Check `UI_UX_SPEC.md` section 6 at 900x600, 1100x700, and 1600x900, in light and dark. Fix overflow issues.
- **Verify:** No horizontal scrollbar in the main area at any size; toolbar wraps correctly at narrow widths.

### 11.3 Performance pass
- **Do:** Generate a conversation with 200 messages using a dev-only script placed in a test (not in app code). Confirm streaming updates do not re-render the whole list (memoize `MessageBubble` with `React.memo` keyed by message id and text).
- **Verify:** Opening the 200-message chat takes under 500 ms (measure with `performance.now()` in a test or devtools).

### 11.4 README for users
- **Do:** Replace the template `README.md` with: what the app is, how to install from the NSIS installer, how to add a key, the two modes, known limits (transcript grows each turn, no cost tracking), and how to build from source.
- **Verify:** A new reader can follow the README steps without any missing command.

---

## Phase 12: Testing and build

### 12.1 Full automated check
- **Do:** Run all checks and fix any failure.
- **Verify:** `npm run typecheck && npm run lint && npm test` all exit with code 0.

### 12.2 Icon and builder configuration
- **Do:** Add `resources/icon.ico` (256x256; ask the user to supply one, or generate a simple placeholder with a script outside the repo) and create `electron-builder.yml` per `TECH_STACK.md` 8.6. Add the `build:win` script if missing.
- **Verify:** `npm run build` completes with zero errors.

### 12.3 Build the Windows installer
- **Do:** Run `npm run build:win`.
- **Verify:** `dist/` contains `AI-Debate-<version>-nsis.exe` and `AI-Debate-<version>-portable.exe`.

### 12.4 Manual acceptance checklist (human)
Install the NSIS build on a clean Windows user and confirm each item:
1. App starts in under 3 seconds and shows Welcome.
2. Add two keys from different providers; both Test OK.
3. Debate with 3 agents, 3 rounds, judge set: streams live, verdict appears.
4. Co-op with 4 agents and a final answer: stances hidden, "Final answer" card shown.
5. Stop mid-run works; Retry works after a forced error.
6. Follow-up message adds exactly one more round.
7. Rename, delete, and export (md and txt) work.
8. Close and reopen: history, theme, and keys persist; keys are masked.
9. Offline: history opens; starting a run shows the no-connection error.
10. Look in `%APPDATA%\ai-debate\ai-debate-data\keys.json`: no plain-text key appears.
11. Nothing from `PRD.md` section 4.2 exists in the app.

### 12.5 Tag the release
- **Do:** Update `package.json` version to `1.0.0`, commit, and run `git tag v1.0.0`.
- **Verify:** `git tag` lists `v1.0.0`; `PROGRESS.md` has every task checked.

---

## Appendix: When something goes wrong

| Symptom | Likely cause | Fix |
|---|---|---|
| `Cannot find module '@shared/...'` | Alias missing in one of tsconfig or `electron.vite.config.ts` | Add the alias in all three places (task 1.5) |
| Blank window in dev | CSP blocks scripts or preload path wrong | Re-check task 1.9 CSP and `preload` path |
| `window.api` is undefined | Preload not built or `contextBridge` name mismatch | Check `src/preload/index.ts` exposes `'api'` and `webPreferences.preload` path |
| Streaming text appears all at once | Batcher flushing only at end, or renderer not applying deltas | Check tasks 4.3 and 6.3 tests |
| safeStorage error on add key | `app` not ready or encryption unavailable | Call it only after `app.whenReady()` (task 2.6) |
| Duplicate messages after Retry | Failed message not deleted before resume | Task 4.2 and 4.5 rules |
| Installer build fails on icon | Icon not 256x256 `.ico` | Task 12.2 |
