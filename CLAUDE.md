# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev      # Start Vite dev server (http://localhost:5173)
npm run build    # Build for production (outputs to dist/)
```

No test runner or linter is configured.

## Architecture

Single-page React app that presents AI-powered stock recommendations for OMXS30 (Swedish) and DJIA (US) stocks, with portfolio tracking persisted to `localStorage`.

### Key files

- `main.jsx` — React entry point; also installs a `window.storage` shim over `localStorage`
- `muppet-stock-screener.jsx` — The entire application in one file (~1 200 lines): all state, logic, and components

### Data flow

1. **Screening** — User triggers scan → `runScreener()` calls the Anthropic API (`claude-sonnet-4-20250514`) with the stock universe + current holdings + available capital. The model uses the `web_search` tool to fetch live prices and news, then returns a JSON array of decisions (`BUY`, `BUY MORE`, `HOLD`, `SELL`, `NO BUY`) with price targets and reasoning.

2. **Trade execution** — Buy/Sell modals update `holdings` state and recalculate `capital` (available = initial − invested − fees). Closed trades append to a `history` array with realized P&L.

3. **Persistence** — After every mutation, holdings, history, and capital are written to `localStorage` via `window.storage`.

### Key state shapes

```js
// holdings  (keyed by ticker symbol)
{ "ERIC-B.ST": { symbol, name, shares, buyPrice, currency, costSEK, date, market } }

// history entries
{ symbol, name, date, shares, buyPrice, sellPrice, currency, pnlSEK, pnlPct }

// capital
{ total: Number, available: Number }
```

### Stock universe

Two hard-coded arrays at the top of `muppet-stock-screener.jsx`:
- `OMXS30_BASE` — 30 Swedish large-cap tickers (`.ST` suffix)
- `DJIA_BASE` — 30 US blue-chip tickers

To add or remove stocks, edit those arrays directly.

### Anthropic API call

Located in `runScreener()`. Requires a valid Anthropic API key — the key is read from wherever it is injected at runtime (environment variable or request header). Max position size is capped at 25 % of available capital per the system prompt.
