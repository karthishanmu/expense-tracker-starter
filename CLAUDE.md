# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project intent — read this first

Per the README, this is the starter project for the [Code with Mosh Claude Code course](https://codewithmosh.com/p/claude-code). It **intentionally** ships with a bug, poor UI, and messy code, which are fixed progressively as course exercises.

Do not spontaneously fix the **intentional** rough edges listed below — they are the teaching material, so fix them only when explicitly asked. The **latent issues** in the second list are not course material; they are genuine defects and are fair game to raise.

### The amount bug — already fixed

The original starter stored `transaction.amount` as a **string**, so the totals' `reduce((sum, t) => sum + t.amount, 0)` concatenated instead of adding — Income rendered as `$05000` and Balance as `$-120015080095649500`.

This is fixed by keeping `amount` numeric at both points where a value enters state: the seed data in `App.jsx` holds number literals, and `TransactionForm`'s `handleSubmit` runs `parseFloat` behind a `Number.isFinite` guard before calling `onAdd`. The two `reduce` calls in `Summary` were deliberately left untouched — they were always correct arithmetic, and patching them instead would have left strings in state for the next consumer to trip over.

**Invariant to preserve:** `amount` must stay a `number` in state. `<input type="number">` yields a string, so any new write path into `transactions` has to parse first.

### Intentional rough edges — leave alone unless asked

- There is no currency formatting — amounts interpolate raw, so `10.50` renders as `$10.5`, and float addition can surface artifacts like `$0.30000000000000004`. A `.toFixed(2)` on the three summary cards and the row amounts would settle it.
- The "Freelance Work" seed row is `type: "expense"` with `category: "salary"`, so the Income + Salary filter pair matches nothing. Probably unintended in the original data, but it is seed data, not logic.

### Latent issues — real defects, not course material

- **`id: Date.now()` can collide** if two transactions are ever added within the same millisecond, which would make one delete click remove two rows and duplicate React keys. Not reachable through the UI today — a human form submit gates every add, and StrictMode double-invokes render, not event handlers — but any bulk import or programmatic add would expose it. One-line fix: `crypto.randomUUID()`. Ids are only used for `key` and `===`, never sorted or used in arithmetic, so the type change is safe.
- The table header cells lack `scope="col"`.
- Delete is immediate and there is no undo. A page reload resurrects the eight seed rows but not anything the user added, so "reload to recover" is misleading advice. Worth revisiting if persistence is ever added.

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

Vite 7 + React 19, client-only. `index.html` → `src/main.jsx` (mounts `<App>` in `StrictMode`) → `src/App.jsx`. Components sit flat in `src/`, not in a `components/` subdirectory.

`App` is now a thin shell: it owns the `transactions` array and an add handler, and composes three children. Each child holds whatever state only it cares about, so nothing is prop-drilled and there are no setter props.

| File | Owns | Props |
|---|---|---|
| `src/App.jsx` | the `transactions` array; assigns `id` and `date` on add, removes by `id` on delete | — |
| `src/Summary.jsx` | nothing; derives the three totals | `transactions` |
| `src/TransactionForm.jsx` | the four form fields, validation, reset | `onAdd` |
| `src/TransactionList.jsx` | `filterType` / `filterCategory`; derives the filtered rows | `transactions`, `onDelete` |

Two boundaries worth respecting:

- **`TransactionForm` reports only user-entered fields** — `onAdd({ description, amount, type, category })`. `App` supplies `id` and `date`, because those are system-generated rather than typed. The form parses and validates before calling `onAdd`, so `App` trusts what it receives.
- **`Summary` gets the full `transactions`, never the filtered list.** `TransactionList` filters internally and does not expose the result, which is what keeps the totals reflecting all transactions while the table is filtered. Do not lift the filter state into `App` without accounting for this.
- **Delete removes by `id`, never by index**, and the `window.confirm` prompt lives in `TransactionList`, not `App`. Index-based removal would delete the wrong row whenever a filter is active, since a row's position in `filteredTransactions` does not match its position in `transactions`. Keeping the dialog in the child mirrors `TransactionForm` validating before it calls `onAdd`, leaving `App` purely about data.
- **Do not derive the category filter's `<option>` list from the transaction data.** It reads from the static `categories` array, which is why deleting the last `food` row leaves `filterCategory === "food"` pointing at a still-valid option that simply matches zero rows. Deriving the options would produce the classic stale-filter bug where the select's value matches no option and the control renders blank.

Component styles all stay in `App.css` rather than per-component files, because `.income-amount` and `.expense-amount` are shared between the summary cards and the table rows — splitting them would duplicate those rules. `App.css` is imported once by `App`, and CSS is global, so the classes resolve in every child.

- **No backend, no persistence, no routing, no state library.** All transactions live in a single `useState` array in `App`, seeded with eight hardcoded entries. State resets on every page reload.
- **Totals and filtering are derived inline on each render** — recomputed rather than stored, so they stay consistent automatically. Filtering chains two independent predicates, each with an `"all"` sentinel that skips the filter.
- **`src/categories.js` exports the one hardcoded category array**, imported by both the form's `<select>` and the list's category filter. It lives in its own module rather than being passed down, since `App` itself has no use for it. Adding a category means touching only that file.
- **A transaction's `type`** (`"income"` / `"expense"`) drives both the totals split and the row's sign and CSS class. Note the seed data contains a mislabeled row: "Freelance Work" is categorized `salary` but typed `expense`.
- **Styling is plain CSS**, no preprocessor or CSS modules: `src/index.css` (minimal global reset) and `src/App.css` (plain class selectors plus bare `form` / `table` / `th` element selectors — these are global, so element-level changes affect everything).

## Lint configuration

Flat config in `eslint.config.js`: `js.configs.recommended` + `react-hooks` (flat recommended) + `react-refresh` (vite preset), ignoring `dist`. One override worth knowing: `no-unused-vars` is an **error**, with `varsIgnorePattern: '^[A-Z_]'` so unused constants named in caps are allowed.
