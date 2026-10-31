# DATA_MODELS: Schemas, Storage, State, and IPC Contract

All types in section 3 are copied verbatim into `src/shared/types.ts`. Zod schemas in section 4 go in `src/shared/schemas.ts` and must stay 1:1 with the types. IPC definitions in section 6 go in `src/shared/ipc.ts`.

## 1. Storage overview

Data is stored as JSON files under one root folder:

```
root = path.join(app.getPath('userData'), 'ai-debate-data')
```

On Windows this is `%APPDATA%\ai-debate\ai-debate-data` (the app `name` in `package.json` is `ai-debate`).

```
ai-debate-data/
├─ settings.json                  # Settings
├─ keys.json                      # StoredKeysFile (secrets encrypted)
└─ conversations/
   ├─ index.json                  # ConversationIndexFile (lightweight list)
   ├─ <conversationId>.json       # Conversation (full)
   └─ <conversationId>.json.bak   # previous good copy
```

### 1.1 File contract

| File | Content | Written when |
|---|---|---|
| `settings.json` | `SettingsFile` | Any settings change |
| `keys.json` | `StoredKeysFile` | Key added or removed |
| `conversations/index.json` | `ConversationIndexFile` | Conversation created, renamed, deleted, or its status/updatedAt changes |
| `conversations/<id>.json` | `ConversationFile` | After each completed/failed turn, every 2 s while streaming (debounced), on rename, on status change |

Every file has a top-level `"version": 1` integer. On load, if `version` is greater than the app supports, refuse to load and show an error `STORAGE`. Future versions add migration functions keyed by version.

### 1.2 Atomic write procedure (`storage/atomic.ts`)
`writeJsonAtomic(path, data)`:
1. `JSON.stringify(data, null, 2)`.
2. Write to `path + '.tmp'` with `fs.promises.writeFile`.
3. If `path` exists, copy it to `path + '.bak'` (`fs.promises.copyFile`).
4. `fs.promises.rename(path + '.tmp', path)`.

`readJson(path, schema)`:
1. If file missing → return `null`.
2. Try parse + `schema.safeParse`. If it succeeds, return the data.
3. If it fails, try `path + '.bak'` the same way. If it succeeds, return that and log a warning.
4. If both fail → throw `STORAGE` error (do not delete the files).

All writes for one file go through a per-path promise queue so writes never interleave.

### 1.3 Key vault (`storage/keyVault.ts`)
- Secrets are encrypted with `safeStorage.encryptString(secret)` and stored as base64 in `keys.json` field `secretEnc`.
- `last4` (last 4 characters of the secret) is stored in plain text for display.
- Decrypt only inside `getSecret(keyId)`, called from the orchestrator and Test. Never return a secret over IPC.
- If `safeStorage.isEncryptionAvailable()` is `false`, `addKey` returns `STORAGE` with message "Secure storage is not available on this system."

## 2. Limits and defaults (`src/shared/constants.ts`)

```ts
export const LIMITS = {
  topicMax: 2000,
  agentNameMax: 40,
  agentRoleMax: 120,
  keyLabelMax: 40,
  modelMax: 120,
  titleMax: 80,
  agentsMin: 2,
  agentsMax: 6,
  roundsMin: 1,
  roundsMax: 8,
  wordsMin: 50,
  wordsMax: 400,
} as const

export const DEFAULTS = {
  rounds: 3,
  agents: 2,
  wordsPerTurn: 150,
  theme: 'system',
} as const

export const PROVIDERS = {
  anthropic: { label: 'Anthropic', defaultModel: 'claude-sonnet-5-5' },
  openai: { label: 'OpenAI', defaultModel: 'gpt-4.1' },
  gemini: { label: 'Google Gemini', defaultModel: 'gemini-2.5-flash' },
  openrouter: { label: 'OpenRouter', defaultModel: 'deepseek/deepseek-chat' },
} as const
```

## 3. Core types (`src/shared/types.ts`)

