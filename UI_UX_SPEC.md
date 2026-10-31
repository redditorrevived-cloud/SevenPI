# UI_UX_SPEC: Interface and Design System

The look is modelled on the Claude desktop app: calm warm neutrals, a quiet sidebar, a centered reading column, a large rounded composer, serif display headings. Use the tokens in section 8 exactly. Do not invent colors, sizes, or components not listed here.

## 1. Layout blueprint

Window: default 1200x800, minimum 900x600. Native frame (no custom title bar in MVP).

```
┌──────────────┬───────────────────────────────────────────────────┐
│ SIDEBAR 280  │ HEADER (52px)  [title ...........] [badge][Stop][Export] │
│              ├───────────────────────────────────────────────────┤
│ AI Debate    │                                                   │
│ [+ New chat] │   MAIN CONTENT (scrolls)                          │
│              │   centered column, max-width 720px                │
│ Recents      │                                                   │
│  chat item   │   Setup view  OR  Conversation view              │
│  chat item   │                                                   │
│  ...         ├───────────────────────────────────────────────────┤
│ ───────────  │ DOCK: FollowUpComposer (Conversation view only)  │
│ [API keys]   │                                                   │
│ [Settings]   │                                                   │
└──────────────┴───────────────────────────────────────────────────┘
```

Regions:
- **Sidebar:** 280 px wide, background `--bg-side`, right border `--line`. Collapsible. Top: app name (serif, 19 px, semi-bold) and a "New chat" button. Middle: "Recents" label and scrollable chat list. Bottom (pinned): **API keys** and **Settings** buttons.
- **Header:** 52 px high, no border, shows chat title (ellipsis), mode badge, **Stop** (only while running), **Export** (only with an active chat), and a sidebar toggle icon at the far left when the sidebar is collapsed.
- **Main content:** vertical scroll area; content in a centered column.
- **Dock:** bottom area with the follow-up composer; hidden in Setup view and while running.
- **Modals** (Radix Dialog, centered, 560 px wide max, overlay `rgba(0,0,0,0.4)`): API Keys, Settings, Confirm Delete, Rename.

## 2. Screens

### 2.1 Welcome view (no keys)
Centered serif heading "Add an API key to begin" (28 px), one muted line "You can add several, from different providers.", primary button **Add API key** (opens Keys dialog).

### 2.2 Setup view (new chat)
From top to bottom, inside the 720 px column:
1. **Hero heading** (serif, 32 px, centered): Debate → "What should the agents debate?"; Co-op → "What should the team work on?". Top margin 7vh.
2. **Composer card** (radius 20, border `--line`, shadow `0 2px 14px rgba(0,0,0,.06)`):
   - Multi-line textarea (min 64 px, auto-grows to 220 px), placeholder: Debate "State the topic, e.g. Should schools ban phones?", Co-op "Describe the task, e.g. Plan a launch for a budgeting app". Character counter appears at 1800+ chars ("1850 / 2000").
   - Toolbar row (wraps on narrow widths): `ModeToggle` (Debate | Co-op), Agents stepper chip, Rounds chip (numeric input), Wrap-up chip (select labelled "Judge" or "Final answer"; first option "None"), and the round **Send** button at the far right (36x36, accent background, arrow-up icon).
3. **Agents grid** (margin-top 20): CSS grid `repeat(auto-fill, minmax(220px, 1fr))`, gap 12. One `AgentCard` per agent.

`AgentCard` content: avatar circle with index number; name input (placeholder = selected key's label); key/model select; role input with suggestions datalist (Skeptic, Optimist, Researcher, Devil's advocate, Engineer, Editor, Strategist, Creative thinker); stance select (Debate only: "Argues for", "Argues against", "Own view").

Send is disabled until the topic is non-empty. Ctrl+Enter also sends.

### 2.3 Conversation view
Column of items in order:
- `MessageBubble` (agent): avatar circle (agent color, initial), name in semi-bold 13 px, muted meta line "role · stance · round N", then the message text rendered as Markdown (15 px, line-height 1.6).
- `UserMessage`: right-aligned pill-like block, background `--bg-side`, radius 14, max-width 85%.
- `WrapUpCard`: border `--accent`, background `--accent-soft`, radius 14, padding 14x16; title "Judge · {key label}" or "Final answer · {key label}".
- `ErrorBanner` (when the last message failed): red text line with message and **Retry** button.
- "Thinking..." muted line while waiting for the first token of a streaming message (text empty).

