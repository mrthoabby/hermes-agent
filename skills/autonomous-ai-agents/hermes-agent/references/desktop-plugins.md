# Desktop App Plugins — UI Panes, Commands, Widgets

Write plugins for the Hermes desktop app: statusbar items, layout panes,
command-palette commands, keybinds, routes, and themes. A plugin is a single
plain-JavaScript ESM file the app loads at runtime — no build step, no repo
changes. A plugin can also talk to its own Python backend namespace
(`ctx.rest`/`ctx.socket` → `/api/plugins/<id>`); the general Python plugin
system (`~/.hermes/plugins/`) is otherwise documented separately.

Full human reference (every export, area payloads, backend, security):
`website/docs/developer-guide/desktop-plugin-sdk.md`.

## Current Hermes flow

Hermes has **three different plugin systems** that people often mix up:

- **Desktop UI plugins** — plain ESM files under
  `$HERMES_HOME/desktop-plugins/<id>/plugin.js`. This is the surface this
  reference covers.
- **General Python plugins** — `plugin.yaml` + `__init__.py` under
  `$HERMES_HOME/plugins/<id>/`. Use these for agent tools, hooks, slash
  commands, CLI commands, and optional backend APIs.
- **Dashboard plugins** — separate browser-side extensions for
  `hermes dashboard`; they do not share the desktop SDK.

If a desktop plugin needs only UI plus gateway RPC, keep it as a **desktop
file only**. If it also needs custom server-side logic, ship an **optional
paired backend** at `$HERMES_HOME/plugins/<id>/dashboard/manifest.json` +
`plugin_api.py`; Hermes serves that namespace at `/api/plugins/<id>`, but
only after the general plugin is enabled with `hermes plugins enable <id>`
or added to `plugins.enabled` in `config.yaml`.

## When to Use

- The user asks for a new desktop UI element (a pane, a statusbar widget, a
  dashboard, a command) without modifying the app itself.
- You want to surface data you compute (via gateway RPC) inside the app.

## Prerequisites

- The Hermes desktop app (it loads plugins; the CLI/gateway alone does not).
- Write access to `$HERMES_HOME/desktop-plugins/` (usually
  `~/.hermes/desktop-plugins/`).

## How to Run

1. Create `$HERMES_HOME/desktop-plugins/<name>/plugin.js` from
   `templates/plugin.js` (in this skill directory) — that's
   `~/.hermes/...` by default, or `~/.hermes/profiles/<profile>/...` under a
   named profile. Keep `<name>` equal to the plugin `id`.
2. The desktop app watches that directory: the plugin loads within a few
   seconds of the file landing, and every later save hot-reloads it in
   place. No reload step. (Fallback if it doesn't appear: ⌘K →
   **Reload desktop plugins**.)
3. If loading fails the app shows a toast naming the error — fix the file
   and save again.
4. If the plugin also needs a Python backend, create the matching general
   plugin under `$HERMES_HOME/plugins/<id>/dashboard/`, then enable it with
   `hermes plugins enable <id>` so `ctx.rest(...)` / `ctx.socket(...)` have a
   live backend to talk to.

## Quick Reference

The ONLY import surface is `@hermes/plugin-sdk` (plus `react` /
`react/jsx-runtime`, which resolve to the app's own React — write UI with
`jsx()` calls, not JSX syntax; the file is not compiled).

- `host.state.*` — readonly reactive atoms: `activeSessionId`, `cwd`,
  `gateway`, `model`, `profile`, `viewport`. Read with `.get()` in handlers,
  `useValue(atom)` in components.
- `host.request(method, params)` — gateway JSON-RPC (sessions, config,
  skills, cron — everything the app uses).
- `host.onEvent(type, fn)` — live gateway events (`'*'` for all). Returns a
  disposer.
- `host.notify({ kind, message })`, `host.navigate(path)`, `host.logs(...)`,
  `host.status()`, `haptic('tap')`.
- `ctx.register({ id, area, order?, render?, data? })` — contribute UI.
  Key areas: `'statusBar.right'`/`'statusBar.left'` (chips),
  `'panes'` (layout zones — set `title` and
  `data: { placement, dock?, width?, height? }`; the pane auto-joins a
  matching zone), `PALETTE_AREA` (⌘K commands), `KEYBINDS_AREA` (rebindable
  actions).
