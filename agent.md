# Stock Dashboard — Agent Instructions

## Mission

Maintain and improve the `tecepeipe/stockdashboard` repository as a reliable, understandable, static stock-market visualisation application.

The application is intentionally a single-file HTML project. Prioritise correctness, resilience and preserving the existing user experience over introducing architectural complexity.

## Repository

- Repository: `tecepeipe/stockdashboard`
- Default branch: `main`
- Main file: `index.html`
- Deployment: GitHub Pages
- Public static application
- No build step should be assumed.

## Non-negotiable behaviour

### 1. Keep mock/demo mode

The application must remain useful without an API key.

Mock financial data is intentional and must not be removed merely because live APIs are available.

When live credentials are absent:
- load mock data
- make it obvious that the data is simulated/demo data
- allow visitors to explore charts, indicators and pattern recognition

When live credentials are present:
- use the selected provider
- clearly identify live/provider mode
- never silently present stale/mock data as current live data

### 2. Never equate HTTP success with data success

A live request is successful only when its response contains valid, usable market data.

Examples already encountered:
- HTTP 200 with a usable daily bar
- HTTP 200 with `{"bars":null,"next_page_token":null,"symbol":"TSLA"}`
- HTTP 204 No Content

The agent must validate:
- HTTP status
- response body
- JSON structure
- symbol
- timestamp
- bar array/object presence
- OHLC numeric values
- finite numbers
- usable bar count

A 200 response with null/empty bars must not be treated as valid chart data.

### 3. Trace data all the way to rendering

If the browser receives a valid response but the chart does not change, debug the complete chain:

1. request construction
2. network response
3. response parsing
4. provider validation
5. normalization
6. state update
7. indicator recalculation
8. pattern detection
9. chart data transformation
10. React render

Do not assume that a 200 in DevTools means the graph has been updated.

Add temporary diagnostics where useful, especially:
- provider
- symbol
- timeframe
- status
- response shape
- normalized bar count
- first/last timestamp
- state bar count

Remove noisy diagnostics when the fix is complete unless they provide lasting value.

## Single-file architecture

Do not split the project into modules unless explicitly requested.

Within `index.html`, prefer logical sections such as:
- configuration/constants
- mock data
- provider adapters
- API validation/normalisation
- indicator calculations
- candlestick detection
- Pattern Lab statistics
- chart components/helpers
- alert/watchlist logic
- UI components
- application state

Logical modularity is encouraged even though the physical project remains one file.

## React/Babel safety

The project uses browser-side React/Babel.

A previous blank-page failure was caused by:

`Identifier 'getPatternStatistics' has already been declared`

Therefore:
- never introduce duplicate top-level declarations
- search for an existing function before creating it
- prefer modifying an existing helper when appropriate
- after edits, inspect the browser console for Babel syntax errors
- remember that one parse-time error can prevent the entire application from rendering

## Data model

Use one internal normalized OHLC representation regardless of provider.

A typical normalized bar should contain:
- timestamp
- open
- high
- low
- close
- volume

Provider-specific fields should be translated before analysis.

For Alpaca-style daily data, fields may arrive as:
- `o` open
- `h` high
- `l` low
- `c` close
- `v` volume
- `t` timestamp
- `n` trade/count metadata

Do not make chart code dependent on Alpaca's raw field names.

## Numerical correctness

Internal calculations should retain full precision.

For display-only values such as:
- day change
- day range
- percentage changes

use two decimal places where appropriate.

Do not round the underlying OHLC series merely to fix visual decimal noise.

Guard all calculations against:
- division by zero
- missing history
- null values
- NaN
- Infinity
- flat-price RSI cases
- zero/near-zero candle ranges

## Indicators

Preserve the documented periods unless a change is explicitly requested:
- RSI: 14
- MACD: 12/26/9
- volume average: commonly 14
- other SMA/EMA/Bollinger/ATR periods as defined by the current implementation

When modifying an indicator:
- document the formula/period
- preserve insufficient-history handling
- test the first valid output
- test flat and missing data
- verify chart alignment with candles

## Candlestick detectors

Treat pattern detection as classification.

Known patterns:
- Hammer
- Shooting Star
- Bullish Engulfing
- Bearish Engulfing
- Marubozu
- other patterns present in the current implementation

Rules must be explicit and deterministic.

Examples of established logic:
- Hammer/Shooting Star use body-to-range and shadow-to-body relationships.
- Engulfing is based on real-body relationships.
- Marubozu requires a dominant body and small shadows and can use ATR as a scale filter.
- Volume confirmation can compare current volume with the previous 14-candle average.
- Trend/context can use preceding candles.

Do not change thresholds casually. If changing them, explain what changed and why.

Do not describe a detected pattern as a guaranteed reversal or trading signal.

## Pattern Lab

Pattern Lab is an historical event-study view.

It should distinguish:
- detected signal
- completed outcome
- incomplete/current signal

For a five-bar outcome:
- only count a signal as completed when five future candles exist
- do not treat the current/latest signal as a completed five-bar result

