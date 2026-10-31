# TECH_STACK: Architecture and Tech Stack

Read `PRD.md` first. This file fixes every technical choice. **Do not substitute libraries, upgrade major versions, or add packages not listed here.**

## 1. Decision summary

| Decision | Choice | Why |
|---|---|---|
| Desktop shell | **Electron** | Most documented desktop stack; no Rust toolchain; main-process `fetch` removes browser CORS limits on provider APIs. |
| Language | **TypeScript only** (strict). No Python, no JavaScript source files. | One language across main, preload, renderer. |
| Build tool | **electron-vite** | Official-style template for Electron + Vite + React. |
| UI | **React 18** + **Tailwind CSS 3** + **Radix UI** primitives | Predictable, well-known to code models. |
| State | **Zustand** | Small, no boilerplate. |
| Validation | **Zod** | Runtime check of all stored JSON and IPC input. |
| Storage | **JSON files** in the OS user-data folder | No native modules, no rebuild problems. |
| Secrets | Electron **safeStorage** | OS-level encryption (Windows DPAPI). |
| Packaging | **electron-builder** (NSIS installer + portable) | Standard. |
| Tests | **Vitest** | Same Vite pipeline. |

## 2. Runtime and versions

Pin exactly the **major.minor** below using caret ranges in `package.json`. If `npm install` fails for a version, run `npm view <package> versions --json`, pick the newest version with the **same major**, and note it in `PROGRESS.md`.

| Tool | Version |
|---|---|
| Node.js | 22.x LTS (`"engines": {"node": ">=22 <23"}`) |
| npm | 10.x |
| Electron | ^33.2.0 |
| electron-vite | ^2.3.0 |
| Vite | ^5.4.0 |
| TypeScript | ^5.6.0 |
| React / React DOM | ^18.3.1 |
| Tailwind CSS | ^3.4.14 (with `postcss` ^8.4, `autoprefixer` ^10.4) |
| Zustand | ^4.5.5 |
| Zod | ^3.23.8 |
| electron-builder | ^25.1.8 |
| Vitest | ^2.1.0 |

## 3. Dependencies (complete list)

### 3.1 Runtime dependencies
| Package | Purpose |
|---|---|
| `react`, `react-dom` | UI |
| `zustand` | Global state |
| `zod` | Schemas and validation |
| `lucide-react` ^0.460.0 | Icons (only icons from this package; no emoji icons) |
| `react-markdown` ^9.0.1, `remark-gfm` ^4.0.0 | Render message text |
| `@radix-ui/react-dialog` ^1.1.2 | Modals |
| `@radix-ui/react-select` ^2.1.2 | Dropdowns |
| `@radix-ui/react-tooltip` ^1.1.3 | Tooltips |
| `@radix-ui/react-dropdown-menu` ^2.1.2 | Context menus (chat rename/delete) |
| `clsx` ^2.1.1, `tailwind-merge` ^2.5.4 | Class name helpers (`cn()`) |

Not used: any HTTP library (use global `fetch`), `uuid`/`nanoid` (use `crypto.randomUUID()`), `electron-store`, `better-sqlite3`, UI kits (MUI, Chakra, Ant), CSS-in-JS, Redux.

### 3.2 Dev dependencies
`electron`, `electron-vite`, `vite`, `@vitejs/plugin-react` ^4.3.3, `typescript`, `electron-builder`, `tailwindcss`, `postcss`, `autoprefixer`, `vitest`, `@testing-library/react` ^16.0.1, `@testing-library/jest-dom` ^6.6.3, `jsdom` ^25.0.1, `eslint` ^9.14.0, `typescript-eslint` ^8.14.0, `eslint-plugin-react-hooks` ^5.0.0, `eslint-plugin-react-refresh` ^0.4.14, `prettier` ^3.3.3, `prettier-plugin-tailwindcss` ^0.6.8, `@types/node` ^22, `@types/react` ^18.3, `@types/react-dom` ^18.3.

## 4. Process architecture

```
┌────────────────────────── Electron ──────────────────────────┐
│ MAIN process (Node)             PRELOAD            RENDERER  │
│  - window lifecycle        contextBridge:      React app     │
│  - storage (JSON files)    window.api = {...}  (sandboxed,   │
│  - key vault (safeStorage)  typed IPC wrappers  no Node)     │
│  - provider adapters (fetch+SSE)                              │
│  - orchestrator (turn loop)  ◄── invoke / events ──►          │
└───────────────────────────────────────────────────────────────┘
```

