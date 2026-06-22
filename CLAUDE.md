# CLAUDE.md

Guidance for Claude Code (or any AI coding agent) working in this repository.

## What this project is

A single-page React app, distributed as one self-contained `index.html`, that compares cloud AI model APIs against self-hostable local models on a shared cost basis ($/M tokens). There is **no build step, no package.json, no node_modules**. React and Babel Standalone load from CDN `<script>` tags in `index.html`, and Babel transpiles the inline JSX in the browser at load time.

Do not introduce a build step (Vite, webpack, Next.js, etc.) unless explicitly asked. The zero-tooling property is a deliberate feature — anyone should be able to download this and double-click it.

## File layout

```
index.html   — the actual app. Open this in a browser to run it.
App.jsx      — same component code, as plain JSX, kept in sync with index.html for easier editing/diffing.
README.md
CLAUDE.md    — this file
```

`index.html` = `App.jsx` body, wrapped in:
```html
<script src="https://unpkg.com/react@18/umd/react.development.js"></script>
<script src="https://unpkg.com/react-dom@18/umd/react-dom.development.js"></script>
<script src="https://unpkg.com/@babel/standalone@7/babel.min.js"></script>
<div id="root"></div>
<script type="text/babel">
const { useState, useContext, createContext, useMemo, useEffect, useRef } = React;
/* App.jsx body goes here, minus the `import` lines and minus `export default` on App */
ReactDOM.createRoot(document.getElementById('root')).render(React.createElement(App));
</script>
```

**When editing app logic:** edit `App.jsx`, then regenerate `index.html` by stripping the `import` lines, removing `export default` from `function App()`, and re-wrapping per the template above. Keep both files in sync — don't let them drift.

## Architecture

Single component tree, no router, no external state library:

```
App (root)
├── owns: dark/light theme, cloudModels, userScores (legacy, unused), userTps, benchmarkModel (modal state)
├── ModalOverlay (renders at App's own return, NOT inside CostIndex — see "Modal placement" below)
│   └── TpsPanel (the TPS measurement tool)
└── CostIndex (the main table + controls)
    ├── owns: hardware selection, custom prices, hours/day, ratio, filters, sort
    ├── localRows = useMemo(...) — computes local model rows from LOCAL_MODELS + current hw/hrs/price/userTps
    ├── processed = useMemo(...) — merges cloudModels + localRows, computes blended cost
    └── filtered/rows = useMemo(...) — applies search/tier/vendor filters and sort, groups by tier
```

### Modal placement — read this before touching modals

`position: fixed` modals **must render at the `App` root's return**, not inside `CostIndex`'s JSX tree. `CostIndex`'s outer `<div>` does not itself break `position:fixed`, but if you ever add a `transform`, `filter`, `will-change`, or `overflow: auto/scroll` to an ancestor of a modal, the modal will get clipped or mispositioned. We've hit this bug twice already. If a modal stops opening correctly or appears in the wrong place, check this first.

Do **not** add `ReactDOM.createPortal` — it requires a separate `react-dom` import that isn't available as a standalone ES module in the Babel/CDN setup this project uses. The `ModalOverlay` component (plain fixed-position div with `zIndex: 99999`) is the correct pattern here; don't "fix" it by reaching for a portal.

### Data model

Two flat arrays hold all model data. Every field on every object matters for the table — there is no schema validation, so a typo'd key just silently shows as blank/`—` in the UI.

```js
// CLOUD_MODELS — one entry per (model, variant) pair
{ family, model, variant, score, ctx, params, arch, input, output, tps, reasoning, note }

// LOCAL_MODELS — one entry per locally-runnable model
{ family, model, quant, ctx, score, params, arch, tps: { zbook, macbook }, reasoning, hfUrl, note }
```

`family` on local models is the **actual model vendor** (Alibaba, Meta, Microsoft, ...), not the literal string `"Local"` — the LOCAL badge is a separate visual tag layered on top in the `FamilyChip` component. Don't regress this; an earlier version used `family: "Local"` for everything, which broke vendor filtering.

