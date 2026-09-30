# Stock Dashboard — Agent Instructions

## Mission

Maintain `tecepeipe/stockdashboard` as a reliable, understandable static stock-market visualisation application.

The project is intentionally a **single-file HTML application**. Prioritise correctness, resilience, maintainability and preservation of the existing UX over architectural complexity.

## Repository

- Repository: `tecepeipe/stockdashboard`
- Default branch: `main`
- Main application: `index.html`
- Deployment: GitHub Pages
- Live site: `https://tecepeipe.github.io/stockdashboard/`
- No build step.
- Do not migrate to Vite/npm/modules unless explicitly requested.

## Non-negotiable behaviour

### Mock mode

Mock data is a product feature.

When no live credentials exist:
- load mock market data
- keep the dashboard fully explorable
- make demo/simulation status clear

Never silently present mock or stale data as current live data.

### Provider correctness

A request is not successful merely because it returned HTTP 200.

Validate:
- status
- response body
- JSON structure
- symbol
- timestamps
- bar presence
- OHLC numeric values
- finite values
- usable bar count

Handle:
- HTTP 204
- null bars
- empty arrays
- malformed JSON
- provider error objects
- incomplete/invalid OHLC

A provider failure must not corrupt chart state or blank the entire React/Babel application.

## Architecture

Keep one physical `index.html`, but maintain logical boundaries:

```text
Provider request
     |
     v
Provider adapter
     |
     v
Validation + normalization
     |
     v
Common market-data model
     |
     +--------+---------+
     |        |         |
     v        v         v
   Chart  Indicators  Detector
     |        |         |
     +--------+---------+
              |
              v
         Pattern Lab/UI
```

Provider-specific field names must stop at the adapter/normalization boundary.

### Current logical layers

Inside `index.html`, preserve separation between:
- constants/config
- mock data
- storage
- provider adapters (module scope)
- validation/normalization
- timeframe/cache
- symbol metadata
- technical analysis
- candlestick detection
- Pattern Lab
- support/resistance
- chart model/rendering
- range selection
- formatting
- translations (React `LanguageContext` + `t()`)
- alerts/watchlist
- `useMarketDataPipeline` hook
- section components (HeaderBar, StatusBar, ApiKeyPanel, WatchlistSidebar, ChartSection, RightRail)
- Dashboard assembly
- regression/indicator/candle-pattern test suites (`window.__APP_DEBUG__`)

## Provider precedence

Do not change without updating the UI/docs:

1. Alpaca if both Alpaca key ID and secret exist.
2. Alpha Vantage if its key exists.
3. Twelve Data if its key exists.
4. DEMO otherwise.

## Alpaca rules

Alpaca is the provider with the most historical debugging attention.

### Market data

- Current timeframe mapping:
  - 1D -> 5Min
  - 1W -> 15Min
  - 1M -> 30Min
  - 1Y -> 1Day
- Browser implementation uses the IEX feed.
- Use explicit date bounds.
- Normalize `t/o/h/l/c/v`.
- Apply canonical timeframe filtering.
- Recalculate indicators after normalization.
- Explicitly update the active chart after valid data is processed.
- Keep quote retrieval separate from historical OHLC retrieval.

### No-data handling

Do not interpret these as equivalent:

```text
HTTP 200 + usable bars  -> usable market data
HTTP 200 + bars:null     -> no usable bars
HTTP 204                 -> no content
HTTP error               -> provider/request failure
network failure          -> transport failure
```

A valid symbol can still have no usable bars for the requested feed/timeframe.

### Alpaca metadata

Use authenticated `/v2/assets/{symbol}` to resolve company metadata when possible.

Rules:
- metadata lookup is separate from OHLCV retrieval
- cache successful metadata during the session
- asynchronous metadata resolution may update the active symbol/search result
- metadata failure must not prevent ticker selection
- fall back to the ticker as display name
- do not use asset lookup as proof that historical bars exist

## Other providers

### Twelve Data

Keep the adapter isolated. Convert `values` to the common model before analysis.

### Alpha Vantage

Current timeframe mapping:
- 1D -> `TIME_SERIES_INTRADAY` 5-minute
- 1W -> `TIME_SERIES_INTRADAY` 15-minute
- 1M -> `TIME_SERIES_INTRADAY` 30-minute
- 1Y -> `TIME_SERIES_DAILY` full history

Normalize Alpha Vantage's numbered fields into common OHLCV.

Use provider quote/fundamental endpoints where implemented, but keep the dashboard's technical indicators calculated locally.

Do not promise intraday availability on plans that do not provide it.

## Normalized data model

Use a model similar to:

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

Do not make chart/indicator/pattern code consume raw provider objects.