Rules:
1. **All network calls happen in main.** The renderer never calls `fetch` to a provider and never receives a secret.
2. **The orchestrator runs in main.** The renderer only starts, stops, and listens.
3. Renderer talks to main only through `window.api` (defined in preload, typed in `src/shared/ipc.ts`).
4. Every IPC handler validates its input with Zod and returns `Result<T>` (see `DATA_MODELS.md`). Handlers never throw to the renderer.
5. Events from main to renderer use one channel, `run:event`, carrying a `RunEvent` union.

### 4.1 BrowserWindow security (mandatory)
```ts
new BrowserWindow({
  width: 1200, height: 800, minWidth: 900, minHeight: 600,
  show: false,
  backgroundColor: '#faf9f5',
  webPreferences: {
    preload: join(__dirname, '../preload/index.js'),
    contextIsolation: true,
    nodeIntegration: false,
    sandbox: true,
    webSecurity: true,
  },
})
```
- Call `win.show()` on `ready-to-show`.
- `win.webContents.setWindowOpenHandler(() => ({ action: 'deny' }))`.
- Block navigation: `win.webContents.on('will-navigate', (e) => e.preventDefault())`.
- Remove the default menu in production: `Menu.setApplicationMenu(null)`.
- Single-instance lock: `app.requestSingleInstanceLock()`; quit if not acquired.
- CSP in `src/renderer/index.html`:
  `default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data:; font-src 'self'; connect-src 'self' ws://localhost:* http://localhost:*;`
  (the `localhost` entries are needed only for the Vite dev server).

## 5. Provider adapter contract

File: `src/main/providers/types.ts`

```ts
export interface StreamParams {
  apiKey: string
  model: string
  system: string
  user: string
  maxTokens: number
  signal: AbortSignal
  onDelta: (text: string) => void
}
export interface ProviderAdapter {
  id: ProviderId
  stream(params: StreamParams): Promise<string>   // resolves with the full text
  test(params: { apiKey: string; model: string; signal: AbortSignal }): Promise<void>
}
```

Every adapter sends exactly **one system string and one user string** per call. The user string contains the whole transcript (see section 6). This avoids per-provider role-alternation rules.

### 5.1 Endpoints

| Provider | URL | Headers | Body | Stream parse |
|---|---|---|---|---|
| anthropic | `POST https://api.anthropic.com/v1/messages` | `x-api-key`, `anthropic-version: 2023-06-01`, `content-type: application/json` | `{model, max_tokens, system, stream:true, messages:[{role:'user',content:user}]}` | SSE; on event `content_block_delta` with `delta.type==='text_delta'` take `delta.text` |
| openai | `POST https://api.openai.com/v1/chat/completions` | `Authorization: Bearer <key>` | `{model, stream:true, max_completion_tokens, messages:[{role:'system',content:system},{role:'user',content:user}]}` | SSE; take `choices[0].delta.content`; stop on `[DONE]` |
| openrouter | `POST https://openrouter.ai/api/v1/chat/completions` | `Authorization: Bearer <key>` | `{model, stream:true, max_tokens, messages:[...same...]}` | same as openai |
| gemini | `POST https://generativelanguage.googleapis.com/v1beta/models/{model}:streamGenerateContent?alt=sse` | `x-goog-api-key: <key>` | `{systemInstruction:{parts:[{text:system}]}, contents:[{role:'user',parts:[{text:user}]}], generationConfig:{maxOutputTokens}}` | SSE; join `candidates[0].content.parts[].text` |

Default models (placeholders only): anthropic `claude-sonnet-5-5`, openai `gpt-4.1`, gemini `gemini-2.5-flash`, openrouter `deepseek/deepseek-chat`.

`max tokens` for a turn = `Math.ceil(maxWords * 2.2)`; wrap-up = 700.

### 5.2 SSE reader
Implement once in `src/main/providers/sse.ts`: `async function* readSse(response: Response): AsyncGenerator<{event?: string; data: string}>`. Decode with `TextDecoder`, buffer partial lines, split events on blank lines, ignore lines starting with `:`, join multi-line `data:` fields with `\n`. Unit-test it with split-chunk input.