```ts
export type ProviderId = 'anthropic' | 'openai' | 'gemini' | 'openrouter'
export type Mode = 'debate' | 'coop'
export type Stance = 'for' | 'against' | 'free'
export type ThemePref = 'system' | 'light' | 'dark'

export type ErrorCode =
  | 'INVALID_KEY' | 'MODEL_NOT_FOUND' | 'RATE_LIMITED' | 'SERVER'
  | 'NETWORK' | 'TIMEOUT' | 'ABORTED' | 'VALIDATION'
  | 'NOT_FOUND' | 'STORAGE' | 'UNKNOWN'

export interface AppError {
  code: ErrorCode
  message: string      // safe, human-readable, never contains secrets
  details?: string     // optional provider text, max 500 chars
}

export type Result<T> = { ok: true; data: T } | { ok: false; error: AppError }

/** What the renderer sees. Never contains the secret. */
export interface ApiKeyInfo {
  id: string
  provider: ProviderId
  label: string
  model: string
  last4: string
  createdAt: string    // ISO 8601
}

/** What is written to keys.json. Main process only. */
export interface StoredKey extends ApiKeyInfo {
  secretEnc: string    // base64 of safeStorage ciphertext
}

export interface StoredKeysFile {
  version: 1
  keys: StoredKey[]
}

export interface Agent {
  id: string
  name: string         // final, deduplicated display name
  keyId: string        // references ApiKeyInfo.id
  role: string         // may be ''
  stance: Stance       // ignored in coop mode, store 'free'
}

export type MessageKind = 'agent' | 'user' | 'wrapup'
export type MessageStatus = 'streaming' | 'complete' | 'error' | 'aborted'

export interface Message {
  id: string
  kind: MessageKind
  agentId: string | null     // null for user messages; for wrapup, the wrap-up key's pseudo-agent id 'wrapup'
  agentName: string          // snapshot at creation; 'You' for user; 'Judge' or 'Final answer' for wrapup
  agentRole: string          // snapshot; '' if none
  stance: Stance | null      // debate agent messages only
  round: number | null       // 1-based; null for user and wrapup
  text: string
  status: MessageStatus
  error: AppError | null     // set when status === 'error'
  keyLabel: string           // snapshot of the key label, for display in wrap-up header
  createdAt: string
}

export type ConversationStatus = 'idle' | 'running' | 'completed' | 'stopped' | 'failed'

export interface Conversation {
  id: string
  title: string
  mode: Mode
  topic: string
  rounds: number             // total rounds, grows by 1 on each follow-up
  agents: Agent[]            // length 2..6
  wrapUpKeyId: string | null
  wordsPerTurn: number       // snapshot of the setting at creation
  status: ConversationStatus
  messages: Message[]
  createdAt: string
  updatedAt: string
}

export interface ConversationFile {
  version: 1
  conversation: Conversation
}

export interface ConversationSummary {
  id: string
  title: string
  mode: Mode
  status: ConversationStatus
  createdAt: string
  updatedAt: string
}

export interface ConversationIndexFile {
  version: 1
  items: ConversationSummary[]   // sorted by updatedAt descending
}

export interface Settings {
  theme: ThemePref
  wordsPerTurn: number           // 50..400
}

export interface SettingsFile {
  version: 1
  settings: Settings
}

/** Input to create a conversation (renderer → main). */
export interface NewConversationInput {
  mode: Mode
  topic: string
  rounds: number
  wrapUpKeyId: string | null
  agents: Array<{ name: string; keyId: string; role: string; stance: Stance }>
}

export type ExportFormat = 'markdown' | 'text'

export type RunEvent =
  | { type: 'run-started'; conversationId: string }
  | { type: 'turn-started'; conversationId: string; message: Message }
  | { type: 'turn-delta'; conversationId: string; messageId: string; delta: string }
  | { type: 'turn-completed'; conversationId: string; message: Message }
  | { type: 'turn-failed'; conversationId: string; message: Message }
  | { type: 'run-finished'; conversationId: string; status: ConversationStatus }
```

`turn-started`, `turn-completed`, and `turn-failed` are used for all message kinds including `wrapup`. `turn-completed` carries the full final message so the renderer replaces its streaming draft.

## 4. Zod schemas (`src/shared/schemas.ts`)

Implement one schema per type with these exact constraints (names follow `<Type>Schema`):

| Schema | Constraints |
|---|---|
| `ProviderIdSchema` | `z.enum(['anthropic','openai','gemini','openrouter'])` |
| `ModeSchema` | `z.enum(['debate','coop'])` |
| `StanceSchema` | `z.enum(['for','against','free'])` |
| `NewKeyInputSchema` | `{provider, label: string trim max 40 (may be empty), model: string trim max 120 (may be empty), secret: string trim min 8 max 500}` |
| `NewConversationInputSchema` | `mode`; `topic` trim min 1 max 2000; `rounds` int 1..8; `wrapUpKeyId` string or null; `agents` array min 2 max 6 of `{name: trim max 40, keyId: string min 1, role: trim max 120, stance}` |
| `SettingsSchema` | `theme` enum; `wordsPerTurn` int 50..400 |
| `RenameInputSchema` | `{id: string, title: trim min 1 max 80}` |
| `StoredKeysFileSchema`, `ConversationFileSchema`, `ConversationIndexFileSchema`, `SettingsFileSchema` | Match section 3 exactly, `version: z.literal(1)` |

