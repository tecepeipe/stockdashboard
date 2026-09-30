# Stock Dashboard — Project Skills

## Project identity

- Repository: `tecepeipe/stockdashboard`
- Primary application: `index.html`
- Current application version: **1.10.0**
- Deployment: static GitHub Pages
- Live site: `https://tecepeipe.github.io/stockdashboard/`
- Language: HTML/CSS/JavaScript
- Architecture: intentionally single-file client-side application
- Main purpose: interactive stock-market visualisation for price action, technical indicators, candlestick reversal patterns, support/resistance, historical pattern analysis, market data, watchlists and browser-side alerts.
- Supported UI languages: English, Brazilian Portuguese, Spanish and French.
- Chart supports click-and-drag range selection with absolute and percentage change.
- Mock/demo data is a permanent product requirement.

## Core principles

1. **Preserve demo mode.**
   - The application must work without credentials.
   - Mock data must remain clearly distinguishable from live data.
2. **Validate data, not just HTTP status.**
   - HTTP 200 is not sufficient.
   - Validate body shape, symbol, timestamps, OHLC values and usable bar count.
   - Handle null, empty, malformed and HTTP 204 responses explicitly.
3. **Keep calculations precise.**
   - Round only for presentation.
   - Preserve source OHLC/indicator precision internally.
4. **Prefer defensive, observable code.**
   - Guard against NaN, Infinity, nulls, zero ranges and malformed storage.
   - A provider failure must not cause a blank React/Babel page.
5. **Preserve provider independence.**
   - Provider-specific response formats stop at the adapter/normalization boundary.
   - Chart and analysis code consume the normalized model.

## Architecture rules

The project must remain a single HTML file unless explicitly requested otherwise.

Do not introduce:
- Vite.
- npm/package-manager build requirements.
- Separate runtime modules.
- A framework migration.

Logical modularity inside `index.html` is encouraged. Keep these concerns visibly separated:
- constants/configuration
- mock data
- storage
- provider adapters (module scope)
- response validation/normalization
- timeframe/caching
- indicators
- candlestick detection
- Pattern Lab
- support/resistance
- chart model/rendering
- range selection
- translations (React `LanguageContext` + `t()`)
- alerts/watchlist
- `useMarketDataPipeline` hook
- section components (HeaderBar, StatusBar, ApiKeyPanel, WatchlistSidebar, ChartSection, RightRail)
- Dashboard assembly
- regression/indicator/candle-pattern test suites (`window.__APP_DEBUG__`)

## Current provider architecture

Supported live providers:
- Twelve Data.
- Alpaca.
- Alpha Vantage.

Current precedence:
1. Alpaca when both key ID and secret are present.
2. Alpha Vantage when its key is present.
3. Twelve Data when its key is present.
4. DEMO when no live credentials are present.

Keep this precedence stable unless the UI and documentation are intentionally updated.

Preferred flow:

`provider request -> provider adapter -> validation -> normalized market data -> timeframe selection -> indicators/patterns -> UI`

### Normalized market model

Use a common model similar to:

```js
{
  symbol,
  name,
  exchange,
  currency,
  provider,
  modelVersion,
  bars: [
    { time, open, high, low, close, volume }
  ]
}
```

Provider-specific fields such as Alpaca `t/o/h/l/c/v` or Twelve Data `datetime/open/high/low/close/volume` must be normalized before chart/analysis logic.

### Alpaca

Important implementation lessons:
- A valid symbol and usable historical data are separate concepts.
- `bars:null` does not by itself prove an invalid ticker.
- HTTP 204 means no content and must not update chart state with fake/empty success.
- Validate the response body before changing chart state.
- Log useful diagnostics while debugging: provider, symbol, timeframe, status, response shape and normalized-bar count.
- The active chart must be updated explicitly after valid bars are normalized and processed.
- Keep Alpaca OHLC retrieval separate from quote retrieval.
- Use explicit date bounds and the configured timeframe mapping.
- Current timeframe mapping:
  - 1D -> 5Min
  - 1W -> 15Min
  - 1M -> 30Min
  - 1Y -> 1Day