## Timeframe/caching rules

Current configuration:
- 1D: 5min, 10-day lookback, 100 visible candles, 5-minute freshness.
- 1W: 15min, 21-day lookback, 160 maximum.
- 1M: 30min, 70-day lookback, 320 maximum.
- 1Y: 1day, 430-day lookback, 270 maximum, 2-hour freshness.

Filtering:
- sort chronologically
- 1D = latest available session
- 1W = latest five sessions
- 1M = rolling UTC month
- 1Y = rolling UTC year

Cache:
- must be provider/symbol/timeframe specific
- must be versioned
- must have expiry/freshness checks
- force refresh must bypass relevant caches

Important cached-symbol behaviour (since 1.9.2):
- a cached historical dataset does not guarantee a current quote exists
- when the active symbol has cached chart data but no quote snapshot, refresh the active quote
- avoid reloading unrelated data solely because the user changed to a cached symbol

## Symbol metadata

Current sources include:
- `TICKER_DICTIONARY`
- `resolvedSymbols`
- live provider metadata

Do not assume the local dictionary contains every live ticker.

The next intended improvement is to centralize these sources behind one metadata-resolution path.

## Technical-analysis engine

Reusable helpers include:
- average
- SMA
- EMA
- RSI
- true range
- Bollinger Bands
- technical-indicator orchestration

Preserve current indicator periods and algorithms unless the task explicitly changes them.

Known indicators:
- EMA 9/21/50
- SMA 20/50
- MACD 12/26/9
- RSI 14
- ATR
- Bollinger Bands
- volume average/ratio

Requirements:
- guard insufficient history
- guard zero division
- handle flat RSI
- prevent NaN/Infinity from reaching SVG coordinates
- preserve numerical precision
- test first valid outputs and edge cases

## Candlestick detection

Detection is separated through `detectCandlestickPatterns`.

Implemented patterns:
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

Treat detection as deterministic classification.

Do not claim:
- guaranteed reversal
- guaranteed profitability
- future prediction

Do not change pattern thresholds casually.

## Pattern Lab

Pattern Lab is an historical event study.

It may expose:
- bullish/bearish counts
- total signals
- completed/open signals
- average/median returns
- +1/+3/+5 returns
- 5-bar win rate
- sample size
- trend/context
- body percentage
- volume ratio
- detector test status

For signal candle N:
- only count +5 as completed when N+5 exists
- current/latest signals may be incomplete
- do not convert historical observations into predictive probabilities

If adding a true backtester, explicitly define:
- entry
- exit
- position sizing
- transaction costs
- slippage
- execution timing
- look-ahead controls

## Support/resistance

Keep S/R separate from grid lines.

Current algorithm:
- visible candles only
- five-candle swing window
- swing high = high >= two candles before and two after
- swing low = low <= two candles before and two after
- nearby levels clustered with `Math.max(priceDelta * 0.015, 0.01)`
- minimum two touches
- sort by touch count then price
- maximum three support and three resistance levels
- displayed price = average clustered touch price

Do not change this logic without documenting the change.

## Chart model/rendering

`buildChartModel(data)` centralizes chart geometry.

Preserve:
- price/MACD extents
- panel dimensions
- candle spacing
- RSI/MACD positions
- sanitized chart input

The chart must defend against non-finite OHLC/indicator values before calculating SVG coordinates.

There must be exactly one active `CandlestickChart` component.

## Range selection

Use `calculateRangeSelection(data, selection)`.

Requirements:
- empty state -> null
- validate numeric indexes
- clamp indexes
- support reversed drag
- calculate first/last selected candles
- calculate close-to-close price change
- calculate percentage from first close
- protect against zero starting price
- preserve visual overlay and existing interaction

Do not call this a backtest.

## Formatting

Use centralized helpers:
- `formatFixed`
- `formatPrice`
- `formatPercent`
- `formatRatio`
- `formatVolume`
- `calculateRangePercent`

Display rounding is presentation only. Keep underlying market data precise.

## Translation

Supported:
- EN
- PT-BR
- ES
- FR

Central translation dictionary: `uiTranslations`.

UI text is resolved through React's `LanguageContext` and the `useT()` hook (`t('key')`). There is no DOM TreeWalker/`translateStaticText` pass anymore.

When adding UI text:
- update all four languages
- preserve translation keys
- do not translate ticker/provider identifiers
- test without reload
- preserve `document.documentElement.lang` and `document.title`
- test dark/light themes
- audit chart titles, S/R, range, alerts and Pattern Lab

## State/storage

Use the centralized `Storage` helper and `STORAGE_KEYS`.

Use `useStoredState` for persistent React state where appropriate.