Derive TypeScript types from the schemas only for IPC inputs; the interfaces in section 3 remain the source of truth for domain objects. Add a compile-time check in `schemas.ts`: `const _check: z.infer<typeof ConversationFileSchema> = null as unknown as ConversationFile` (and vice versa) so drift fails typecheck.

## 5. Business rules for data

1. **Creating a conversation:** validate input; verify every `keyId` exists; assign `id`, timestamps; compute final agent names: `name.trim() || key.label`, then for duplicates append ` 2`, ` 3`, ... in order of appearance. Set `agent.stance = 'free'` when mode is `coop`. Set `title = topic` collapsed to one line and cut to 60 chars with `…` if cut. Set `wordsPerTurn` from current settings. Status `idle`. Save conversation and index.
2. **Updating index:** whenever a conversation changes status, title, or `updatedAt`, rewrite its entry and re-sort descending by `updatedAt`.
3. **Follow-up:** append a `user` message (`status: 'complete'`), `rounds += 1`, remove any `wrapup` message, then start the run.
4. **Delete conversation:** abort its run if active; delete `<id>.json` and `<id>.json.bak`; remove the index entry.
5. **Startup repair:** for each index item with status `running`, set it to `stopped`; for each message with status `streaming`, set it to `aborted`. Persist.
6. **Index rebuild:** if `index.json` is missing or invalid, rebuild it by reading every `conversations/*.json`.
7. **Message ids and ordering:** messages are append-only, ordered by creation. The only mutations are `text`, `status`, `error` on the last message, and deletion of trailing `error`/`aborted` messages before a retry.

## 6. IPC contract (`src/shared/ipc.ts`)

```ts
export const CH = {
  keysList: 'keys:list',
  keysAdd: 'keys:add',
  keysRemove: 'keys:remove',
  keysTest: 'keys:test',
  convList: 'conv:list',
  convGet: 'conv:get',
  convCreate: 'conv:create',
  convRename: 'conv:rename',
  convDelete: 'conv:delete',
  runStart: 'run:start',
  runStop: 'run:stop',
  runFollowUp: 'run:followUp',
  runRetry: 'run:retry',
  exportConv: 'export:conv',
  settingsGet: 'settings:get',
  settingsSet: 'settings:set',
  runEvent: 'run:event',        // main → renderer only
} as const
```

| Channel | Request | Response `Result<...>` | Notes |
|---|---|---|---|
| `keys:list` | none | `ApiKeyInfo[]` | |
| `keys:add` | `NewKeyInput` | `ApiKeyInfo` | Empty label → provider label; empty model → provider default |
| `keys:remove` | `{id}` | `null` | |
| `keys:test` | `{id}` | `null` | Sends a 1-token request; maps errors per TECH_STACK 5.3 |
| `conv:list` | none | `ConversationSummary[]` | |
| `conv:get` | `{id}` | `Conversation` | |
| `conv:create` | `NewConversationInput` | `Conversation` | Does not start the run |
| `conv:rename` | `{id, title}` | `ConversationSummary` | |
| `conv:delete` | `{id}` | `null` | |
| `run:start` | `{id}` | `null` | Returns immediately after `run-started`; progress arrives as events |
| `run:stop` | `{id}` | `null` | |
| `run:followUp` | `{id, text}` | `null` | `text` trim min 1 max 2000; rejects with `VALIDATION` if running |
| `run:retry` | `{id}` | `null` | Allowed only if status is `failed`; deletes trailing error message, then runs |
| `export:conv` | `{id, format}` | `{path: string} \| null` | Opens native save dialog; `null` if cancelled |
| `settings:get` | none | `Settings` | |
| `settings:set` | `Settings` | `Settings` | |

`ApiSurface` (exposed as `window.api`):
```ts
export interface ApiSurface {
  keys: {
    list(): Promise<Result<ApiKeyInfo[]>>
    add(input: NewKeyInput): Promise<Result<ApiKeyInfo>>
    remove(id: string): Promise<Result<null>>
    test(id: string): Promise<Result<null>>
  }
  conversations: {
    list(): Promise<Result<ConversationSummary[]>>
    get(id: string): Promise<Result<Conversation>>
    create(input: NewConversationInput): Promise<Result<Conversation>>
    rename(id: string, title: string): Promise<Result<ConversationSummary>>
    remove(id: string): Promise<Result<null>>
  }
  run: {
    start(id: string): Promise<Result<null>>
    stop(id: string): Promise<Result<null>>
    followUp(id: string, text: string): Promise<Result<null>>
    retry(id: string): Promise<Result<null>>
    onEvent(cb: (e: RunEvent) => void): () => void   // returns unsubscribe
  }
  exportConversation(id: string, format: ExportFormat): Promise<Result<{ path: string } | null>>
  settings: {
    get(): Promise<Result<Settings>>
    set(s: Settings): Promise<Result<Settings>>
  }
}
```

