# Stock Visualisation with Multiple Backend APIs

An interactive, single-file stock market dashboard for exploring price action, technical indicators, candlestick reversal patterns, historical pattern behaviour, support/resistance, market data, watchlists and browser-side price alerts.

The application starts with **built-in mock market data**, allowing visitors to explore the dashboard without API credentials. When live credentials are supplied, the dashboard can retrieve market data from Twelve Data, Alpaca or Alpha Vantage.

## Features

### Interactive market chart
- Custom SVG candlestick/OHLC chart.
- Hover/crosshair inspection.
- 1D, 1W, 1M and 1Y timeframes.
- Interactive click-and-drag range selection.
- Range selection reports absolute price change and percentage change from the first selected candle close to the last selected candle close.
- Calculated support/resistance overlays.
- Candlestick reversal markers.
- RSI, MACD, volume, Bollinger Bands and other chart information.
- Dark and light themes.

### Technical analysis
The dashboard calculates indicators locally so the analysis logic remains consistent across providers:
- Support/resistance.
- EMA 9, 21 and 50.
- SMA 20 and 50.
- MACD 12/26/9 and signal line.
- RSI 14.
- ATR (Wilder smoothing).
- Bollinger Bands.
- Volume moving average / volume ratio.
- Trend/context information used by pattern analysis.

### Candlestick pattern recognition
The detector is separated from Pattern Lab and operates as a deterministic classification layer.

Patterns implemented include:
- Hammer.
- Shooting Star.
- Bullish Engulfing.
- Bearish Engulfing.
- Marubozu.
- Morning Star.
- Evening Star.
- Tweezer Bottom.
- Tweezer Top.
- Three Inside Up.
- Three Inside Down.
- Three White Soldiers.
- Three Black Crows.

Patterns are presented as technical-analysis formations and potential reversal signals, **not predictions or guarantees**.

### Pattern Lab
Pattern Lab is a **historical event study**, not a full trading-strategy backtester.

It can report:
- Detected bullish/bearish events.
- Completed versus incomplete events.
- Average and median subsequent returns.
- +1, +3 and +5 bar returns.
- 5-bar win rate.
- Sample size.
- Prior trend/context.
- Candle body percentage.
- Volume ratio.
- Detector/self-test status.

Events near the end of the available dataset may not have enough future candles for a complete +5-bar observation and are kept separate from completed outcomes.

### Support and resistance
Support/resistance is calculated independently from the chart's faint grid lines.

The current implementation:
- Finds five-candle swing highs and lows in the visible dataset.
- Treats swing highs as resistance candidates and swing lows as support candidates.
- Clusters nearby levels using a price-dependent tolerance.
- Requires at least two touches.
- Displays up to three support and three resistance levels.
- Uses the average clustered price as the displayed level.

### Data providers

Supported live providers:
- Twelve Data.
- Alpaca.
- Alpha Vantage.

Provider-specific responses are converted into a common internal market-data model before charting and analysis.

The intended pipeline is:

`Provider API -> validation -> normalization -> timeframe selection -> indicators/patterns -> chart/UI`

The chart and analysis layers should not depend on provider-specific OHLC field names.

#### Provider precedence
When multiple credentials are present, the current application selects:
1. Alpaca when both Alpaca key ID and secret are configured.
2. Alpha Vantage when its API key is configured.
3. Twelve Data when its API key is configured.
4. DEMO/mock mode when no live credentials are configured.

### Alpaca-specific handling
Alpaca uses compact OHLCV fields such as `t/o/h/l/c/v`. These are normalized before analysis.

The dashboard also resolves Alpaca instrument metadata through the authenticated asset endpoint when possible, allowing company names to be displayed for live tickers that are not in the built-in ticker dictionary.

Important distinctions:
- A valid symbol does not guarantee that the requested market-data feed/timeframe has bars.
- HTTP 200 with `bars: null` is not usable chart data.
- HTTP 204 is treated as no content.
- Empty or malformed responses must not be treated as successful market data.
- Metadata lookup is separate from OHLCV retrieval.
- If Alpaca metadata cannot be resolved, the ticker remains usable and falls back to the ticker as its display name.