Useful statistics:
- total signals
- bullish/bearish count
- completed signals
- incomplete signals
- average and median returns
- +1/+3/+5 bar returns
- 5-bar win rate
- sample size
- trend
- candle-body percentage
- volume ratio

If bearish outcomes are direction-adjusted, make that methodology explicit.

Never turn historical event statistics into claims of future probability.

## Chart conventions

The faint horizontal lines behind the OHLC chart are grid lines.

Do not call them support/resistance.

Support/resistance requires a separate, explicit algorithm and should be labelled only when actually calculated.

Preferred chart titles:
- PRICE + REVERSAL PATTERNS
- RSI (14)
- MACD (12,26,9)
- VOLUME
- BOLLINGER BANDS
- ATR
- PATTERN LAB // HISTORICAL EVENT STUDY

Titles should make the purpose of each graph immediately clear.

## Security

The application is public and client-side.

If API keys are stored using localStorage, describe this accurately:

> API keys are stored unencrypted in the browser's localStorage.

Do not say merely "stored locally" if security is being discussed, because that can imply encryption.

Rules:
- never commit real API keys
- never hard-code credentials
- do not present browser storage as a secure secret store
- explain that public client-side code cannot keep an API secret
- recommend a server-side/serverless proxy if true credential confidentiality is required

## Caching and stale data

The HTML can include no-cache meta tags, but meta tags are not a substitute for server-side HTTP cache-control headers.

When a refresh appears not to work:
- inspect network status
- inspect response body
- inspect timestamps
- inspect normalized bar count
- inspect state
- inspect chart input
- check browser cache/service workers
- confirm symbol/timeframe
- confirm that an empty response has not replaced valid state

Never "fix" stale data by simply forcing a redraw if the underlying state is wrong.

## UI preservation

The existing style is a compact trading-terminal interface.

Preserve:
- dark/light theme
- Space Grotesk
- JetBrains Mono for technical/numeric content
- Tailwind-based styling
- SVG icons
- dense cards/tables
- green/red market semantics
- responsive behaviour

Avoid broad visual rewrites when the requested change is functional.

## Alerts and watchlists

Browser-based alerts/watchlists may use localStorage.

Requirements:
- validate stored JSON
- tolerate deleted/corrupt entries
- compare numeric prices safely
- avoid duplicate alerts
- avoid repeated firing on every render
- keep browser persistence separate from live API state

These are browser features, not server-side notifications.

## Debugging procedure

For a blank page:
1. open DevTools Console
2. fix the first syntax/runtime error
3. check for duplicate declarations
4. check Babel compilation
5. only then investigate application logic

For an API refresh problem:
1. reproduce with one symbol, preferably MSFT/TSLA
2. inspect the exact request
3. inspect HTTP status
4. inspect raw response
5. validate the response structure
6. inspect normalized bars
7. inspect React state
8. inspect chart input
9. compare with working Twelve Data path
10. test empty/null/204 responses

For a visual problem:
1. determine whether the data is wrong or only the rendering is wrong
2. inspect actual values
3. check SVG dimensions/scales
4. check clipping/overflow
5. check responsive sizing
6. avoid changing calculations to solve CSS/layout problems

## Testing requirements

Before considering a data-path change complete, test:
- no API key / mock mode
- valid Twelve Data
- valid Alpaca
- Alpaca 200 + valid bars
- Alpaca 200 + null bars
- 204
- malformed response
- empty bars
- invalid OHLC
- stale timestamps
- symbol switching
- chart refresh
- indicator recomputation
- pattern markers
- Pattern Lab
- localStorage persistence
- alerts
- theme
- responsive layout

Always verify the page loads from a fresh browser session.

## Documentation rules

README and inline documentation should describe the application as a visualisation/analysis tool.

Good wording:

> An interactive, single-file stock market dashboard for exploring price action, technical indicators, candlestick reversal patterns, historical pattern statistics, and market data.

Explain:
- mock/demo mode
- Twelve Data and Alpaca support
- browser-side API-key storage limitation
- technical indicators
- pattern recognition
- Pattern Lab as historical analysis
- static deployment

Avoid language claiming that patterns predict the market or provide guaranteed trading signals.

## Change discipline

For every requested modification:
1. inspect existing implementation
2. identify the smallest correct change
3. preserve mock mode
4. preserve existing providers unless intentionally changing them
5. avoid duplicate declarations
6. validate external data
7. preserve internal numerical precision
8. test the browser console
9. test the affected UI path
10. document important behavioural changes

Do not remove existing functionality just to simplify a fix.

## Backlog

Potential future work:
- robust provider adapters and diagnostics
- clearer stale-data detection
- stronger response/schema validation
- logical internal modules while retaining a single HTML file
- configurable overlays
- richer signal context
- true historical backtesting
- expanded watchlists
- browser alerts
- automated tests
- explicit support/resistance algorithms
- improved chart legends/titles
- API/provider diagnostics

Backtesting must use explicit strategy rules and avoid look-ahead bias. It is separate from Pattern Lab's descriptive historical event statistics.