## 7. Export formats

**Markdown:**
```
# {title}

*{Debate|Co-op} · {N} agents · {rounds} rounds · exported {YYYY-MM-DD HH:mm}*

**Topic:** {topic}

## Agents
- {name} — {role or "no role"} — {key label / model}{ — stance (debate only)}

## Conversation

### {agentName} ({role}, round {n})
{text}

### You
{text}

## {Judge verdict | Final answer}
{text}
```
**Text:** same content, no Markdown symbols: `{agentName}: {text}` blocks separated by blank lines, header lines `Topic: ...`, `Mode: ...`.
Only `complete` messages are exported. The file name suggestion is `slug(title).md|txt` where `slug` lowercases, replaces non-alphanumerics with `-`, and trims to 50 chars.

## 8. Global state shape (renderer, Zustand)

Persistence rule: **the renderer persists nothing**. Everything durable lives in main-process files. Renderer stores are caches populated from IPC at startup.

```ts
// stores/keysStore.ts
interface KeysState {
  keys: ApiKeyInfo[]
  loaded: boolean
  load(): Promise<void>
  add(input: NewKeyInput): Promise<ApiKeyInfo>      // throws AppError
  remove(id: string): Promise<void>
}

// stores/conversationsStore.ts
interface ConversationsState {
  summaries: ConversationSummary[]
  activeId: string | null            // null → show Setup view
  active: Conversation | null        // full object for activeId
  loadList(): Promise<void>
  open(id: string): Promise<void>
  newChat(): void                    // activeId = null, active = null
  create(input: NewConversationInput): Promise<Conversation>
  rename(id: string, title: string): Promise<void>
  remove(id: string): Promise<void>
  applyEvent(e: RunEvent): void      // merges turn events into `active` and updates summaries
}

// stores/runStore.ts
interface RunState {
  runningId: string | null           // conversation currently running (at most one)
  setRunning(id: string | null): void
}

// stores/settingsStore.ts
interface SettingsState {
  settings: Settings
  load(): Promise<void>
  update(patch: Partial<Settings>): Promise<void>
}

// stores/uiStore.ts   (ephemeral, not persisted)
interface UiState {
  sidebarOpen: boolean
  keysDialogOpen: boolean
  settingsDialogOpen: boolean
  toasts: Array<{ id: string; kind: 'success' | 'error' | 'info'; text: string }>
  setupDraft: SetupDraft              // survives switching chats during a session
  pushToast(t: Omit<Toast, 'id'>): void
  dismissToast(id: string): void
}

interface SetupDraft {
  mode: Mode
  topic: string
  rounds: number
  wrapUpKeyId: string | null
  agents: Array<{ name: string; keyId: string; role: string; stance: Stance }>
}
```

### 8.1 What lives where

| Data | Location | Reason |
|---|---|---|
| API secrets | Main only, `keys.json` encrypted | Security |
| Key metadata (label, model, last4) | Disk + `keysStore` | Needed for pickers |
| Conversations, messages | Disk + `conversationsStore.active` | Durable history |
| Streaming draft text | `conversationsStore.active.messages[last].text` (updated by `turn-delta`) | Live render |
| Settings | Disk + `settingsStore` | Durable |
| Setup form values | `uiStore.setupDraft` | Survives chat switching, not restarts |
| Open/closed dialogs, toasts, sidebar | `uiStore` | Ephemeral |
| Hover, focus, input text of a follow-up, select open state | Component `useState` | Local only |

### 8.2 `applyEvent` rules
- Ignore events whose `conversationId` is not `activeId`, except `run-finished`, which always updates the matching summary status and clears `runningId`.
- `turn-started`: append `message` to `active.messages`.
- `turn-delta`: find message by `messageId`; append `delta` to `text`. Batch in the store with one `requestAnimationFrame` flush.
- `turn-completed` / `turn-failed`: replace the message with the same `id`.
- `run-started`: set `runningId`, set `active.status = 'running'`.
- `run-finished`: set `active.status`, clear `runningId`.

### 8.3 Startup sequence (renderer)
1. `settingsStore.load()` and apply theme.
2. `keysStore.load()`.
3. `conversationsStore.loadList()`.
4. Subscribe once to `window.api.run.onEvent` (in `useRunEvents`).
5. Render. If `keys.length === 0`, show `WelcomeView`; otherwise show the Setup view.
