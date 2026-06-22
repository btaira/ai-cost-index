# AI Model Intelligence & Cost Index

A tiny, zero-build React app that compares cloud model APIs and self-hostable local models on intelligence score, throughput (TPS), context window, architecture, and unified cost ($ per 1M tokens).

![tiers: 5](https://img.shields.io/badge/tiers-5-blueviolet) ![models: 100+](https://img.shields.io/badge/models-100%2B-orange) ![zero-build](https://img.shields.io/badge/build%20step-none-green)

## Overview

This project shows cloud and local models side-by-side using the same cost units so you can answer: "Is it cheaper to run this locally, and by how much?" Local costs are calculated from editable hardware prices, runtime hours, and measured or community TPS benchmarks — not guesses.

## Quick links

- **Open the app**: `index.html` (double-click in Explorer or run `start index.html` on Windows)
- **Edit source**: `App.jsx` (same code as `index.html`, without the CDN wrapper)

## Quick start

Clone and open the single HTML file. No dependencies, no build step.

```powershell
git clone <this-repo>
cd ai-cost-index
start index.html   # Windows (double-click also works)
# on macOS: open index.html
```

## How it works

- **Local cost** is amortized from hardware price and measured throughput:

```text
hourly_rate = hardware_price ÷ (years × 365 × hours_per_day)
cost_per_1M = (hourly_rate ÷ (tokens_per_second × 3600)) × 1_000_000
```

- **Cloud blended cost** mixes input/output prices with an adjustable input:output ratio (default 25:1):

```text
blended_cost = (input_price × ratio + output_price × 1) ÷ (ratio + 1)
```

## Key features

- 100+ models: cloud APIs (Anthropic, OpenAI, Google, Mistral, Meta, etc.) and ~50 self-hostable models
- Live hardware amortization (editable prices, hours/day)
- Built-in TPS measurement tool (stopwatch + timed prompts) so you can measure your own model
- Clickable model cards and pricing links
- Light/dark theme, sortable columns, filters, and tier grouping
- Zero build step: runs from `index.html` using React + Babel from CDN

## Data model & editing

All model data is in two arrays near the top of the app source:

- **`CLOUD_MODELS`**: objects for cloud models with fields `family`, `model`, `score`, `input`, `output`, `tps`, `ctx`, `params`, `arch`, etc.
- **`LOCAL_MODELS`**: objects for local models with fields `family`, `model`, `quant`, `score`, `tps: { zbook, macbook }`, `ctx`, `params`, `arch`, `hfUrl`, etc.

To add or edit a model, copy an object literal in the appropriate array and update its fields — the UI reads these arrays directly.

## Keeping `index.html` in sync with `App.jsx`

`index.html` is `App.jsx` wrapped with CDN script tags and a bootstrapping block. When editing `App.jsx`:

1. Remove `import` statements from the top
2. Remove `export default App` from the function signature
3. Wrap the component code with the CDN template (React, React-DOM, Babel script tags, plus the `ReactDOM.createRoot` call)

For the full template and additional details, see `CLAUDE.md`.

## Important notes

- Do not add `import ReactDOM from "react-dom"` — `ReactDOM` is provided by the CDN in this zero-build setup.
- Keep `ModalOverlay` rendered at the app root: fixed-position modals must not be inside transformed/scrollable ancestors.
- Filter out `tps: null` rows before taking top-N slices to avoid showing `$0.00` costs for unrunnable models.
- See `CLAUDE.md` for a complete list of development gotchas and architectural details.

## Data sources

- **Cloud benchmark scores**: Artificial Analysis Intelligence Index v4.0
- **Cloud pricing**: official vendor pages (linked from the UI)
- **Local model list/scores**: Onyx Self-Hosted LLM Leaderboard
- **Local TPS**: community benchmarks (llama.cpp, MLX) — replace with your own measurements for accuracy

## Contributing

Small edits and model data updates welcome. For anything bigger (new UI, build step), open an issue describing the change first.

- Edit model data directly in `App.jsx` and submit a PR.
- If adding a model, consider adding a link in `MODEL_CARD_URL` for quick reference and search.

## License & disclaimer

All pricing and benchmark figures are aggregated from public sources and provided as-is for informational use only. Verify vendor pricing before making purchasing decisions. Code is MIT-licensed — adapt freely.