- Alpaca currently uses the IEX feed in the browser implementation.
- Metadata lookup uses authenticated `/v2/assets/{symbol}` when available.
- Cache successful asset metadata during the session.
- Metadata lookup must never block ticker usability; fall back to the ticker when necessary.
- Do not use asset lookup as a substitute for market-data availability testing.

### Twelve Data

Keep the existing Twelve Data adapter isolated from UI code.

Normalize returned `values` into the common bar model before timeframe filtering and indicator processing.

### Alpha Vantage

Current mapping:
- 1D -> `TIME_SERIES_INTRADAY` 5-minute.
- 1W -> `TIME_SERIES_INTRADAY` 15-minute.
- 1M -> `TIME_SERIES_INTRADAY` 30-minute.
- 1Y -> `TIME_SERIES_DAILY` full history.

Current implementation also uses Alpha Vantage quote/profile/fundamental endpoints where available. Keep technical indicator calculations local so all providers use the same indicator algorithms.

Alpha Vantage intraday availability depends on provider entitlement; do not promise equivalent access on all plans.

## Timeframes and caching

Current timeframe configuration:
- 1D: 5-minute, 10-day lookback, max 100 visible candles, 5-minute cache freshness.
- 1W: 15-minute, 21-day lookback, max 160, 15-minute cache freshness.
- 1M: 30-minute, 70-day lookback, max 320, 30-minute cache freshness.
- 1Y: daily, 430-day lookback, max 270, 2-hour cache freshness.

Canonical timeframe filtering:
- Sort bars chronologically.
- 1D uses the latest available trading session.
- 1W uses the latest five trading sessions.
- 1M uses the rolling UTC month window.
- 1Y uses the rolling UTC year window.

Cache keys are provider/symbol/timeframe specific and versioned.

Important refresh rule:
- Cached historical data may be usable even when the active quote is absent.
- When switching to a cached symbol, the active quote must still be obtained when no quote snapshot exists.
- Force refresh must bypass relevant caches.

## Symbol metadata

Symbol metadata has several sources:
- built-in `TICKER_DICTIONARY`
- `resolvedSymbols`
- provider-specific live metadata, especially Alpaca assets.

Do not assume the local dictionary contains every live ticker.

Current Alpaca behaviour:
- exact-symbol metadata lookup
- company name/exchange resolution
- session cache
- ticker fallback if lookup fails

Future metadata work should centralize these sources rather than creating additional competing lookup paths.

## Technical-analysis engine

Technical calculations are separated into reusable helpers including:
- average
- SMA
- EMA
- RSI
- true range
- Bollinger Bands
- technical-indicator orchestration

Current indicator families:
- EMA 9/21/50
- SMA 20/50
- MACD 12/26/9
- RSI 14
- ATR
- Bollinger Bands
- volume moving average/ratio

Requirements:
- explicit periods
- safe insufficient-history handling
- no NaN/Infinity in chart coordinates
- safe flat-price RSI handling
- no division-by-zero
- preserve numerical precision
- test first valid output and edge cases

## Candlestick detection

Candlestick recognition is separated from Pattern Lab through `detectCandlestickPatterns`.

Known patterns include:
- Hammer
- Shooting Star
- Bullish Engulfing
- Bearish Engulfing
- Marubozu
- Morning Star
- Evening Star
- Tweezer Bottom
- Tweezer Top
- Three Inside Up
- Three Inside Down
- Three White Soldiers
- Three Black Crows

Detection is classification, not prediction.

Preserve existing thresholds unless intentionally changed. Pattern rules may use:
- body/range ratios
- shadow/body relationships
- real-body engulfing
- ATR-relative filters
- preceding trend/context
- volume confirmation

Do not describe a detection as a guaranteed reversal.

## Pattern Lab

Pattern Lab consumes detector events and adds historical context/outcomes.

It reports or can report:
- bullish/bearish counts
- total signals
- completed/open signals
- +1/+3/+5 bar returns
- average/median return
- 5-bar win rate
- prior trend
- candle body %
- volume ratio
- detector/self-test status

It is an **event study**, not a complete strategy backtest.

For a signal at candle N:
- +1 uses a future candle when available.
- +3 uses a future candle when available.
- +5 is completed only when N+5 exists.
- Signals without enough future candles remain incomplete.
- Do not count incomplete +5 observations as completed results.
- Be alert to look-ahead bias if adding new metrics.