### Mock/demo mode
Without live credentials:
- The dashboard generates/uses mock market data.
- Charts, indicators, pattern detection and Pattern Lab remain explorable.
- Mock mode is deliberately retained as part of the product so visitors can understand the application before connecting a provider.

## Caching and refresh behaviour

The dashboard caches timeframe data in browser localStorage with provider/timeframe/symbol-specific keys and configurable freshness periods.

Current timeframe configuration is approximately:
- 1D: 5-minute candles, latest trading session, up to 100 visible candles.
- 1W: 15-minute candles, latest five trading sessions.
- 1M: 30-minute candles, rolling one-month window.
- 1Y: daily candles, rolling one-year window.

The application distinguishes cached historical/chart data from the active quote. When switching to a cached symbol, the active quote can still be refreshed when no current quote snapshot exists.

Manual refresh is intended to bypass relevant cached data.

Cache data is versioned so changes to the cached model can invalidate older entries.

## Search and symbol metadata

Live ticker handling is provider-aware:
- Twelve Data uses live symbol search and filters supported US exchange results.
- Alpha Vantage uses `SYMBOL_SEARCH`.
- Alpaca uses exact-symbol asset resolution rather than relying solely on the local ticker dictionary.
- Successful Alpaca asset metadata is cached during the session.
- Unknown live tickers can flow through the normal OHLCV pipeline when the provider supports them.
- Mock mode uses the local `TICKER_DICTIONARY`.

Symbol metadata and market-data availability are deliberately treated as separate concerns.

## Watchlist and alerts

- Watchlist entries persist in browser localStorage.
- Price alerts can be configured for the selected instrument.
- Default alert thresholds can come from the ticker dictionary.
- Alert configuration is retained locally.
- Browser notifications can be used when the configured price threshold is approached.
- These are browser-side features; they are not server-side trading or notification services.

## Multilingual UI

The UI currently supports:
- English (EN).
- Brazilian Portuguese (PT-BR).
- Spanish (ES).
- French (FR).

The selected language is persisted in browser localStorage and applied without requiring a page reload.

## Getting started

### Open the application

Open `index.html` in a modern browser.

For local development, a simple static server can be used:

```bash
python -m http.server 8000
```

Then browse to `http://localhost:8000`.

### Explore demo mode

Open the application without credentials to explore the mock market data, chart, indicators, patterns, Pattern Lab, watchlist and alerts.

### Connect a provider

1. Obtain credentials from the provider you want to use.
2. Open the dashboard's connection controls.
3. Enter the required credential(s).
4. Select a ticker.
5. Select a timeframe.
6. Load or refresh the market data.

Provider limits, symbol coverage, exchange coverage, realtime access and historical-data availability depend on the provider and account plan.

## Architecture

The application deliberately remains a single HTML file. It does **not** use Vite, npm, a JavaScript build pipeline or separate source modules.

Logical separation is maintained inside `index.html` through reusable functions and sections:

```text
Mock / Twelve Data / Alpaca / Alpha Vantage
                    |
                    v
          Provider-specific adapter
                    |
                    v
       Validated normalized market data
                    |
          +---------+---------+
          |         |         |
          v         v         v
        Chart   Indicators  Pattern detector
          |         |         |
          +---------+---------+
                    |
                    v
              Pattern Lab/UI
```

Recent refactoring has separated:
- Browser storage access.
- Market-data normalization.
- Provider loading (module scope).
- Alpha Vantage, Twelve Data and Alpaca adapters.
- Technical-analysis calculations.
- Candlestick detection and Pattern Lab analysis.
- Formatting/calculation helpers.
- Chart geometry/model construction.
- Range-selection calculations.
- Translation data and React-level i18n (`LanguageContext` + `t()`).
- The data pipeline in the `useMarketDataPipeline` hook.
- Dashboard state handling and section components (`HeaderBar`, `StatusBar`, `ApiKeyPanel`, `WatchlistSidebar`, `ChartSection`, `RightRail`).