- Pane placement: `placement: 'left'|'right'|'bottom'|'main'` is the
  semantic role — the pane stacks (tabs) with existing panes of that role.
  To land on a specific EDGE instead, add `dock: { pane, pos }` — the same
  gesture as dragging onto a pane's drop chip. `pane` is any pane id
  (`workspace` is the main thread; also `sessions`, `terminal`, `files`,
  `review`, `logs`), `pos` is `'top'|'bottom'|'left'|'right'|'center'`.
  E.g. "below the conversation" = `dock: { pane: 'workspace', pos: 'bottom' }`
  — declare a `height` (e.g. `'200px'`) so it doesn't take half the zone.
- Full PAGES: register `area: ROUTES_AREA` with `data: { path: '/my-page' }`
  and a `render` — the page mounts in the workspace (main) pane like any
  built-in view. Make it reachable with a sidebar nav row:
  `ctx.register({ id: 'nav', area: SIDEBAR_NAV_AREA, data: { path: '/my-page', label: 'My Page', codicon: 'project' } })`
  (renders below Artifacts, lights up at the route) — and/or a
  `PALETTE_AREA` command calling `host.navigate('/my-page')`.
- `ctx.storage.get/set/remove` — persistence namespaced to your plugin.
- `ctx.os` — the curated OS door, attributed to your plugin:
  `ctx.os.notify({ title, body?, silent? })` posts a native OS notification.
  Fires only while the user is away from Hermes (use `host.notify` for the
  in-app toast); gated by Settings ▸ Notifications ▸ "Plugin notifications"
  and throttled per plugin — reserve it for genuinely notable events.
  `ctx.os.openExternal(url)`, `ctx.os.revealPath(path)`, and
  `ctx.os.writeClipboard(text)` resolve `false` (never throw) when the
  capability isn't available.
- `ctx.i18n.register({ en, ja, ... })` — ship your OWN locale bundles, scoped
  to your plugin (never edit core `en.ts`). Values are literal strings or
  interpolator functions; nested trees are addressed by dot-path. Read them
  reactively in components with `usePluginI18n(id)` returning `t('key', ...args)`
  (re-renders on a locale switch), or via `ctx.i18n.t` in handlers/stores.
  Resolution follows the app's active locale, then your `en`, then the raw key.
- Data: `useQuery`/`useMutation`/`useQueryClient`/`queryClient` (the app's ONE
  React Query client — cache, dedupe, `refetchInterval`, invalidate like core;
  never hand-roll a poll loop), plus `atom`/`computed` for plugin-local state.
- Backend: if the plugin ships a Python `plugin_api.py` (under
  `~/.hermes/plugins/<id>/dashboard/`, manifest `"api": "plugin_api.py"`), reach
  it with `ctx.rest('/path', { method?, body?, timeoutMs? })` and its live twin
  `ctx.socket('/events', onMessage)` — both scoped to `/api/plugins/<id>` by
  construction (traversal rejected). `ctx.socket` is a **no-op on OAuth
  remotes**, so always keep a polling fallback. The Python backend is imported
  only when the plugin is in `plugins.enabled` in `config.yaml` (separate from
  the in-app enable toggle). For gateway-wide data use `host.request` /
  `host.onEvent` instead.
- `Contribute` (mount-scoped): render `jsx(Contribute, { area, id, children })`
  inside a component so page-owned chrome (e.g. a titlebar control in
  `TITLEBAR_AREAS.center`) leaves when the page unmounts — `ctx.register` is for
  permanent contributions.
- `defaultEnabled: false` on the default export ships an opt-in plugin: it
  inventories in Settings → Plugins, off until the user flips it on.
- Users manage plugins in Settings → Plugins (enable/disable live, reveal
  folder). A disabled plugin stays disabled across restarts — don't fight
  it; the user turned you off.