A true backtester requires explicit entry/exit rules, position sizing, costs/slippage assumptions and careful treatment of look-ahead bias.

## Support/resistance

Current implementation is distinct from chart grid lines.

- Uses visible candles.
- Finds 5-candle swing highs/lows.
- Swing high: high >= highs of two preceding and two following candles.
- Swing low: low <= lows of two preceding and two following candles.
- Swing highs become resistance candidates.
- Swing lows become support candidates.
- Clusters nearby prices using `Math.max(priceDelta * 0.015, 0.01)`.
- Requires at least two touches.
- Sorts by touch count, then price.
- Shows up to three support and three resistance levels.
- Displays the mean price of clustered touches.

Document any change to swing detection, tolerance, clustering, minimum touches, ranking or level count.

## Chart model and rendering

Chart geometry is centralized through `buildChartModel(data)`.

It should own/derive:
- price extents
- price delta
- MACD extents/delta
- chart dimensions
- panel gaps
- candle/RSI/MACD areas
- RSI/MACD positions
- candle spacing

The chart sanitizes invalid OHLC/indicator values before SVG coordinate calculations.

There must be only one active `CandlestickChart` declaration.

## Range selection

Range selection is centralized through `calculateRangeSelection(data, selection)`.

It must:
- handle empty data
- validate numeric indexes
- clamp indexes
- support reversed selections
- identify first/last selected candles
- calculate absolute close-to-close price change
- calculate percentage change from the first close
- handle a zero starting price safely
- preserve the existing click-drag interaction and visual overlay

Range performance is descriptive only and must not be presented as a backtest.

## Formatting

Formatting helpers are centralized:
- `formatFixed`
- `formatPrice`
- `formatPercent`
- `formatRatio`
- `formatVolume`
- `calculateRangePercent`

Display values such as price changes and percentages normally use two decimals. Never round the underlying data to solve display issues.

## Translation system

Supported languages:
- EN
- PT-BR
- ES
- FR

Translation data is centralized in `uiTranslations` and applied through React's `LanguageContext` with the `useT()` hook (`t('key')`). There is no DOM TreeWalker/`translateStaticText` pass.

Language state is persisted through browser storage and applied without reload. `document.documentElement.lang` and `document.title` must remain synchronized.

When adding UI text:
- update all four languages
- preserve stable translation keys
- do not translate provider/API identifiers, tickers or raw response fields
- audit chart titles, S/R labels, range-selection text, alerts and Pattern Lab text
- test both themes

## Dashboard state and storage

Persistent state uses the reusable `useStoredState` helper where appropriate.

Persistent values include:
- theme
- language
- watchlist
- Twelve Data key
- Alpaca key ID
- Alpaca secret
- Alpha Vantage key

Storage access is centralized through `STORAGE_KEYS` and the `Storage` helper.

Use defensive parsing and tolerate missing/corrupt localStorage.

## Alerts

Browser alerts:
- persist locally
- can use dictionary defaults
- should avoid duplicates
- should not fire repeatedly on every render
- compare numeric prices safely
- browser notifications depend on browser permission/support

They are not server-side alerts or order execution.

## Regression checks

The application runs three load-time suites, reported through the `window.__APP_DEBUG__` test hook:

- `[REGRESSION CHECKS]` — price formatting, signed percent formatting, ratio formatting, normal/reversed/invalid range selection, session keys, quarter labels, timeframe bars, cache keys.
- `[INDICATOR CHECKS]` — EMA/RSI/SMA/ATR/Bollinger/MACD behaviour including null warm-ups.
- `[CANDLE PATTERN TESTS]` — candlestick detector fixtures.

When changing a pure helper, add or update a focused regression check.

## Security

API keys are stored unencrypted in browser localStorage.

Never:
- commit real credentials
- claim localStorage is encrypted
- imply client-side credentials are secret
- put production secrets in source code

For confidential production use, a backend/serverless proxy is required.

## React/Babel safety

A previous blank-page failure resulted from duplicate declarations, including a duplicate `getPatternStatistics`.

