# Stock Visualisation with Twelve Data API

An interactive, single-file stock market dashboard for exploring price action, technical indicators, candlestick reversal patterns, and market data.

The application starts with **built-in mock market data**, allowing visitors to explore the interface without an API key. When a Twelve Data API key is provided, the dashboard can retrieve market data and use it in the visualisations.

## Features

- **Interactive candlestick charts**
  - Visualise historical price movement.
  - Inspect chart data using an interactive crosshair.
  - Change the chart timeframe.

- **Candlestick pattern recognition**
  - Detect selected bullish and bearish candlestick formations.
  - Highlight potential market reversal signals.
  - Explore detected patterns through the Pattern Lab.

- **Technical indicators**
  - EMA 9, 21, and 50
  - SMA 20 and 50
  - MACD and signal line
  - RSI
  - ATR
  - Bollinger Bands
  - Volume moving average

- **Pattern Lab**
  - Summarises detected bullish and bearish patterns.
  - Provides illustrative historical observations based on subsequent candles.
  - Helps users explore how patterns behaved in the available dataset.

- **Mock data fallback**
  - The dashboard remains usable without an API key.
  - Mock data demonstrates the application’s functionality before connecting to live market data.

- **Twelve Data integration**
  - Connect the dashboard to Twelve Data by supplying an API key.
  - Use retrieved market data in the dashboard when the API is configured.

- **Price alerts**
  - Create simple price-threshold entries for the currently selected instrument.
  - Store alert entries locally in the browser.

- **Single-file application**
  - Designed to run as a standalone HTML file.
  - No Vite migration or build process is required.

## Getting Started

### 1. Open the application

Open the HTML file in a modern web browser.

You can also serve the directory locally, for example:

```bash
python -m http.server 8000
```

Then open:

```text
http://localhost:8000
```

### 2. Explore the mock data

The dashboard loads with sample market data by default. This allows you to explore the charts, indicators, and pattern-recognition features without configuring an API key.

### 3. Connect Twelve Data

1. Obtain an API key from [Twelve Data](https://twelvedata.com/).
2. Use the dashboard’s connection control.
3. Enter your API key.
4. Select an instrument and timeframe.
5. Load the available market data.

API availability, rate limits, supported symbols, and historical-data access depend on your Twelve Data account and plan.

## How to Interpret the Patterns

Candlestick patterns are **technical-analysis signals, not guarantees**.

A detected bullish or bearish reversal pattern indicates that the price structure resembles a recognised formation. It does not establish that a reversal will occur, nor does it account for broader market conditions, news, liquidity, transaction costs, or future volatility.

The Pattern Lab’s historical observations are intended for exploration and education. They should not be treated as a standalone trading strategy or financial advice.

## Data and Privacy

- Mock data is included in the application so that it can be used without an API key.
- API keys and locally stored preferences may be retained in browser local storage, depending on the application configuration.
- Do not share your API key publicly or commit it to a public repository.
- Review Twelve Data’s terms, usage limits, and API-key security recommendations before using the application with a production key.

## Technology

The project is intentionally implemented as a single HTML file and uses browser-based JavaScript for the dashboard functionality.

Core concepts include:

- HTML, CSS, and JavaScript
- Candlestick chart visualisation
- Technical-indicator calculations
- Candlestick-pattern recognition
- Twelve Data API integration
- Browser local storage

## Project Status

This project is an interactive visualisation and experimentation tool. Pattern recognition and technical indicators are provided for research and educational purposes and should be validated independently before being used in any investment workflow.

## Possible Future Improvements

- Automatic alert evaluation when prices cross configured thresholds
- Browser notifications for triggered alerts
- More candlestick formations and configurable pattern filters
- Backtesting with configurable entry, exit, and risk-management rules
- Additional indicators and multi-chart layouts
- Improved API error handling and rate-limit feedback
- Exporting chart data, signals, and analysis results
- Optional backend proxy to keep API keys out of client-side code

## Disclaimer

This project is provided for informational and educational purposes only. It is not financial advice, and no pattern, indicator, statistic, or visualisation should be interpreted as a recommendation to buy or sell a financial instrument. Always perform your own research and consider consulting a qualified financial professional.