### 5.3 Error mapping
After `fetch`, if `!response.ok`, read the body text (max 2000 chars) and map:

| Condition | `ErrorCode` | User message |
|---|---|---|
| 401, 403 | `INVALID_KEY` | "The API key was rejected. Check it in API keys." |
| 404 | `MODEL_NOT_FOUND` | "Model not found. Check the model name." |
| 429 | `RATE_LIMITED` | "Rate limit reached. Wait a moment and retry." |
| 5xx | `SERVER` | "The provider had a problem. Retry in a moment." |
| fetch throws `TypeError` | `NETWORK` | "No internet connection or the provider is unreachable." |
| `AbortError` | `ABORTED` | (not shown as an error) |
| no bytes for 60 s | `TIMEOUT` | "The provider took too long to respond." |
| other | `UNKNOWN` | "Something went wrong." plus provider text in `details` |

Timeout: an inactivity timer reset on each chunk; on expiry abort the request and throw `TIMEOUT`.

## 6. Orchestrator rules

File: `src/main/orchestrator/runner.ts`. One active run per conversation; at most **one run in the whole app** at a time (reject a second `run:start` with `VALIDATION`).

### 6.1 Progress derivation
```
n = conversation.agents.length
done = messages.filter(m => m.kind === 'agent' && m.status === 'complete').length
if done >= n * conversation.rounds  -> go to wrap-up
else next agent index = done % n ; round = Math.floor(done / n) + 1
```
Before resuming after an error, delete messages with `status` `error` or `aborted` that are at the end of the list.

### 6.2 Transcript string
```
transcript = messages
  .filter(m => m.status === 'complete')
  .map(m => `${m.kind === 'user' ? 'User' : m.agentName}: ${m.text}`)
  .join('\n\n')
if empty -> '(nothing said yet)'
```
`agentName` is stored on each message at creation so renaming never rewrites history.

### 6.3 Prompt templates (use verbatim)

Let `others` = other agents as `Name (role)` or `Name` when role is blank, joined by `, `. Let `role` = agent role or the default. Let `W` = max words per turn setting.

**Debate system:**
```
You are {name}, taking part in a debate with {others} on: "{topic}". Your role: {role}. {stanceLine} Be direct, concrete and fair. Maximum {W} words. No preamble.
```
`stanceLine`: `for` → `You argue FOR the topic.`; `against` → `You argue AGAINST the topic.`; `free` → `Argue the position you find most defensible.`

**Co-op system:**
```
You are {name}. Your role: {role}. You are on a team of AI agents ({others}) working together on: "{topic}". Stay in your role, build on teammates' ideas, fill gaps, fix errors. Maximum {W} words. No preamble.
```

**Turn user message (both modes):**
```
Conversation so far:

{transcript}

Round {round} of {rounds}. {phase}
```

`phase` selection (first match wins):
1. Last message is a `user` message and this is agent index 0: `Respond to the user's latest message.`
2. No messages yet (first turn): Debate `Give your opening statement.` / Co-op `Propose your first contribution.`
3. `round === rounds`: Debate `Give your closing statement, answering the strongest points against you.` / Co-op `Converge: state what the team should settle on.`
4. Otherwise: Debate `Rebut the others' latest points, then advance your case.` / Co-op `Add your next contribution.`

**Judge system:** `You are an impartial debate judge. Name the winner, give 3 short reasons, and one thing each side missed. Maximum 200 words.`
**Final-answer system:** `You are the team lead. Merge the team conversation into one clear final answer. Maximum 250 words.`
**Wrap-up user message:** `Topic: {topic}\n\n{transcript}`

### 6.4 Loop pseudocode
```
run(conversationId):
  set status 'running'; emit run-started
  controller = new AbortController()
  loop while next turn exists and not controller.signal.aborted:
     create message {status:'streaming', text:''}; emit turn-started
     try  text = await adapter.stream({... onDelta: d => { message.text += d; emit turn-delta }})
     catch e -> if ABORTED: mark 'aborted', status 'stopped'; else mark 'error', status 'failed'; emit turn-failed; persist; break
     mark 'complete'; persist; emit turn-completed
  if finished all turns and wrapUpKeyId: same as a turn with kind 'wrapup'
  set final status ('completed' | 'stopped' | 'failed'); emit run-finished
```
- Persist the conversation file after every completed or failed turn, and every 2 seconds while streaming (debounced) so a crash loses at most 2 s of text.
- `turn-delta` events are batched in main to at most one event per 50 ms per message.
- If the conversation is deleted during a run, abort first, then delete.