Auto-scroll: keep the view pinned to the bottom while new text arrives **only if** the user was within 80 px of the bottom before the update. Otherwise show a floating "Jump to latest" pill (bottom center above the dock) that scrolls down on click.

### 2.4 API Keys dialog
Title "API keys". Muted note: "Keys are stored encrypted on this computer and sent only to the provider you pick." List of `KeyRow`s: label (bold), "provider · model · ...last4" (muted), **Test** button, **Remove** button. Below: `AddKeyForm` with provider select, name input (placeholder "Name, e.g. Claude"), model input (placeholder = provider default), key input (password type, with show/hide eye toggle), **Add key** primary button. Footer: **Done**.

### 2.5 Settings dialog
Title "Settings". Rows: Theme (segmented: System, Light, Dark); Words per turn (number input 50 to 400, with muted hint "Applies to new conversations"). Footer: **Done**. Changes apply immediately.

### 2.6 Other dialogs
- **Rename chat:** single input prefilled; buttons Cancel / Save.
- **Delete chat:** "Delete this chat? This can't be undone." buttons Cancel / Delete (danger style). If the chat is running, text adds "The running conversation will be stopped."
- **Export:** a small dropdown menu from the Export button with "Markdown (.md)" and "Plain text (.txt)".

## 3. Component inventory

All in `src/renderer/src/components/`. Props are exact; add no others.

| Component | Props | Notes |
|---|---|---|
| `ui/Button` | `variant: 'primary'\|'secondary'\|'ghost'\|'danger'`, `size: 'sm'\|'md'`, `loading?: boolean`, standard button props | Loading shows Spinner and disables |
| `ui/IconButton` | `icon: LucideIcon`, `label: string` (aria-label + tooltip), `onClick`, `disabled?` | 32x32 |
| `ui/Input`, `ui/Textarea` | standard props + `invalid?: boolean` | |
| `ui/Select` | `value`, `onChange`, `options: {value,label}[]`, `ariaLabel`, `placeholder?` | Radix Select |
| `ui/Dialog` | `open`, `onOpenChange`, `title`, `children`, `footer?` | Traps focus; Esc closes |
| `ui/Tooltip` | `content`, `children` | 400 ms delay |
| `ui/Badge` | `children`, `tone: 'accent'\|'muted'` | |
| `ui/Toast` + `ToastHost` | read from `uiStore.toasts` | Bottom-right, auto-dismiss 4 s (errors 7 s) |
| `ui/Spinner` | `size?: number` | CSS spin, 16 px default |
| `ui/SegmentedControl` | `value`, `onChange`, `options: {value,label}[]`, `ariaLabel` | Used for Mode and Theme |
| `ui/Stepper` | `value`, `min`, `max`, `onChange`, `label` | − value + |
| `layout/AppShell` | `children` | Sidebar + header + main + dock + ToastHost |
| `layout/Sidebar` | none | Reads stores |
| `layout/SidebarItem` | `summary`, `active`, `onOpen`, `onRename`, `onDelete` | Context menu via "..." icon on hover |
| `layout/Header` | none | |
| `setup/SetupView` | none | Reads `uiStore.setupDraft` |
| `setup/ModeToggle` | `value: Mode`, `onChange` | |
| `setup/AgentCard` | `index`, `agent`, `mode`, `keys`, `onChange` | |
| `conversation/ConversationView` | none | |
| `conversation/MessageBubble` | `message`, `agentIndex`, `mode` | |
| `conversation/UserMessage` | `message` | |
| `conversation/WrapUpCard` | `message`, `mode` | |
| `conversation/FollowUpComposer` | `disabled`, `onSend(text)` | Enter sends, Shift+Enter newline |
| `conversation/ErrorBanner` | `error`, `onRetry` | |
| `keys/KeysDialog`, `keys/KeyRow`, `keys/AddKeyForm` | per 2.4 | |
| `settings/SettingsDialog` | none | |

Avatar colors: agent index 0..5 maps to the `--agent-1..6` tokens. The wrap-up card uses `--accent`.

## 4. Interactive states for every view

Each state below must be implemented and visually distinct. "Toast" means a `ToastHost` notification.

### 4.1 Welcome view
| State | Behaviour |
|---|---|
| Loading | While `keysStore.loaded === false`, show a centered Spinner (no heading flash). |
| Empty | The default content of this view (no keys): heading, line, **Add API key** button. |
| Error | If `keys:list` fails: heading "Couldn't load your keys", the error message, **Try again** button. |
| Success | After a key is added the view is replaced by the Setup view and a success toast "Key added" appears. |

