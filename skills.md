# Stock Dashboard — Project Skills

## Project identity

- Repository: `tecepeipe/stockdashboard`
- Primary application: `index.html`
- Deployment: static GitHub Pages
- Live site: `https://tecepeipe.github.io/stockdashboard/`
- Language: HTML/CSS/JavaScript
- Architecture: intentionally single-file client-side application
- Main purpose: interactive stock-market visualisation for price action, technical indicators, candlestick reversal patterns, historical pattern statistics, market data, watchlists and browser-side alerts.

## Core product principles

1. Keep the application usable without credentials.
   - Built-in mock market data is intentional.
   - A visitor should be able to understand and explore the dashboard before entering an API key.
   - Never make mock data look like live market data. The UI should clearly distinguish mock/demo mode from live provider data.

2. Live data must be validated before it reaches chart state.
   - HTTP 200 does not necessarily mean usable market data.
   - Validate status, body, provider-specific structure, symbol, timestamps and the presence of usable bars.
   - Handle HTTP 204 and empty/null responses explicitly.
   - Do not silently replace a failed live refresh with stale data while implying that the graph is current.

3. Preserve numerical precision internally.
   - Formatting such as daily change/range should normally use two decimal places for display only.
   - Use `Number(value).toFixed(2)` or equivalent presentation formatting.
   - Do not round source OHLC/indicator data merely to make the UI look cleaner.

4. Prefer defensive, observable code.
   - Invalid API responses should produce a visible diagnostic/error state.
   - Avoid exceptions that leave React/Babel rendering a blank page.
   - Guard against missing, null, malformed and non-finite values.

## Technology and structure

The current design is a browser-loaded single HTML file rather than a Vite/npm project.

Typical dependencies are loaded from CDNs:
- React 18
- ReactDOM
- Babel Standalone
- Tailwind CSS
- Google fonts such as Space Grotesk and JetBrains Mono

Do not introduce a build system, package manager or multiple modules unless explicitly requested. The code can still be logically modular inside the single file through clearly separated functions/sections.

## Data providers

The dashboard has been designed around:
- Twelve Data
- Alpaca market-data API

Provider-specific response handling must remain separate from the application's normalized internal model.

Preferred pipeline:

`provider response -> validation -> normalization -> application state -> indicators/patterns -> chart/UI`

The chart and analysis layers should not need to know whether data came from Twelve Data, Alpaca or mock data.

### Alpaca lessons learned

During debugging, the browser showed successful requests to `data.alpaca.markets` with HTTP 200, but the OHLC graph did not update.

Observed response examples included:
- a valid daily-bar style object containing `c/h/l/n/o/t/v`
- `{"bars":null,"next_page_token":null,"symbol":"TSLA"}`
- HTTP 204 No Content responses

Therefore:
- treat HTTP 200 + `bars:null` as no usable data, not success
- treat HTTP 204 as an empty/no-content result
- validate the actual response body before updating chart state
- log provider, symbol, timeframe, HTTP status and normalized-bar count during troubleshooting
- ensure the successful response is actually passed through the same state update/render path as Twelve Data
- do not assume a network request appearing as 200 in browser DevTools proves the UI has received usable OHLC data

## Charting

The dashboard uses custom/client-side chart rendering, including SVG-based visualisation.

Expected chart features include:
- OHLC/candlestick price chart
- reversal-pattern markers
- responsive layout
- hover/crosshair information
- range selection
- indicator panels
- volume
- overlays such as Bollinger Bands
- chart grid lines

### Grid lines

The faint horizontal lines visible behind OHLC candles are chart grid lines.

They are NOT automatically support/resistance levels.

Only label them as support/resistance if an explicit support/resistance calculation and visualisation has been implemented.

Recommended titles:
- `PRICE + REVERSAL PATTERNS`
- `RSI (14)`
- `MACD (12,26,9)`
- `VOLUME`
- `BOLLINGER BANDS`
- `ATR`
- `PATTERN LAB // HISTORICAL EVENT STUDY`

## Technical indicators

Known indicators/analysis include:
- SMA
- EMA
- MACD (12/26/9)
- RSI (14)
- Bollinger Bands
- ATR
- Volume SMA / volume ratio
- trend/context calculations

Indicator implementation requirements:
- return null/undefined safely when there is insufficient history
- avoid NaN/Infinity entering chart coordinates or statistics
- handle flat-price RSI edge cases
- keep ATR-based thresholds scale-independent
- preserve the exact source series where possible
- make period constants explicit

## Candlestick pattern recognition

Known reversal/event patterns include:
- Hammer
- Shooting Star
- Bullish Engulfing
- Bearish Engulfing
- Marubozu
- additional multi-candle/context-aware detections as implemented in the current code

Pattern logic should be treated as deterministic classification, not prediction.

Important rules discussed:
- Hammer/Shooting Star use body/range and shadow/body relationships.
- A representative implementation uses body/range <= 35%, relevant shadow >= 2x body, and the opposite shadow constrained relative to the body.
- Engulfing should compare real bodies rather than requiring the entire high/low range to engulf the previous candle.
- Marubozu can use body/range >= 90%, very small shadows, and an ATR-relative minimum body size.
- Volume confirmation can use current volume / prior 14-candle average >= 1.10x.
- Context can include the preceding three candles for trend classification.