- UI: the app's design language, importable directly — `Button`, `Input`,
  `Textarea`, `Select*`, `Switch`, `Checkbox`, `SegmentedControl`, `Tabs*`,
  `Dialog*`, `ConfirmDialog`, `DropdownMenu*`, `ContextMenu*`, `Popover*`,
  `Tip`/`Tooltip*`, `Badge`, `Kbd`/`KbdGroup`, `SearchField`, `ScrollArea`,
  `Separator`, `Skeleton`, `GlyphSpinner`, `EmptyState`, `ErrorState`,
  `CopyButton`, `StatusDot`, `LogView`, `Codicon`, `DecodeText`, plus `cn`
  and `icons.*`. Prefer these over hand-rolled elements so the plugin looks
  native; style with theme vars, never hardcoded colors.

## Compatibility rules

To keep a desktop plugin working across Hermes updates unless the SDK has a
real breaking change:

- Use only the **public SDK surface**: `@hermes/plugin-sdk`, `react`, and
  `react/jsx-runtime`. Never import from `apps/desktop/src/...` or other app
  internals.
- Treat `host.request(...)`, `host.onEvent(...)`, `ctx.register(...)`,
  `ctx.storage`, and `ctx.i18n` as the stable contract. Prefer them over
  reading internal DOM shape or private renderer state.
- Keep the plugin **id stable** and make the folder name match it. Storage,
  backend routing, and enable/disable state key off that id.
- If you add a paired Python backend, keep it optional and narrow: use
  `host.request(...)` for normal gateway/session/config operations, and use
  `ctx.rest(...)` only for plugin-specific behavior.
- Always tolerate missing optional capabilities. `ctx.socket(...)` is a no-op
  on OAuth remotes, `ctx.os.*` may return `false`, and host state can be empty
  during startup or reconnects.
- Accept host evolution: avoid assuming fixed pane layouts, exact DOM
  structure, or undocumented event payloads. Register contributions through
  area ids instead of patching core UI directly.

## Procedure

1. Pick a short kebab-case `id`; the folder name must match.
2. Start from `templates/plugin.js`; keep the default export shape
   (`{ id, name, register(ctx) }`).
3. For a pane, register `area: 'panes'` with a `placement` hint and a
   `render` returning your component — the app places it into a sensible
   zone automatically; the user can drag it anywhere afterwards.
4. Fetch data with `host.request` and/or subscribe with `host.onEvent`;
   never poll faster than a few seconds.
5. If you created a paired backend under `$HERMES_HOME/plugins/<id>/`, enable
   it with `hermes plugins enable <id>` before testing.
6. Write the file with your file tools, then ask the user to run
   **Reload desktop plugins** from ⌘K.

## Pitfalls

- NEVER hardcode colors or backgrounds (`#000`, `black`, `rgb(...)`). Panes
  already sit on the app's editor background — leave the background alone
  and use theme variables for everything else: `var(--ui-text-secondary)`,
  `var(--ui-text-quaternary)`, `var(--ui-stroke-secondary)`,
  `var(--ui-accent)`. For canvas drawing, resolve them once with
  `getComputedStyle(canvas).getPropertyValue('--ui-accent')`.
- Reference only what you imported — a component you forgot to import
  (e.g. `StatusDot`) is a ReferenceError at render. Double-check every
  identifier in your `jsx()` calls appears in the import line.
- Canvas panes MUST track their container with a `ResizeObserver` and
  re-size the canvas (width/height attributes, not just CSS) — panes resize
  constantly (sash drags, layout switches); a mount-time-only size leaves
  blank space or blurry scaling.
- JSX syntax will not parse — the file loads uncompiled. Use
  `jsx('div', { children: ... })` from `react/jsx-runtime`.
- Do not import anything except `@hermes/plugin-sdk`, `react`, and
  `react/jsx-runtime`; other specifiers fail to resolve.
- Handlers must read state imperatively (`$atom.get()`), never from render
  closures — rapid events will otherwise see stale values.
- Keep components small; subscribe (`useValue`) only in the leaf that
  renders the value.

## Verification

- The plugin's UI appears after **Reload desktop plugins**.
- No error toast ("Plugin <name> failed to load") appears; if it does, the
  message names the failure — fix and reload.
- For panes: the new zone is visible and draggable like any core pane.