Rules:
- search before declaring a new top-level function/constant
- avoid duplicate components
- avoid duplicate translation arrays/helpers
- after edits, verify the first browser-console error
- remember that a Babel parse error can prevent the entire application from loading

The project previously also suffered from duplicate trailing content after `</html>`; keep exactly one document and no executable content after the closing tag.

## Debugging workflow

### Blank page
1. Check DevTools Console.
2. Fix the first syntax/parse error.
3. Search for duplicate declarations.
4. Check Babel compilation.
5. Confirm exactly one React root.
6. Only then debug application logic.

### Provider/API problem
1. Reproduce with one symbol.
2. Inspect request.
3. Inspect HTTP status.
4. Inspect raw body.
5. Validate response structure.
6. Inspect normalized bars.
7. Inspect state.
8. Inspect chart props/model.
9. Compare with a working provider.
10. Test null/empty/204/malformed cases.

### Chart problem
Separate data correctness from rendering:
1. inspect values
2. inspect chart model
3. inspect SVG dimensions/scales
4. inspect clipping/overflow
5. inspect theme-specific styling

### S/R problem
1. verify visible candles
2. inspect swing points
3. inspect tolerance/clusters
4. verify two-touch minimum
5. verify ordering/count
6. distinguish overlays from grid lines

### Range problem
1. inspect start/end indexes
2. verify clamping
3. verify first/last closes
4. verify absolute and percentage calculations
5. test reversed/empty/single-candle cases

### Translation problem
1. inspect language state
2. inspect `matrix_language`
3. inspect translation key
4. audit all visible strings
5. verify document language
6. test switching without reload

## Testing matrix

Before a provider/data change is complete, test:
- DEMO/mock mode
- Twelve Data
- Alpaca valid bars
- Alpaca null bars
- Alpaca 204
- malformed/empty data
- invalid OHLC
- symbol switching
- cached symbol with missing quote
- force refresh
- indicator recalculation
- candlestick markers
- Pattern Lab
- S/R
- range selection
- reversed/empty/single range
- EN/PT-BR/ES/FR
- language persistence
- theme
- localStorage persistence
- alerts
- responsive layout
- fresh browser session

## Change discipline

For every change:
1. Fetch current `main` before editing.
2. Make the smallest coherent change.
3. Keep mock mode.
4. Preserve existing providers unless intentionally changing them.
5. Avoid duplicate declarations.
6. Keep provider-specific logic isolated.
7. Preserve normalized data boundaries.
8. Preserve internal numerical precision.
9. Test affected UI and console.
10. Update `README.md`, `skills.md` and `agent.md` when behaviour changes.
11. Prefer small, reversible commits.
12. Re-fetch after updates when verifying SHA/content.

## Current refactor state

The project has completed the following logical refactors while remaining single-file:
1. Central browser storage helper.
2. Canonical market-data normalization.
3. Provider-loading isolation (module scope).
4. Alpha Vantage adapter.
5. Twelve Data adapter.
6. Defensive chart-data sanitization.
7. Duplicate chart-component removal.
8. Technical-analysis engine separation (null warm-ups, Wilder ATR).
9. Candlestick detector / Pattern Lab separation.
10. Central formatting/calculation helpers.
11. Central chart model.
12. Central range-selection calculation.
13. React-level translation system (`LanguageContext` + `t()`).
14. `useMarketDataPipeline` hook.
15. Load-time test suites: core/indicator/candle-pattern via `window.__APP_DEBUG__`.
16. Removal of duplicated trailing document content.
17. Alpaca company metadata resolution for active tickers.
18. Active quote refresh when switching to a cached symbol.
19. Section components (HeaderBar, StatusBar, ApiKeyPanel, WatchlistSidebar, ChartSection, RightRail).
20. SRI-pinned CDN scripts and no-referrer policy.
21. Memoized CandlestickChart with rAF-coalesced hover.

The current version is **1.10.0**.

## Next engineering priorities

Planned work should focus on:
1. Centralized symbol metadata resolution across local and provider sources.
2. More robust Alpaca null/204/no-data diagnostics.
3. Clear quote versus last-OHLC semantics.
4. Centralized caching/request policy.
5. More provider fixture/regression tests.