### 4.2 Setup view
| State | Behaviour |
|---|---|
| Loading | Never blocks; renders from the draft immediately. The Send button shows a Spinner while `conv:create` is pending and everything is disabled. |
| Empty | Initial: empty textarea, 2 default agents, Send disabled, helper text under the composer in muted 13 px: "Pick a model and a role for each agent." |
| Error | Inline red message under the composer for validation (topic empty, key missing) and for `conv:create` failures. A deleted key in an agent card shows its select in an invalid state with the text "Key removed. Choose another." |
| Success | On create success, switch to the Conversation view and clear `setupDraft.topic` (keep agents and settings). |

### 4.3 Conversation view
| State | Behaviour |
|---|---|
| Loading | Opening a chat from the sidebar: skeleton of three message rows (avatar circle + two gray lines, pulsing) until `conv:get` returns. |
| Empty | A conversation with zero messages and status `idle` (rare): muted line "Starting..." and the run starts automatically. |
| Streaming | Last message shows text growing with a blinking caret (`▍`, 1 s blink) at the end; header shows **Stop**; dock hidden. Before the first token: "Thinking..." muted. |
| Error | `ErrorBanner` under the failed message; message text so far stays visible; **Retry** button; dock stays hidden until retry or the user starts a new chat. Toast for non-turn errors (e.g., `run:start` rejected). |
| Stopped | Last message gets a muted "Stopped" tag. Dock shown. |
| Success | Status `completed`: wrap-up card (if configured) visible; dock enabled with placeholder "Add a message and run another round...". |

### 4.4 Sidebar chat list
| State | Behaviour |
|---|---|
| Loading | 5 skeleton rows. |
| Empty | Muted text "No chats yet". |
| Error | "Couldn't load chats" with **Try again** link-button. |
| Success | List sorted by `updatedAt` descending; active row highlighted; a small pulsing dot on the running chat. |

### 4.5 API Keys dialog
| State | Behaviour |
|---|---|
| Loading | Spinner in list area. |
| Empty | "No keys yet." plus the form. |
| Error | Inline red text above the form: add failure message (e.g., "Secure storage is not available on this system."). |
| Success | New row appears; form clears; row-level **Test** shows Spinner, then a green check "Works" or a red message from the mapped error; both revert after 5 s. |

### 4.6 Settings dialog
| State | Behaviour |
|---|---|
| Loading | Controls disabled until `settingsStore` loaded (normally instant). |
| Empty | Not applicable (defaults always exist). |
| Error | Inline red text if `settings:set` fails; the control reverts to its previous value. |
| Success | Change applies immediately (theme switches at once); no toast. |

### 4.7 Export
| State | Behaviour |
|---|---|
| Loading | Export button shows Spinner while the dialog/write is pending. |
| Empty | Export button disabled when the conversation has no complete messages. |
| Error | Error toast with the message. |
| Success | Toast "Exported to {file name}". Cancelled dialog shows nothing. |

## 5. Interaction rules

- **Keyboard shortcuts** (registered in `useShortcuts`, ignored while typing in inputs unless noted): `Ctrl+N` new chat; `Ctrl+B` toggle sidebar; `Ctrl+,` open Settings; `Ctrl+K` open API keys; `Ctrl+Enter` send from Setup (works in textarea); `Esc` close dialog; `Enter` send follow-up.
- **Focus:** on entering Setup view, focus the topic textarea. After a dialog closes, return focus to the trigger.
- **Disabled while running:** mode toggle, agent controls (Setup is hidden anyway), follow-up composer. Only Stop, sidebar navigation, and header controls stay active.
- **Switching chats during a run:** allowed; the run continues in main. Returning shows the live state. The sidebar dot remains.
- **Starting a second run** while one is active: Send is disabled with tooltip "Another conversation is running."
- **Hover actions** on sidebar items: "..." icon appears with Rename and Delete.
- **Motion:** only these transitions: 120 ms background/border color on buttons and list items; 150 ms dialog fade/scale (0.98 to 1); caret blink; skeleton pulse. Respect `prefers-reduced-motion` by disabling all of them.
- **Text selection** enabled in messages; copy button (Lucide `Copy`, ghost IconButton) appears on message hover in the top-right of the bubble and copies the raw text; toast "Copied".

## 6. Responsive desktop resizing