## 7. Directory structure

Use this exact layout. It matches the `electron-vite` `react-ts` template, so keep the template's `src/main`, `src/preload`, `src/renderer/src` locations.

```
ai-debate/
├─ package.json
├─ electron.vite.config.ts
├─ electron-builder.yml
├─ tsconfig.json                 # references the two below
├─ tsconfig.node.json            # main + preload + shared
├─ tsconfig.web.json             # renderer + shared
├─ eslint.config.mjs
├─ .prettierrc.json
├─ .editorconfig
├─ .gitignore
├─ tailwind.config.ts
├─ postcss.config.js
├─ vitest.config.ts
├─ README.md
├─ PRD.md  TECH_STACK.md  DATA_MODELS.md  UI_UX_SPEC.md  CLINE_TASK_MASTER.md
├─ PROGRESS.md                   # Cline appends one line per finished task
├─ resources/
│  └─ icon.ico                   # 256x256 app icon
├─ build/                        # electron-builder assets (empty at first)
└─ src/
   ├─ shared/                    # imported by main, preload, renderer
   │  ├─ types.ts                # domain interfaces (DATA_MODELS.md)
   │  ├─ schemas.ts              # Zod schemas matching types.ts
   │  ├─ ipc.ts                  # channel names, request/response types, ApiSurface
   │  ├─ constants.ts            # limits, defaults, provider metadata
   │  └─ text.ts                 # pure helpers: trimTitle, dedupeNames, buildTranscript
   ├─ main/
   │  ├─ index.ts                # app lifecycle, window creation
   │  ├─ window.ts               # createMainWindow()
   │  ├─ ipc/
   │  │  ├─ register.ts          # registers all handlers
   │  │  ├─ keys.ts
   │  │  ├─ conversations.ts
   │  │  ├─ run.ts
   │  │  ├─ settings.ts
   │  │  └─ exporter.ts
   │  ├─ storage/
   │  │  ├─ paths.ts             # resolves data folder
   │  │  ├─ atomic.ts            # readJson, writeJsonAtomic
   │  │  ├─ keyVault.ts          # safeStorage encrypt/decrypt
   │  │  ├─ conversationStore.ts
   │  │  └─ settingsStore.ts
   │  ├─ providers/
   │  │  ├─ types.ts
   │  │  ├─ sse.ts
   │  │  ├─ errors.ts            # mapHttpError
   │  │  ├─ anthropic.ts
   │  │  ├─ openai.ts
   │  │  ├─ openrouter.ts
   │  │  ├─ gemini.ts
   │  └─ orchestrator/
   │     ├─ prompts.ts           # section 6.3 templates as pure functions
   │     ├─ progress.ts          # section 6.1
   │     └─ runner.ts            # section 6.4
   ├─ preload/
   │  ├─ index.ts                # contextBridge.exposeInMainWorld('api', api)
   │  └─ index.d.ts              # declare global { interface Window { api: ApiSurface } }
   └─ renderer/
      ├─ index.html
      └─ src/
         ├─ main.tsx
         ├─ App.tsx
         ├─ styles/
         │  └─ globals.css       # Tailwind layers + CSS variables
         ├─ lib/
         │  ├─ cn.ts
         │  ├─ api.ts            # thin wrapper: unwrap Result, throw AppError
         │  └─ format.ts         # relative time, initials
         ├─ stores/
         │  ├─ keysStore.ts
         │  ├─ conversationsStore.ts
         │  ├─ runStore.ts
         │  ├─ settingsStore.ts
         │  └─ uiStore.ts
         ├─ hooks/
         │  ├─ useRunEvents.ts   # subscribes to run:event once
         │  ├─ useAutoScroll.ts
         │  ├─ useShortcuts.ts
         │  └─ useTheme.ts
         ├─ components/
         │  ├─ ui/               # Button, IconButton, Input, Textarea, Select, Dialog, Tooltip, Badge, Toast, Spinner, SegmentedControl, Stepper
         │  ├─ layout/           # AppShell, Sidebar, SidebarItem, Header
         │  ├─ setup/            # SetupView, ModeToggle, AgentCard, SetupToolbar
         │  ├─ conversation/     # ConversationView, MessageBubble, UserMessage, WrapUpCard, FollowUpComposer, ErrorBanner
         │  ├─ keys/             # KeysDialog, KeyRow, AddKeyForm
         │  └─ settings/         # SettingsDialog
         ├─ views/
         │  ├─ WelcomeView.tsx   # no-key state
         │  ├─ SetupRoute.tsx
         │  └─ ConversationRoute.tsx
         └─ assets/              # static images only
```