Thresholds are implementation rules, not claims that a pattern guarantees a reversal.

## Pattern Lab

Pattern Lab is best described as a historical event study.

It should communicate:
- number of detected signals
- bullish/bearish counts
- completed versus incomplete signals
- average return after 1/3/5 bars where available
- median return
- 5-bar win rate
- sample size
- trend/context at signal time
- candle body percentage
- volume ratio
- detector test status

Do NOT describe these statistics as guaranteed predictive probabilities.

For bearish patterns, direction-adjusted returns may be useful so that a move in the expected bearish direction is represented consistently. The UI should make this methodology clear.

Current/latest signals may be incomplete because there are not yet five future candles. They must not be counted as completed 5-bar outcomes.

## API keys and security

The application is a public client-side application.

If the implementation stores credentials in browser localStorage, the accurate wording is:

> API keys are stored unencrypted in the browser's localStorage.

Important:
- `localStorage` does not encrypt values.
- Never hard-code a real API key into the repository.
- Do not imply that a client-side API key is secret.
- Anyone with browser access can inspect client-side JavaScript and storage.
- For genuinely confidential credentials, use a server-side/serverless proxy or another backend-controlled secret mechanism.

Local storage is acceptable for a personal/demo browser application when the user understands this limitation, but it is not a secure secret store.

## Browser persistence

Browser localStorage may be used for:
- API configuration
- selected provider
- watchlist
- browser alerts
- UI preferences/theme

Use defensive parsing:
- handle missing keys
- handle malformed JSON
- validate data types
- avoid breaking startup because localStorage contains corrupted data

## Caching and freshness

The HTML may include no-cache meta directives such as:
- `Cache-Control: no-cache, no-store, must-revalidate`
- `Pragma: no-cache`
- `Expires: 0`

These are useful hints but are not equivalent to server-side HTTP cache headers.

When debugging stale UI:
1. check the actual network response
2. check whether the response body is valid
3. check normalization
4. check state assignment
5. check whether the selected symbol/timeframe actually changed
6. check chart data props/state
7. check for browser caching/service-worker effects
8. confirm the displayed timestamp is from the latest response

## Common failure modes

### Blank page after a code change

A known failure was:

`Uncaught SyntaxError: Identifier 'getPatternStatistics' has already been declared`

This prevents Babel from evaluating the script and can leave the page completely blank.

Before adding/refactoring functions:
- search for duplicate function/const/let declarations
- avoid redeclaring top-level names
- keep helper names unique
- inspect the first console error, not later cascading errors

### Successful request but unchanged graph

Do not stop at the network tab.

Trace:
`request -> response -> validation -> normalization -> state -> derived indicators/patterns -> chart props -> render`

Log the number of usable OHLC bars at each boundary.

### 204 response

HTTP 204 means no content. Do not attempt normal JSON parsing. Treat it as an explicit empty response.

### 200 with null bars

A response such as `bars:null` is not usable OHLC data. Do not overwrite valid chart state with it unless the application intentionally wants to clear the chart.

## UI style

The visual language is a dense trading-terminal/dashboard aesthetic:
- dark/light theme
- Space Grotesk for UI text
- JetBrains Mono for numerical/technical values
- Tailwind utility styling
- SVG icons
- compact cards and tables
- green/red market semantics
- responsive panels

Avoid unnecessary redesigns when fixing functional bugs.

## Testing checklist

Every meaningful data change should be tested in:
- startup with no API key -> mock data
- valid Twelve Data response
- valid Alpaca response
- Alpaca HTTP 200 with valid bars
- Alpaca HTTP 200 with `bars:null`
- HTTP 204
- malformed JSON
- provider error response
- empty bar array
- wrong symbol
- stale/older timestamps
- insufficient history
- indicator edge cases
- pattern detector edge cases
- current/incomplete Pattern Lab signals
- watchlist persistence
- alert persistence and numeric comparison
- theme switching
- ticker/symbol switching
- responsive chart rendering

Always check the browser console after significant edits.

## Safe development style

When modifying the single HTML file:
- make the smallest coherent change
- preserve existing mock mode
- preserve existing provider support
- avoid duplicate declarations
- avoid unnecessary dependency changes
- keep provider adapters isolated
- keep UI formatting separate from calculations
- validate every external response
- use comments where a rule is non-obvious
- do not silently change pattern thresholds without documenting the change

## Future enhancements

Possible roadmap:
1. stronger provider response validation and diagnostics
2. clean provider normalization layer
3. further logical modularisation while retaining single-file deployment
4. configurable chart overlays
5. clearer signal/context annotations
6. historical backtesting using real historical data
7. expanded watchlists
8. browser-based price alerts
9. broader automated detector/unit tests
10. optional explicit support/resistance calculations
11. richer chart legends/titles
12. clearer stale-data/API diagnostics

Backtesting must be kept distinct from Pattern Lab: Pattern Lab describes historical pattern outcomes; a backtester evaluates an explicit trading strategy with entry/exit/position rules and should account for look-ahead bias, incomplete bars and transaction assumptions.
