# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project intent — read this first

Per the README, this is the starter project for the [Code with Mosh Claude Code course](https://codewithmosh.com/p/claude-code). It **intentionally** ships with a bug, poor UI, and messy code, which are fixed progressively as course exercises.

Do not spontaneously fix the remaining rough edges below. They are the teaching material. Fix them only when explicitly asked.

### The amount bug — already fixed

The original starter stored `transaction.amount` as a **string**, so the totals' `reduce((sum, t) => sum + t.amount, 0)` concatenated instead of adding — Income rendered as `$05000` and Balance as `$-120015080095649500`.

This is fixed by keeping `amount` numeric at both points where a value enters state: the seed data holds number literals, and `handleSubmit` runs `parseFloat` behind a `Number.isFinite` guard before storing. The two `reduce` calls were deliberately left untouched — they were always correct arithmetic, and patching them instead would have left strings in state for the next consumer to trip over.

**Invariant to preserve:** `amount` must stay a `number` in state. `<input type="number">` yields a string, so any new write path into `transactions` has to parse first.

### Other deliberate rough edges

- `.delete-btn` is styled in `src/App.css` but no delete button exists in the JSX. The transactions table carries a matching empty trailing `<th>`/`<td>` pair — a placeholder for the per-row delete feature.
- Everything (state, derived totals, filtering, form handling, and all markup) lives in one ~155-line `App` component. There is no component decomposition yet.
- There is no currency formatting — amounts interpolate raw, so `10.50` renders as `$10.5`, and float addition can surface artifacts like `$0.30000000000000004`. A `.toFixed(2)` on the three summary cards and the row amounts would settle it.

## Commands

```bash
npm install
npm run dev      # Vite dev server → http://localhost:5173
npm run build    # production build → dist/
npm run preview  # serve the built dist/
npm run lint     # ESLint over **/*.{js,jsx}
```

**There is no test infrastructure** — no test script, no runner, no test files. If asked to add tests, Vitest is the natural fit for a Vite project, but it requires installing and wiring it from scratch; there is no existing convention to follow.

## Local environment (non-obvious — this machine)

Tooling is **only** reachable through WSL; it does not exist on the Windows side.

- Windows has **no `node`, `npm`, or `git`** on PATH at all.
- WSL's apt `nodejs` package is **broken**: dpkg reports 12.22.9 installed, but `/usr/bin/node` is missing and only the dangling `nodejs -> node` symlink remains.
- The usable Node lives in **asdf** at `~/.asdf/installs/nodejs/` (20.17.0 and 18.15.0), but **no global asdf version is set**, so the shims fail with `No version is set for command node`.

Run commands by putting the install dir on PATH directly, which avoids mutating the user's asdf config:

```bash
export PATH=$HOME/.asdf/installs/nodejs/20.17.0/bin:$PATH
cd /mnt/c/Users/299797/AI_Projects/expense-tracker-starter
npm run dev -- --host 0.0.0.0
```

Caveats when working this way:

- **Node 20.17.0 is below Vite 7's declared `engines.node` (`^20.19.0 || >=22.12.0`).** Install and `npm run dev` both emit loud version warnings but do work. `asdf install nodejs 22.12.0` clears them.
- **Hot reload will likely not fire.** The source sits on `/mnt/c` while Vite runs in WSL, and inotify does not propagate over the 9p mount. Use `server: { watch: { usePolling: true } }` in `vite.config.js`, or clone into the WSL filesystem.
- Headless **Chrome is blocked by a machine-level policy error** on this box; headless **Edge** works for screenshotting (`msedge.exe --headless=new --screenshot=... --virtual-time-budget=8000`). WSL→Windows localhost forwarding works, so `http://localhost:5173` is reachable from Windows browsers.

## Architecture

Vite 7 + React 19, client-only. `index.html` → `src/main.jsx` (mounts `<App>` in `StrictMode`) → `src/App.jsx`.

- **No backend, no persistence, no routing, no state library.** All transactions live in a single `useState` array in `App`, seeded with eight hardcoded entries. State resets on every page reload.
- **Totals and filtering are derived inline on each render** — recomputed from `transactions` rather than stored, so they stay consistent automatically. Filtering chains two independent predicates (`filterType`, `filterCategory`), each with an `"all"` sentinel that skips the filter.
- **`categories` is a single hardcoded array** driving both the add-transaction `<select>` and the filter `<select>`. Adding a category means touching only that array.
- **A transaction's `type`** (`"income"` / `"expense"`) drives both the totals split and the row's sign and CSS class. Note the seed data contains a mislabeled row: "Freelance Work" is categorized `salary` but typed `expense`.
- **Styling is plain CSS**, no preprocessor or CSS modules: `src/index.css` (minimal global reset) and `src/App.css` (plain class selectors plus bare `form` / `table` / `th` element selectors — these are global, so element-level changes affect everything).

## Lint configuration

Flat config in `eslint.config.js`: `js.configs.recommended` + `react-hooks` (flat recommended) + `react-refresh` (vite preset), ignoring `dist`. One override worth knowing: `no-unused-vars` is an **error**, with `varsIgnorePattern: '^[A-Z_]'` so unused constants named in caps are allowed.