Persisted application values currently include:
- theme
- language
- watchlist
- Twelve Data API key
- Alpaca key ID
- Alpaca secret
- Alpha Vantage API key

Storage must tolerate missing or malformed values.

## Security

API keys are stored unencrypted in browser localStorage.

Never:
- hard-code credentials
- commit secrets
- claim localStorage is encrypted
- describe browser-side API credentials as confidential

For production confidentiality, use a backend/serverless proxy.

## React/Babel safety

Previous failures included:
- duplicate `getPatternStatistics`
- duplicate `CandlestickChart`
- duplicate translation declarations
- duplicate trailing JavaScript after `</html>`
- invalid `await` usage in a non-async context

Before adding code:
1. search for existing declarations
2. preserve valid function boundaries
3. ensure `await` remains inside async functions
4. verify one React root
5. verify one closing `</html>` with no trailing executable content

After editing:
- inspect the browser console
- fix the first parse/runtime error before investigating secondary symptoms

## Regression tests

Three load-time suites run on every page load and report through `window.__APP_DEBUG__`:
- `[REGRESSION CHECKS]` — core formatting, range selection, session keys, quarter labels, timeframe bars, cache keys.
- `[INDICATOR CHECKS]` — indicator math including null warm-up behaviour.
- `[CANDLE PATTERN TESTS]` — candlestick detector fixtures.

When changing a pure helper, add a focused regression check.

Do not remove existing tests just to make a change pass.

## Debugging procedure

### Blank page
1. DevTools Console.
2. First syntax/parse error.
3. Duplicate declarations.
4. Babel compilation.
5. React root.
6. Application logic.

### API/chart failure
1. Reproduce with one ticker.
2. Inspect request.
3. Inspect status.
4. Inspect raw body.
5. Validate provider response.
6. Inspect normalized bars.
7. Inspect state.
8. Inspect chart input/model.
9. Compare provider path.
10. Test null/empty/204/malformed responses.

### Quote inconsistency
Check:
- provider quote
- last normalized OHLC candle
- cached quote snapshot
- active ticker
- refresh timing

Do not silently mix sources without documenting the intended semantics.

### S/R
Check visible candle set, swing points, tolerance, clusters, two-touch minimum and displayed count.

### Range
Check indexes, clamping, first/last close, absolute change, percentage change and overlay.

### Language
Check state, storage, dictionary key, DOM language and all visible strings.

## Test matrix

Before completing a change, test as applicable:
- mock
- Twelve Data
- Alpha Vantage
- Alpaca valid data
- Alpaca null bars
- Alpaca 204
- malformed/empty data
- invalid OHLC
- symbol switching
- cached symbol with missing quote
- force refresh
- indicators
- candlestick markers
- Pattern Lab
- S/R
- range selection
- reversed/empty/single range
- EN/PT-BR/ES/FR
- language persistence
- theme
- localStorage
- alerts
- responsive layout
- fresh browser session

## Change discipline

For every modification:
1. Fetch the current `main` file/SHA.
2. Understand the existing implementation.
3. Make the smallest coherent change.
4. Preserve mock mode.
5. Preserve provider behaviour unless intentionally changing it.
6. Avoid duplicate declarations.
7. Keep normalized data boundaries.
8. Preserve internal numerical precision.
9. Test the affected path and browser console.
10. Update README/skills/agent when behaviour changes.
11. Prefer small, reversible commits.
12. Re-fetch after a GitHub update before any subsequent SHA-dependent update.

## Completed refactors through 1.10.0

The single-file architecture now includes logical separation for:
1. centralized browser storage
2. normalized market data
3. provider loading (module scope)
4. Alpha Vantage adapter
5. Twelve Data adapter
6. defensive chart-data sanitization
7. duplicate chart removal
8. technical-analysis engine (null warm-ups, Wilder ATR)
9. candlestick detector and Pattern Lab
10. formatting/calculation helpers
11. chart model
12. range selection
13. translations (React context + `t()`)
14. `useMarketDataPipeline` hook
15. test suites: core/indicator/candle-pattern via `window.__APP_DEBUG__`
16. duplicate trailing-document cleanup
17. Alpaca active-symbol metadata resolution
18. active quote refresh for cached symbol switching
19. section components (HeaderBar, StatusBar, ApiKeyPanel, WatchlistSidebar, ChartSection, RightRail)
20. SRI-pinned CDN scripts
21. memoized CandlestickChart

## Next priorities

Work should proceed in small, testable commits.

Priority order:
1. centralize symbol metadata resolution
2. improve Alpaca null/204/no-data diagnostics
3. define quote versus last-OHLC semantics consistently
4. centralize cache/request policy
5. add provider fixtures and regression coverage