The goal is logical modularity while preserving the simple static deployment model.

## Data model

Provider responses are normalized to a common model similar to:

```js
{
  symbol: "MSFT",
  name: "Microsoft Corporation",
  exchange: "NASDAQ",
  currency: "USD",
  provider: "ALPACA",
  modelVersion: "v1",
  bars: [
    {
      time: "...",
      open: 497.64,
      high: 498.14,
      low: 491.13,
      close: 493.10,
      volume: 659000
    }
  ]
}
```

Raw provider field names should not leak into chart or analysis code.

## Testing and resilience

The application includes lightweight load-time test suites, exposed through the `window.__APP_DEBUG__` test hook:

- `[REGRESSION CHECKS]` — core pure helpers (price/percent/ratio formatting, range selection, session keys, quarter labels, timeframe bars, cache keys).
- `[INDICATOR CHECKS]` — EMA/RSI/SMA/ATR/Bollinger/MACD/RSI-series behaviour including null warm-ups.
- `[CANDLE PATTERN TESTS]` — candlestick detector fixtures (hammer, engulfing, stars, tweezers, soldiers/crows and more).

Current regression coverage includes checks for:
- Price formatting.
- Signed percentage formatting.
- Ratio formatting.
- Normal/reversed range selection.
- Invalid range selection.
- Candlestick detector behaviour.

Provider troubleshooting should verify the complete path:

`request -> response -> validation -> normalization -> state -> indicators -> pattern analysis -> chart render`

A successful HTTP response alone is not considered proof that the application has received usable market data.

## Data and privacy

- Mock data is bundled with the application.
- API credentials are stored in browser localStorage.
- **API keys are stored unencrypted in browser localStorage.**
- Client-side storage is not a secure secret store.
- A user with browser access can inspect localStorage and client-side application code.
- Do not hard-code real credentials into the repository.
- For a production application where API credentials must remain confidential, use a backend/serverless proxy or another server-side secret mechanism.

Locally cached market data, watchlists, alerts and UI preferences may also reside in browser storage.

## Technology

- HTML/CSS/JavaScript.
- React 18 via browser/CDN loading.
- ReactDOM.
- Babel Standalone.
- Tailwind CSS via CDN/browser loading.
- SVG chart rendering.
- Browser localStorage.
- Browser Notifications API where available.
- Static GitHub Pages deployment.

## Project status

This is an interactive market-data visualisation and technical-analysis experimentation tool. It is designed to make provider data and common technical-analysis concepts easy to explore in a browser.

The project is **not** a broker, execution platform or secure trading system.

Technical indicators, candlestick patterns and Pattern Lab statistics should be independently validated before being used in an investment workflow.

## Security considerations

This is a public client-side application. Credentials entered into the UI are sent directly from the browser to the selected provider and stored unencrypted in localStorage.

Do not publish real credentials in `index.html`, Git history, screenshots or documentation.

For confidential production credentials, introduce a server-side or serverless API layer that keeps secrets outside the browser.

## Future improvements

Likely future work includes:
- Stronger provider response/schema validation and clearer API diagnostics.
- Better handling and explanation of Alpaca no-data/204 cases.
- Centralized symbol metadata resolution across all providers.
- Further caching/request-efficiency improvements.
- Clearer quote versus last-OHLC-candle semantics.
- Additional regression and provider-fixture tests.
- Better live ticker discovery and metadata coverage.
- More capable alerts and watchlists.
- Configurable chart overlays.
- True historical backtesting with explicit strategy rules, entry/exit logic and transaction assumptions.
- Additional indicators and analysis views.
- Export of chart and analysis data.

## Disclaimer

This project is provided for informational and educational purposes only. It is not financial advice. No indicator, pattern, statistic or visualisation should be interpreted as a recommendation or guarantee to buy or sell a financial instrument.