Test files live next to the code as `*.test.ts(x)`.

### 7.1 Path aliases
`@shared/*` → `src/shared/*`; `@renderer/*` → `src/renderer/src/*`; `@main/*` → `src/main/*`. Configure in `electron.vite.config.ts` (`resolve.alias`) **and** both tsconfig files (`paths`).

## 8. Configuration rules

### 8.1 TypeScript
- `"strict": true`, `"noUncheckedIndexedAccess": true`, `"noImplicitOverride": true`, `"exactOptionalPropertyTypes": false`, `"noFallthroughCasesInSwitch": true`, `"forceConsistentCasingInFileNames": true`, `"isolatedModules": true`.
- **Never** use `any`. Use `unknown` and narrow with Zod. No `// @ts-ignore` or `// @ts-expect-error` without a comment explaining why.
- No non-null assertion `!` except in tests.
- Prefer `interface` for object shapes, `type` for unions.
- Exported functions have explicit return types.
- Use `import type` for type-only imports.

### 8.2 Prettier (`.prettierrc.json`)
```json
{ "semi": false, "singleQuote": true, "printWidth": 100, "trailingComma": "all", "arrowParens": "always", "plugins": ["prettier-plugin-tailwindcss"] }
```

### 8.3 ESLint (flat config)
Extend `typescript-eslint` recommended + `react-hooks` + `react-refresh`. Add rules: `@typescript-eslint/no-explicit-any: error`, `@typescript-eslint/no-unused-vars: [error, {argsIgnorePattern: '^_'}]`, `no-console: [warn, {allow: ['warn','error']}]`, `eqeqeq: error`, `@typescript-eslint/consistent-type-imports: error`.

### 8.4 Conventions
- File names: components `PascalCase.tsx`; everything else `camelCase.ts`.
- One React component per file; named exports (default export only for `App`).
- No file over 300 lines; split when exceeded.
- Functions under 50 lines where practical.
- All user-facing strings live inline in the component (no i18n layer), but error messages for `ErrorCode`s live only in `src/shared/constants.ts`.
- IPC channel names are constants in `src/shared/ipc.ts`; never type a channel string twice.
- Never log secrets. Never put secrets in error messages.
- Dates: ISO 8601 UTC strings. IDs: `crypto.randomUUID()`.

### 8.5 npm scripts (`package.json`)
```json
{
  "dev": "electron-vite dev",
  "build": "npm run typecheck && electron-vite build",
  "build:win": "npm run build && electron-builder --win --config electron-builder.yml",
  "typecheck": "tsc --noEmit -p tsconfig.node.json && tsc --noEmit -p tsconfig.web.json",
  "lint": "eslint .",
  "format": "prettier --write .",
  "test": "vitest run"
}
```

### 8.6 electron-builder (`electron-builder.yml`)
`appId: com.aidebate.app`, `productName: AI Debate`, `directories.output: dist`, `files` = built output, `win.target: [nsis, portable]`, `win.icon: resources/icon.ico`, `nsis.oneClick: false`, `nsis.allowToChangeInstallationDirectory: true`, `artifactName: AI-Debate-${version}-${target}.${ext}`.

## 9. Forbidden actions for the coding agent

1. Do not add packages outside section 3.
2. Do not create files outside the tree in section 7 (except `*.test.ts(x)`).
3. Do not read or write storage from the renderer.
4. Do not call a provider from the renderer.
5. Do not store or log raw API keys anywhere except inside `keyVault.ts` memory during a call.
6. Do not implement anything listed in `PRD.md` section 4.2.
7. Do not change the prompt templates in section 6.3.