`tps: null` (cloud) or `tps: { zbook: null, macbook: null }` (local) means "doesn't fit on this hardware" or "no public TPS data" — the cost-formatting helper (`fmtCost`) and the table deliberately render this as `—`, not `$0.00`. If you see `$0.000` for a model in the table or in the "Sample Costs" panel, the bug is almost certainly that something filtered in a `tps: null` row instead of excluding it — check `.filter(m => m.tps && ...)` is present wherever a small "top N" slice is taken from `localRows`.

### Cost formulas — don't reinvent these

```js
// Local hardware amortization
hourlyRate = hardwarePrice / (years * 365 * hoursPerDay)
costPer1M  = (hourlyRate / (tps * 3600)) * 1_000_000

// Cloud blended cost (ratio = input:output token ratio, default 25:1 for agentic work)
blended = (input * ratio + output * 1) / (ratio + 1)
```

Both live in `localCostPerMTok()` / `perHr()` and `calcBlended()` near the top of the file. If you change the formula, update the inline comment above it — the comment is the only documentation of *why* the constants are what they are (e.g. 25:1 ratio rationale, 3-year amortization assumption).

### Theme system

`LIGHT` and `DARK` are plain objects with named color tokens (`text`, `textMuted`, `border`, `elite`, `high`, `mid`, ...), provided via `ThemeCtx` (a plain `createContext()`, not a CSS variable system). Every component reads `const { t } = useTheme()` and uses `t.whatever` for color — never hardcode a hex value in a component; add a new token to both `LIGHT` and `DARK` instead, so the dark/light toggle keeps working everywhere.

## Known footguns (please don't reintroduce these)

1. **`import ReactDOM from "react-dom"`** — does not work in this Babel/CDN setup. `ReactDOM` is already a global from the CDN `<script>` tag. Never add this import to `App.jsx`.
2. **`<>...</>` fragment shorthand inside `.map()`** — needs a `key`, which the shorthand syntax can't take. Use `<React.Fragment key={...}>` explicitly inside any `.map()` callback.
3. **Unescaped `&` in JSX text nodes** — e.g. literally typing `Cost & Pricing` between JSX tags breaks the parser. Use `&amp;` or rephrase. This is *not* an issue inside `{ }` JS expressions or `.js`/`.jsx` string literals — only in literal JSX text between tags.
4. **`window.open()` instead of `<a target="_blank">`** — gets blocked by Chrome's popup blocker when called from inside a sandboxed iframe (e.g. an embedded preview). Always use a real `<a href="..." target="_blank" rel="noopener noreferrer">` for "open in new tab" links; never intercept the click with `e.preventDefault()` + `window.open()`.
5. **`minHeight: "100vh"` on a component that's itself scrollable** — in iframe/sandboxed contexts `100vh` can refer to the outer viewport, not the available space, and combined with internal padding this creates a broken nested scroll container that swallows clicks. Avoid `100vh` on inner content; let height be intrinsic.
6. **Missing `tps` filter before slicing "top N" lists** — see "Data model" above. Always filter `m.tps && m.blended > 0` (or the hardware-specific equivalent) before taking a small preview slice of `localRows`, or you'll surface unrunnable models with `$0.00` cost.

## Conventions

- No semicolons are *not* enforced either way — match the surrounding style in whatever section you're editing (the file mixes both because it's been patched incrementally; new code should prefer semicolons for clarity).
- Inline styles only — there is no CSS file. Keep using the `t.token` pattern from `useTheme()` for anything color-related.
- Keep `index.html` and `App.jsx` byte-for-byte identical in their JS body (modulo the import/export lines and CDN wrapper). If you only edit one, regenerate the other before committing.
- When adding a model to either array, also check whether it needs an entry in `MODEL_CARD_URL` (model name → official page or HuggingFace card) and, for cloud models, that its `family` has a pricing URL in `PRICING_URL`.

## Testing changes

There's no test suite. The practical check is: open `index.html` in a real browser (not just preview it), and manually verify:
1. Light/dark toggle still works and all text stays readable in both
2. Hardware selector + price inputs + hours slider all update the table live
3. Clicking a model name opens its card in a new tab; clicking a cost cell opens the pricing page
4. The TPS measurement modal opens, the stopwatch works, and saving a TPS value updates that model's row
5. Sorting by each column header works and the tier grouping still makes sense after a sort