| Window width | Behaviour |
|---|---|
| ≥ 1100 px | Sidebar open (280 px). Content column 720 px max, centered. |
| 900 to 1099 px | Sidebar auto-collapses on first resize into this range (user can reopen as an overlay with a scrim, 280 px, over the content). |
| Height < 700 px | Hero top margin reduces to 2vh; hero font 26 px. |
| Column < 560 px wide | Toolbar chips wrap to two rows; Agent grid becomes one column. |

The conversation column is always `width: 100%; max-width: 720px; margin: 0 auto; padding: 0 24px`. Never allow horizontal scroll in the main area; code blocks and tables inside messages scroll horizontally in their own container.

## 7. Content rules

- Sentence case everywhere ("New chat", "API keys").
- Buttons use verbs ("Add key", "Start", "Retry").
- Error text states what happened and what to do, never shows stack traces or raw JSON.
- No emoji in UI chrome. Icons only from `lucide-react`, 16 px in buttons, 18 px elsewhere, stroke 1.75.
- Icons used: `Plus`, `PanelLeft`, `KeyRound`, `Settings`, `Square` (Stop), `Download` (Export), `ArrowUp` (Send), `Minus`, `Check`, `X`, `Copy`, `MoreHorizontal`, `Trash2`, `Pencil`, `Eye`, `EyeOff`, `AlertCircle`, `ChevronDown`.

## 8. Design system tokens

Define as CSS variables in `styles/globals.css`; map them in `tailwind.config.ts` (`theme.extend.colors` using `var(--token)`). Dark values apply under `.dark` on `<html>`; `useTheme` toggles the class (for `system`, follow `matchMedia('(prefers-color-scheme: dark)')`).

### 8.1 Colors

| Token | Light | Dark |
|---|---|---|
| `--bg` | `#faf9f5` | `#262624` |
| `--bg-side` | `#f0eee6` | `#1f1e1d` |
| `--panel` | `#ffffff` | `#30302e` |
| `--ink` | `#29261f` | `#eceae3` |
| `--ink-muted` | `#85816f` | `#9a9688` |
| `--line` | `#e6e3d8` | `#3d3d3a` |
| `--accent` | `#c15f3c` | `#d9775a` |
| `--accent-ink` (text on accent) | `#ffffff` | `#1a1410` |
| `--accent-soft` | `#f6e6de` | `#3c2f2a` |
| `--danger` | `#c92a2a` | `#ff8787` |
| `--success` | `#2f9e44` | `#69db7c` |
| `--agent-1..6` | `#c15f3c #3b5bdb #2f9e44 #9c36b5 #0c8599 #c2255c` | same |

Contrast: `--ink` on `--bg` and `--accent-ink` on `--accent` must meet WCAG AA (4.5:1). `--ink-muted` is only for secondary text of 13 px or larger.

### 8.2 Typography
- Body/UI: `system-ui, -apple-system, "Segoe UI", sans-serif`.
- Display (sidebar title, hero, empty-state headings): `Georgia, "Iowan Old Style", serif`.
- Sizes: 13 (meta), 14 (controls), 15 (body, messages), 19 (sidebar title), 26/28/32 (headings). Line-heights: 1.6 body, 1.2 headings. Weights: 400, 500, 600 only.

### 8.3 Spacing, radius, elevation
- Spacing scale (px): 4, 8, 12, 16, 20, 24, 32, 48. Use only these.
- Radius: 8 (inputs, small buttons), 10 (buttons, chips), 14 (cards, message blocks), 20 (composer), 99 (badges, avatars).
- Borders: 1 px `--line`.
- Shadows: composer `0 2px 14px rgba(0,0,0,.06)`; dialogs `0 12px 40px rgba(0,0,0,.2)`; nothing else has a shadow.
- Control heights: buttons md 36, sm 28; inputs 36; icon buttons 32; chips 32.
- Focus ring: `outline: 2px solid var(--accent); outline-offset: 1px` on `:focus-visible`.

### 8.4 Markdown inside messages
Paragraph spacing 8 px; lists indented 20 px; inline code background `--bg-side`, radius 4, padding 1x4, monospace `ui-monospace, "Cascadia Code", Consolas, monospace` 13.5 px; code blocks background `--bg-side`, radius 10, padding 12, horizontal scroll; links `--accent`, underline on hover; headings in messages rendered as bold body-size text (no large headings).

### 8.5 Scrollbars
Thin custom scrollbars (8 px, thumb `--line`, radius 99) via `::-webkit-scrollbar` rules.
