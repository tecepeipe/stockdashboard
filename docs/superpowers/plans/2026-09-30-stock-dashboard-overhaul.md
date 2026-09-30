# Stock Dashboard Review Fixes & Hardening (v1.9.2 → v1.10.0) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix all bugs and issues found in the project review (P0–P3): indicator correctness, provider/data bugs, error handling, rate limiting, security, performance, and structural refactors.

**Architecture:** Single-file React app (`index.html`, React 18 + Babel standalone + Tailwind CDN). All changes stay inside `index.html` (project convention: no build tooling). Pure logic (indicators, detectors, timeframe/session helpers, provider adapters) is progressively moved to module scope; a `useMarketDataPipeline` hook extracts the data pipeline out of `Dashboard`; i18n moves from DOM TreeWalker to React context.

**Tech Stack:** React 18 (UMD), Babel standalone, Tailwind browser CDN, vanilla JS, Playwright MCP for verification.

**Spec:** This plan implements the review findings from the conversation (verified against source). Primary file: `/home/fabricio/stockdashboard/index.html` (2,980 lines at plan time — line numbers shift after each task; re-grep by symbol name, not line number).

## Global Constraints

- **No git commits** — user has not requested commits. Never run `git commit`.
- **No new files except:** `docs/superpowers/plans/*.md` (this plan) and `.gitignore` (Task 15). All app code stays in `index.html`.
- **No build tooling, no npm dependencies, no frameworks added.**
- **Preserve behavior** unless the plan explicitly changes it. The 6 existing candle pattern fixtures and existing core regression tests must keep passing (unless a task explicitly changes an expectation).
- **Version bump:** final version is `1.10.0` (meta tag + docs, Task 16).

### Standard verification procedure (used by every task)

1. Start server (once per session):
   ```bash
   pkill -f "http.server 8123" 2>/dev/null; sleep 0.3
   python3 -m http.server 8123 --directory /home/fabricio/stockdashboard >/tmp/stockdashboard-server.log 2>&1 &
   ```
2. Navigate Playwright to `http://localhost:8123/index.html` (fresh reload for each check).
3. Read console messages (`playwright_browser_console_messages`, level `info`+`error`):
   - Expect: `[REGRESSION CHECKS] N/N passed` (N grows as tests are added).
   - Expect: **zero** `error` messages, zero `pageerror`, zero `unhandledrejection`.
4. Programmatic state assertions via `window.__APP_DEBUG__` (installed in Task 1):
   ```js
   playwright_browser_evaluate({ function: "() => window.__APP_DEBUG__" })
   ```
   Demo mode must show `provider: 'DEMO'` with `chartData` populated.

If a task's steps say "Run standard verification", follow the above.

---

### Task 1: Module-scope foundation — constants, session/timeframe helpers, quarter label, debug hook

**Goal:** Move pure constants/helpers out of `Dashboard` to module scope (enables unit tests + fixes the timezone/Q1 bugs), unify fallback constants, add a test/debug hook.

**Files:**
- Modify: `index.html` (STORAGE_KEYS ~line 323, CORE_REGRESSION_TESTS ~140, `Dashboard` body ~1166–1411, effects ~1175–1180, return ~2164)

**Interfaces (Produces):**
- `fiscalQuarterLabel(dateStr) → 'Q1 25'` (module scope, defined before `CORE_REGRESSION_TESTS`)
- `sessionKey(bar) → 'YYYY-MM-DD'` (module scope)
- `getTimeframeBars(bars, timeframe) → bars[]` (module scope)
- `normalizeMarketBars(bars)`, `createMarketData(symbol, bars, provider)` (module scope)
- `timeframeConfig`, `CACHE_VERSION = 'v8'`, `getCacheKey`, `readTimeframeCache`, `writeTimeframeCache`, `QUOTE_CACHE_TTL_MS`, `getQuoteCacheKey(provider, ticker)`, `FALLBACK_FUNDAMENTALS` (module scope)
- `STORAGE_KEYS.PRICE_ALERT_NOTIFICATIONS`
- `window.__APP_DEBUG__` refreshed every render: `{ provider, chartData, stockPrices, syncStatusText, quoteInfo, tests }`

- [ ] **Step 1: Add failing tests** — append to `CORE_REGRESSION_TESTS` (before line 171's runner):

```js
{
    name: 'fiscalQuarterLabel maps months to calendar quarters',
    run: () => fiscalQuarterLabel('2025-01-15') === 'Q1 25' &&
        fiscalQuarterLabel('2025-04-01') === 'Q2 25' &&
        fiscalQuarterLabel('2025-06-30') === 'Q2 25' &&
        fiscalQuarterLabel('2025-12-31') === 'Q4 25' &&
        fiscalQuarterLabel(null) === 'N/A'
},
{
    name: 'sessionKey uses naive timestamp date part (ET session date)',
    run: () => sessionKey({ time: '2025-03-10 09:30' }) === '2025-03-10' &&
        sessionKey({ time: '2025-03-10 20:00:00' }) === '2025-03-10'
},
{
    name: 'sessionKey converts UTC ISO timestamps to New York date',
    run: () => sessionKey({ time: '2025-03-10T20:00:00Z' }) === '2025-03-10' &&
        sessionKey({ time: '2025-03-11T01:00:00Z' }) === '2025-03-10'
},
{
    name: 'sessionKey prefers explicit sessionDate',
    run: () => sessionKey({ time: 'x', sessionDate: '2025-03-10' }) === '2025-03-10'
},
{
    name: 'getTimeframeBars returns only the latest session for 1D',
    run: () => {
        const bars = [
            { time: '2025-03-10 09:30', close: 1 },
            { time: '2025-03-10 16:00', close: 2 },
            { time: '2025-03-11 09:30', close: 3 }
        ];
        const result = getTimeframeBars(bars, '1D');
        return result.length === 2 && result[0].time === '2025-03-10 09:30';
    }
}
```

- [ ] **Step 2: Run standard verification** — expect FAIL (ReferenceError: `fiscalQuarterLabel` is not defined) listed in `[REGRESSION CHECKS] Failed`.
- [ ] **Step 3: Implement helpers at module scope.**

Insert after the `CORE_REGRESSION_TESTS` runner block's dependencies — i.e. define `fiscalQuarterLabel`/`sessionKey`/`getTimeframeBars` **before** `CORE_REGRESSION_TESTS` (after `calculateRangeSelection` helpers ~line 136):

```js
const fiscalQuarterLabel = date => {
    if (!date) return 'N/A';
    const parts = String(date).split('-');
    const year = parts[0];
    const month = Number(parts[1]);
    if (!year || !Number.isFinite(month)) return 'N/A';
    const quarter = month <= 3 ? 'Q1' : month <= 6 ? 'Q2' : month <= 9 ? 'Q3' : 'Q4';
    return `${quarter} ${String(year).slice(2)}`;
};

const EASTERN_DATE_FORMATTER = new Intl.DateTimeFormat('en-CA', {
    timeZone: 'America/New_York', year: 'numeric', month: '2-digit', day: '2-digit'
});

// Session bucketing key for a bar. Provider timestamps are either naive
// Eastern-time strings (Twelve Data, Alpha Vantage) or UTC ISO strings
// (Alpaca). Naive strings keep their date part (already ET session date);
// UTC strings are converted to the New York calendar date so after-hours
// bars stay in their true session regardless of the browser timezone.
const sessionKey = bar => {
    if (bar?.sessionDate) return String(bar.sessionDate);
    const time = String(bar?.time ?? '');
    const naive = time.match(/^(\d{4}-\d{2}-\d{2})/);
    const hasZone = /[zZ]$|[+-]\d{2}:?\d{2}$/.test(time);
    if (naive && !hasZone) return naive[1];
    const parsed = new Date(time);
    if (Number.isNaN(parsed.getTime())) return '';
    return EASTERN_DATE_FORMATTER.format(parsed);
};

// Canonical timeframe windows: 1D = latest session, 1W = latest five
// sessions, 1M/1Y = rolling calendar cutoffs (UTC, matching API stamps).
const getTimeframeBars = (bars, selectedTimeframe) => {
    if (!Array.isArray(bars) || !bars.length) return [];
    const sorted = [...bars].filter(v => v?.time && !Number.isNaN(new Date(v.time).getTime()))
        .sort((a, b) => new Date(a.time) - new Date(b.time));
    const sessions = [...new Set(sorted.map(v => sessionKey(v)))];
    if (selectedTimeframe === '1D') {
        const latest = sessions.at(-1);
        return sorted.filter(v => sessionKey(v) === latest);
    }
    if (selectedTimeframe === '1W') {
        const recentSessions = sessions.slice(-5);
        return sorted.filter(v => recentSessions.includes(sessionKey(v)));
    }
    const now = new Date();
    const cutoff = new Date(now);
    if (selectedTimeframe === '1M') cutoff.setUTCMonth(cutoff.getUTCMonth() - 1);
    if (selectedTimeframe === '1Y') cutoff.setUTCFullYear(cutoff.getUTCFullYear() - 1);
    return sorted.filter(v => {
        const date = new Date(v.time);
        return date >= cutoff && date <= now;
    });
};
```

- [ ] **Step 4: Move remaining constants/helpers to module scope.**

Move these **unchanged (with noted edits)** from the `Dashboard` body to module scope (place them right after the `Storage` helper ~line 375, before `mockFinancialsData`):

1. `timeframeConfig` (from ~1327) — move verbatim.
2. `CACHE_VERSION` — change `'v7'` → `'v8'` (invalidates caches written with corrupted EMA-era indicator values; done here so all later tasks write v8 caches).
3. `getCacheKey`, `readTimeframeCache`, `writeTimeframeCache` — move verbatim (they only use `Storage` + `timeframeConfig`, both module scope).
4. `QUOTE_CACHE_TTL_MS = 5 * 60 * 1000` (new) and `const getQuoteCacheKey = (provider, ticker) => \`quote_${String(provider).toLowerCase()}_${ticker}\`;`
5. `normalizeMarketBars`, `createMarketData` (from ~1381–1411) — move verbatim.
6. `FALLBACK_FUNDAMENTALS = Object.freeze({ cap: '150B', pe: '22.0' })`.
7. `STORAGE_KEYS`: add `PRICE_ALERT_NOTIFICATIONS: 'matrix_price_alert_notifications'` and replace the hardcoded string at ~1294 with `STORAGE_KEYS.PRICE_ALERT_NOTIFICATIONS`.

Then delete the now-duplicated definitions inside `Dashboard` and update call sites:
- `readTimeframeCache`/`writeTimeframeCache`/`getCacheKey`/`getTimeframeBars`/`normalizeMarketBars`/`createMarketData` calls inside `Dashboard` and loaders keep the same names (module scope is visible) — only the *definitions* are removed from `Dashboard`.
- Replace quote cache key template literals at (~1703, ~1924) with `getQuoteCacheKey(activeProvider, activeStock)`.
- Replace fallback literals `|| { cap: '150B', pe: '22.0' }` (~1620) and `|| { cap: '180B', pe: '25.0' }` (~1906) with `|| FALLBACK_FUNDAMENTALS` (both now identical: 150B/22.0).
- Delete dead `const cacheKey = getCacheKey('FAKE', ...)` (~1614).

- [ ] **Step 5: Fix quarter labels.** Replace the inline quarter logic in the Twelve Data path (~1788–1805):

```js
structuredLiveFinancials.quarter.performance = incQtrJson.income_statement.slice(0, 3).map(statement => ({
    period: fiscalQuarterLabel(statement.fiscal_date),
    revenue: parseFloat((parseFloat(statement.revenue || 0) / 1e9).toFixed(2)),
    netIncome: parseFloat((parseFloat(statement.net_income || 0) / 1e9).toFixed(2))
})).reverse();
```

And in the Alpha Vantage path (~1849–1855) delete the local `quarterLabel` function and replace both usages (~1858, ~1870) with `fiscalQuarterLabel(statement.fiscalDateEnding)` / `fiscalQuarterLabel(item.fiscalDateEnding)`.

- [ ] **Step 6: Add the debug hook.** At the end of the `Dashboard` body (after `handleSearchKeyDown`, before `return (`):

```js
// Test/debug hook read by automated browser checks. Contains no secrets.
React.useEffect(() => {
    window.__APP_DEBUG__ = {
        provider: activeProvider,
        chartData: renderedChartData,
        stockPrices,
        quoteInfo,
        syncStatusText,
        coreTests: CORE_REGRESSION_TEST_RESULTS,
        patternTests: CANDLE_PATTERN_TEST_RESULTS
    };
});
```

- [ ] **Step 7: Run standard verification** — expect `[REGRESSION CHECKS] 10/10 passed` (5 old + 5 new), no errors, app renders.

---

### Task 2: Indicator engine correctness (EMA warm-up, null warm-up windows, Wilder ATR, RSI)

**Goal:** Fix the HIGH bug (EMA warm-up corrupting MACD for ~1/3 of every chart) and align warm-up methodology with standard libraries.

**Files:**
- Modify: `index.html` (`calculateSMA`/`calculateEMA`/`calculateRSI`/`calculateBollingerBands`/`calculateTechnicalIndicators` ~442–565)

**Interfaces:**
- `calculateEMA(period, values) → (number|null)[]` — `null` before warm-up; handles `null` prefix in input (for MACD signal line).
- `calculateSMA(period, values) → (number|null)[]` — `null` until full window.
- `calculateRSI(closes, period=14) → (number|null)[]` — `null` until index `period`.
- `calculateATR(highs, lows, closes, period=14) → (number|null)[]` — Wilder RMA.
- `calculateTechnicalIndicators(data)` bars now carry `null` (not garbage numbers) for `ema*/sma*/bb*/atr/volumeSma20/rsi/macd/signal/hist` during warm-up; `trend` is `'NEUTRAL'` until EMAs exist.
- **Consumers:** chart already coerces non-finite `rsi/macd/signal/hist` to `50/0/0/0` for drawing (unchanged visuals during warm-up); Pattern Lab reads `Number(x) || 0` (unchanged).

- [ ] **Step 1: Add failing indicator tests** — new block `INDICATOR_TESTS` + runner, inserted immediately after `calculateTechnicalIndicators` (`~565`), mirroring the `CORE_REGRESSION_TESTS` runner pattern (try/catch per test, console.info/error with `[INDICATOR CHECKS]` prefix):

```js
const INDICATOR_TESTS = [
    {
        name: 'EMA warm-up emits null before seed',
        run: () => {
            const ema = calculateEMA(3, [1, 2, 3, 4, 5]);
            return ema[0] === null && ema[1] === null && ema[2] === 2 && ema[3] === 3 && ema[4] === 4;
        }
    },
    {
        name: 'EMA handles null prefix in input (MACD signal case)',
        run: () => {
            const ema = calculateEMA(2, [null, null, 4, 6]);
            return ema[0] === null && ema[1] === null && ema[2] === 4 && ema[3] === 5;
        }
    },
    {
        name: 'MACD/signal are null through warm-up, finite after',
        run: () => {
            const data = Array.from({ length: 60 }, (_, i) => ({
                open: 100 + Math.sin(i / 3), high: 101 + Math.sin(i / 3),
                low: 99 + Math.sin(i / 3), close: 100 + Math.sin(i / 3) + i * 0.01, volume: 1000
            }));
            const bars = calculateTechnicalIndicators(data);
            const firstFinite = bars.findIndex(b => Number.isFinite(b.macd));
            return bars.slice(0, 25).every(b => b.macd === null && b.signal === null && b.hist === null) &&
                firstFinite >= 25 &&
                Number.isFinite(bars[59].macd) && Number.isFinite(bars[59].signal) &&
                bars.every(b => b.rsi === null || (Number.isFinite(b.rsi) && b.rsi >= 0 && b.rsi <= 100));
        }
    },
    {
        name: 'SMA emits null until full window',
        run: () => {
            const sma = calculateSMA(3, [1, 2]);
            return sma.length === 2 && sma[0] === null && sma[1] === null;
        }
    },
    {
        name: 'ATR uses Wilder smoothing (constant range converges to range)',
        run: () => {
            const n = 40;
            const highs = Array.from({ length: n }, (_, i) => 102 + i * 0);
            const lows = Array.from({ length: n }, () => 100);
            const closes = Array.from({ length: n }, () => 101);
            const atr = calculateATR(highs, lows, closes, 14);
            return atr[12] === null && atr[13] === 2 && atr[39] === 2;
        }
    },
    {
        name: 'RSI is null during warm-up',
        run: () => {
            const closes = Array.from({ length: 20 }, (_, i) => 100 + Math.sin(i));
            const rsi = calculateRSI(closes, 14);
            return rsi.slice(0, 14).every(v => v === null) && Number.isFinite(rsi[14]);
        }
    },
    {
        name: 'Bollinger bands null until period satisfied',
        run: () => {
            const bands = calculateBollingerBands([1, 2, 3], 20, 2);
            return bands.every(b => b === null);
        }
    }
];
```

Runner (same pattern as line 171–186, logging `[INDICATOR CHECKS]`).

- [ ] **Step 2: Run standard verification** — expect `[INDICATOR CHECKS] Failed` (EMA/SMA/ATR/RSI/BB tests fail; `calculateATR` undefined).

- [ ] **Step 3: Implement.**

Replace `calculateSMA`:

```js
const calculateSMA = (period, values) =>
    values.map((_, index) =>
        index < period - 1 ? null : average(values.slice(index - period + 1, index + 1))
    );
```

Replace `calculateEMA` (null warm-up, null-prefix tolerant):

```js
const calculateEMA = (period, values) => {
    if (!values.length) return [];
    const k = 2 / (period + 1);
    const ema = new Array(values.length).fill(null);
    let start = 0;
    while (start < values.length && !Number.isFinite(values[start])) start++;
    const available = values.length - start;
    if (available <= 0) return ema;
    const seedIndex = start + Math.min(period, available) - 1;
    ema[seedIndex] = average(values.slice(start, seedIndex + 1));
    for (let i = seedIndex + 1; i < values.length; i++) {
        if (!Number.isFinite(values[i]) || !Number.isFinite(ema[i - 1])) continue;
        ema[i] = values[i] * k + ema[i - 1] * (1 - k);
    }
    return ema;
};
```

Replace `calculateRSI` — change `new Array(closes.length).fill(50)` → `fill(null)` (rest of the function unchanged; the `closes.length <= period` early-return then yields all-null; `rsi[period]` onward computed as before).

Replace `calculateBollingerBands`:

```js
const calculateBollingerBands = (closes, period = 20, standardDeviations = 2) =>
    closes.map((_, index) => {
        if (index < period - 1) return null;
        const window = closes.slice(index - period + 1, index + 1);
        const middle = average(window);
        const standardDeviation = Math.sqrt(
            average(window.map(value => (value - middle) ** 2))
        );
        return {
            middle,
            upper: middle + standardDeviations * standardDeviation,
            lower: middle - standardDeviations * standardDeviation
        };
    });
```

Add Wilder ATR after `calculateTrueRange`:

```js
const calculateATR = (highs, lows, closes, period = 14) => {
    const atr = new Array(closes.length).fill(null);
    if (closes.length < period) return atr;
    const trueRange = calculateTrueRange(closes, highs, lows);
    atr[period - 1] = average(trueRange.slice(0, period));
    for (let i = period; i < closes.length; i++) {
        atr[i] = (atr[i - 1] * (period - 1) + trueRange[i]) / period;
    }
    return atr;
};
```

Update `calculateTechnicalIndicators` body:

```js
const macdLine = ema12.map((value, index) =>
    Number.isFinite(value) && Number.isFinite(ema26[index]) ? value - ema26[index] : null
);
const signalLine = calculateEMA(9, macdLine);
const rsi = calculateRSI(closes, 14);
const atr = calculateATR(highs, lows, closes, 14);
const sma20 = calculateSMA(20, closes);
const sma50 = calculateSMA(50, closes);
const volumeSma20 = calculateSMA(20, volumes);
const bands = calculateBollingerBands(closes, 20, 2);
const round3 = value => Number.isFinite(value) ? +value.toFixed(3) : null;

return data.map((d, index) => ({
    ...d,
    ema9: round3(ema9[index]),
    ema21: round3(ema21[index]),
    ema50: round3(ema50[index]),
    sma20: round3(sma20[index]),
    sma50: round3(sma50[index]),
    bbMiddle: bands[index] ? round3(bands[index].middle) : null,
    bbUpper: bands[index] ? round3(bands[index].upper) : null,
    bbLower: bands[index] ? round3(bands[index].lower) : null,
    atr: round3(atr[index]),
    volumeSma20: Number.isFinite(volumeSma20[index]) ? Math.round(volumeSma20[index]) : null,
    macd: round3(macdLine[index]),
    signal: round3(signalLine[index]),
    hist: Number.isFinite(macdLine[index]) && Number.isFinite(signalLine[index])
        ? round3(macdLine[index] - signalLine[index])
        : null,
    rsi: Number.isFinite(rsi[index]) ? +rsi[index].toFixed(2) : null,
    trend: Number.isFinite(ema21[index]) && Number.isFinite(ema50[index])
        ? (ema21[index] >= ema50[index] ? 'BULLISH' : 'BEARISH')
        : 'NEUTRAL'
}));
```

Note: `calculateEMA` is already used for `ema9/21/50` — with the new null behavior those become null during warm-up (intended).

- [ ] **Step 4: Run standard verification** — `[INDICATOR CHECKS] 7/7 passed`, `[REGRESSION CHECKS] 10/10 passed`, no console errors (verify no `.toFixed` of null anywhere by loading the chart — errors would surface as SVG/console errors).

---

### Task 3: Candlestick detector fixes — tweezer span, doji guard, fixture coverage

**Goal:** Fix tweezer detection to use the adjacent candle pair (and thus highlight the right span), prevent exact-doji hammers, and close the fixture coverage gap.

**Files:**
- Modify: `index.html` (`detectCandlestickPatterns` tweezer block ~851–870; hammer/shooting-star guards ~919–939; `CANDLE_PATTERN_TESTS` ~996)

**Interfaces:**
- Detector output unchanged: `{ startIndex, endIndex, type, label, confidence, context }`. Tweezer `startIndex` is now the first candle of the matched adjacent pair.

- [ ] **Step 1: Add failing fixtures** to `CANDLE_PATTERN_TESTS`:

```js
{
    name: 'TWEEZER BOTTOM',
    expected: 'TWEEZER BOTTOM',
    candles: [
        {open:110, high:111, low:108, close:108.5, volume:1000},
        {open:108.5, high:109, low:105.5, close:106, volume:1000},
        {open:106, high:106.5, low:103, close:103.5, volume:1000},
        {open:103.5, high:104, low:100.5, close:101, volume:1000},
        {open:101, high:101.5, low:99, close:99.5, volume:1300},
        {open:100.2, high:102, low:99, close:101.8, volume:1300}
    ]
},
{
    name: 'TWEEZER TOP',
    expected: 'TWEEZER TOP',
    candles: [
        {open:100, high:101, low:99, close:100.5, volume:1000},
        {open:100.5, high:102, low:100, close:101.8, volume:1000},
        {open:101.8, high:103.5, low:101.2, close:103, volume:1000},
        {open:103, high:104.5, low:102.5, close:104, volume:1000},
        {open:104, high:106, low:103.8, close:105.5, volume:1300},
        {open:105.2, high:106, low:103.8, close:104, volume:1300}
    ]
},
{
    name: 'MORNING STAR',
    expected: 'MORNING STAR',
    candles: [
        {open:112, high:113, low:110, close:110.5, volume:1000},
        {open:110.5, high:111, low:108, close:108.5, volume:1000},
        {open:108.5, high:109, low:106, close:106.5, volume:1000},
        {open:106.5, high:107, low:102, close:102.5, volume:1300},
        {open:102.6, high:103.5, low:101.5, close:102.8, volume:1000},
        {open:103, high:107, low:102.8, close:106.5, volume:1400}
    ]
},
{
    name: 'THREE WHITE SOLDIERS',
    expected: 'THREE WHITE SOLDIERS',
    candles: [
        {open:112, high:113, low:110, close:110.5, volume:1000},
        {open:110.5, high:111, low:107.5, close:108, volume:1000},
        {open:108, high:108.5, low:105, close:105.5, volume:1000},
        {open:105.6, high:109.5, low:105.4, close:109, volume:1300},
        {open:108.8, high:112.5, low:108.6, close:112, volume:1300},
        {open:111.8, high:115.5, low:111.6, close:115, volume:1300}
    ]
},
{
    name: 'exact doji is not a HAMMER',
    expected: 'HAMMER',
    expectAbsent: true,
    candles: [
        {open:110, high:111, low:109, close:109, volume:1000},
        {open:109, high:109.5, low:107, close:107.5, volume:1000},
        {open:107.5, high:108, low:105, close:105.5, volume:1000},
        {open:105, high:105, low:101, close:105, volume:1300}
    ]
}
```

Update the runner `runCandlestickPatternSelfTests`:

```js
const runCandlestickPatternSelfTests = () => CANDLE_PATTERN_TESTS.map(test => {
    try {
        const matches = detectCandlestickPatterns(test.candles).some(p => p.label === test.expected);
        return { ...test, passed: test.expectAbsent ? !matches : matches };
    } catch (error) {
        return { ...test, passed: false, error: error?.message || String(error) };
    }
});
```

Also wrap the module-load invocation (Task 5 will harden; do it now since we're here):

```js
let CANDLE_PATTERN_TEST_RESULTS;
try {
    CANDLE_PATTERN_TEST_RESULTS = runCandlestickPatternSelfTests();
} catch (error) {
    CANDLE_PATTERN_TEST_RESULTS = [{ name: 'self-test runner', expected: 'n/a', passed: false, error: error?.message }];
}
```

- [ ] **Step 2: Run standard verification** — expect `[CANDLE PATTERN TESTS]` failures for TWEEZER BOTTOM/TOP, MORNING STAR, THREE WHITE SOLDIERS, doji (current detector matches p2/c0 tweezers → TWEEZER fixtures fail; doji passes as HAMMER → `expectAbsent` fails).

- [ ] **Step 3: Implement detector fixes.**

Tweezer block — match the **adjacent pair** `p1`/`c0` (span `i-1..i`, already what `addPattern(i - 1, i, ...)` highlights):

```js
const tweezerTolerance = Math.max(atr * 0.20, 0.01);
const tweezerBottom =
    threeCandleTrend === 'DOWN' &&
    p1.bearish && c0.bullish &&
    sameLevel(p1.low, c0.low, tweezerTolerance) &&
    c0.lowerWick >= c0.body * 0.5;
const tweezerTop =
    threeCandleTrend === 'UP' &&
    p1.bullish && c0.bearish &&
    sameLevel(p1.high, c0.high, tweezerTolerance) &&
    c0.upperWick >= c0.body * 0.5;
```

Hammer / shooting star — add a real-body guard (excludes exact dojis):

```js
const hasRealBody = bodyPct > 0.01;
if (
    trend === 'DOWN' && hasRealBody &&
    lowerWick >= body * 2 &&
    upperWick <= body * 0.35 &&
    bodyPct <= 0.35
) { ... }
```

and the mirrored condition for shooting star (`trend === 'UP' && hasRealBody && ...`).

- [ ] **Step 4: Run standard verification** — `[CANDLE PATTERN TESTS] 11/11 passed` (6 old + 5 new), `[INDICATOR CHECKS] 7/7`, `[REGRESSION CHECKS] 10/10`, no errors.

---

### Task 4: Demo (FAKE) mode must compute indicators

**Goal:** Default no-API-key experience gets real RSI/MACD instead of flat 50/0 lines.

**Files:**
- Modify: `index.html` demo branch of `fetchPipelineData` (~1612–1634)

- [ ] **Step 1:** In the demo loop, recompute indicators after normalization (normalizeMarketBars strips them):

```js
if (!hasLiveCredentials) {
    for (let ticker of watchlist) {
        const marketData = createMarketData(ticker, generateForcedFakeData(ticker, timeframe), 'FAKE');
        const processedData = calculateTechnicalIndicators(marketData.bars);
        writeTimeframeCache('FAKE', ticker, timeframe, processedData);
        const currentCandle = processedData[processedData.length - 1];
        const staticMeta = TICKER_DICTIONARY.find(i => i.ticker === ticker) || FALLBACK_FUNDAMENTALS;
        localPrices[ticker] = {
            price: currentCandle.close,
            change: ((currentCandle.close - processedData[0].close) / processedData[0].close) * 100,
            volume: '1.2M',
            cap: staticMeta.cap,
            pe: staticMeta.pe
        };
        if (ticker === activeStock) setRenderedChartData(processedData);
    }
    ...
```

(Note: `generateForcedFakeData` still calls `calculateTechnicalIndicators` internally at line 695 — remove that call inside it so indicators are computed once, after normalization: change `return calculateTechnicalIndicators(data);` → `return data;`. Its output feeds only `createMarketData`.)

- [ ] **Step 2: Run standard verification**, then assert via debug hook:

```js
() => {
    const d = window.__APP_DEBUG__;
    const bars = d.chartData;
    const rsiValues = new Set(bars.map(b => b.rsi));
    const macdFinite = bars.filter(b => Number.isFinite(b.macd) && b.macd !== 0).length;
    return { provider: d.provider, bars: bars.length, distinctRsi: rsiValues.size, macdNonZero: macdFinite };
}
```

Expect: `provider: 'DEMO'`, `distinctRsi > 5`, `macdNonZero > 10` (before this task: `distinctRsi === 1` (50) and `macdNonZero === 0`).

---

### Task 5: Error-handling hardening (unhandled rejections, notification guards)

**Goal:** Zero unhandled promise rejections; module-load tests cannot white-screen the app.

**Files:**
- Modify: `index.html` (pipeline effect ~1933, `handleManualRefresh` ~1938, `requestNotificationPermission` ~1286, `evaluatePriceAlerts` ~1292, pattern test runner — done in Task 3 Step 1)

- [ ] **Step 1:** Pipeline effect — attach a catch (pipeline has internal try/catch; the catch covers setup/teardown code outside it):

```js
React.useEffect(() => {
    fetchPipelineData().catch(error => {
        console.error('[PIPELINE] refresh failed:', error);
        setSyncStatusText('API ERROR: PIPELINE FAILURE');
    });
}, [activeStock, timeframe, apiKey, alphaVantageKey, alpacaKeyId, alpacaSecretKey, activeProvider, watchlist]);
```

- [ ] **Step 2:** Manual refresh — add catch:

```js
const handleManualRefresh = React.useCallback(async () => {
    if (isRefreshing) return;
    setIsRefreshing(true);
    try {
        await fetchPipelineData(true);
    } catch (error) {
        console.error('[PIPELINE] manual refresh failed:', error);
        setSyncStatusText(`API ERROR: ${String(error?.message || 'UNKNOWN').slice(0, 20)}`);
    } finally {
        setIsRefreshing(false);
    }
}, [fetchPipelineData, isRefreshing]);
```

- [ ] **Step 3:** Notifications — guard both functions:

```js
const requestNotificationPermission = async () => {
    if (typeof Notification === 'undefined') return;
    try {
        const result = await Notification.requestPermission();
        setNotificationStatus(result);
    } catch (error) {
        console.warn('[NOTIFICATIONS] permission request failed:', error);
    }
};
```

In `evaluatePriceAlerts`, wrap the `new Notification(...)` construction:

```js
if (directionMatch && proximity <= 0.05 && !notified[key]) {
    try {
        new Notification(`${alert.ticker} price alert`, {
            body: `${alert.ticker} is ${current.toFixed(2)} (${alert.direction.toLowerCase()} ${target.toFixed(2)}).`,
            tag: key
        });
        notified[key] = Date.now();
    } catch (error) {
        console.warn('[NOTIFICATIONS] display failed:', error);
    }
} else if (proximity > 0.05 || !directionMatch) {
    delete notified[key];
}
```

- [ ] **Step 4: Run standard verification** — no errors, no unhandled rejections (also check `playwright_browser_console_messages` for `unhandledrejection` after interacting: click Refresh button once).

---

### Task 6: Fetch resilience — rate limits, per-ticker isolation, missing-only refetch, provider fixes

**Goal:** Handle 429s with backoff; one bad ticker no longer aborts the watchlist; only missing tickers are fetched; Alpaca returns newest bars; Twelve Data batches are chunked and status-checked; Alpha Vantage reuses the canonical interval map.

**Files:**
- Modify: `index.html` (loader functions ~1425–1592, `fetchPipelineData` ~1594–1931)

**Interfaces (Produces):**
- Module scope: `sleep(ms)`, `isRateLimitError(error)`, `fetchJsonWithRetry(url, options, { retries, baseDelayMs }) → { res, json }` (retries only on rate-limit/network errors; throws rate-limit `Error` tagged `error.isRateLimit = true` after retries).
- Loaders now accept `watchlist` = **tickers to fetch** (already filtered by caller) and never throw on individual ticker failures (log + continue; abort loop early on rate-limit).

- [ ] **Step 1: Add retry/rate-limit helpers** at module scope (near `Storage`):

```js
const sleep = ms => new Promise(resolve => setTimeout(resolve, ms));
const RATE_LIMIT_PATTERN = /429|rate.?limit|too many requests|call frequency|request frequency|premium plan|thank you for using/i;
const isRateLimitError = error => Boolean(error && (error.isRateLimit || RATE_LIMIT_PATTERN.test(String(error.message || ''))));

// Fetch JSON with bounded exponential backoff for rate-limit responses.
// Non-rate-limit HTTP responses are returned to the caller for inspection.
const fetchJsonWithRetry = async (url, options = {}, { retries = 2, baseDelayMs = 1500 } = {}) => {
    let attempt = 0;
    for (;;) {
        let res, json;
        try {
            res = await fetch(url, options);
            json = await res.json().catch(() => null);
        } catch (networkError) {
            if (attempt >= retries) throw networkError;
            await sleep(baseDelayMs * 2 ** attempt++);
            continue;
        }
        const rateLimited = res.status === 429 ||
            (json && RATE_LIMIT_PATTERN.test(String(json.Note || json.Information || json.message || json.status || '')));
        if (rateLimited) {
            if (attempt >= retries) {
                const error = new Error(`RATE_LIMIT: ${url.split('?')[0]}`);
                error.isRateLimit = true;
                throw error;
            }
            const retryAfter = Number(res.headers?.get?.('Retry-After'));
            await sleep(Number.isFinite(retryAfter) && retryAfter > 0 ? retryAfter * 1000 : baseDelayMs * 2 ** attempt++);
            continue;
        }
        return { res, json };
    }
};
```

- [ ] **Step 2: Restructure `fetchPipelineData` for missing-only fetching.** Replace the `allCached` block (~1601–1609) with:

```js
const cacheNamespace = hasLiveCredentials ? activeProvider : 'FAKE';
const missingTickers = watchlist.filter(ticker => !readTimeframeCache(cacheNamespace, ticker, timeframe));
const needsMarketData = forceRefresh || missingTickers.length > 0;
```

Replace the network gate (~1639) with `if (needsMarketData || needsActiveQuote) {` and inside it, guard each loader call with `if (needsMarketData) {` passing `watchlist: forceRefresh ? watchlist : missingTickers`:

```js
if (needsMarketData) {
    if (activeProvider === 'ALPACA') {
        await loadAlpacaMarketData({ watchlist: forceRefresh ? watchlist : missingTickers, ... });
    } else if (activeProvider === 'ALPHAVANTAGE') {
        await loadAlphaVantageMarketData({ watchlist: forceRefresh ? watchlist : missingTickers, ... });
    } else {
        await loadTwelveDataMarketData({ watchlist: forceRefresh ? watchlist : missingTickers, ... });
    }
}
```

Also replace `needsActiveQuote = !quoteInfo[activeStock]` (~1638) with cache freshness (drops reliance on possibly-stale state):

```js
const quoteCacheKey = getQuoteCacheKey(activeProvider, activeStock);
const quoteCachedEarly = Storage.getJSON(quoteCacheKey);
const quoteIsFresh = quoteCachedEarly && Date.now() - quoteCachedEarly.timestamp < QUOTE_CACHE_TTL_MS;
const needsActiveQuote = !quoteIsFresh;
```

(profile/quote/fundamentals code between the loader call and the catch stays inside the same `if (needsMarketData || needsActiveQuote)` block; reuse `quoteCacheKey` and the later TTL check with `QUOTE_CACHE_TTL_MS`.)

- [ ] **Step 3: Per-ticker isolation + rate-limit abort in Alpaca loader.** Wrap the body of the `for (let ticker of watchlist)` loop (~1437–1504) in:

```js
for (let ticker of watchlist) {
    try {
        ... existing body ...
    } catch (error) {
        if (isRateLimitError(error)) {
            console.warn(`[ALPACA] rate limited, aborting watchlist sync: ${error.message}`);
            setSyncStatusText('RATE LIMITED: WAIT AND REFRESH');
            break;
        }
        console.warn(`[ALPACA] skipping ${ticker}:`, error.message);
    }
}
```

But first convert the loader's fetch to `fetchJsonWithRetry` — replace the raw `fetch` + manual `res.json()` (~1448–1459):

```js
const { res: alpacaRes, json: alpacaJson } = await fetchJsonWithRetry(alpacaUrl, {
    headers: {
        'APCA-API-KEY-ID': alpacaKeyId,
        'APCA-API-SECRET-KEY': alpacaSecretKey,
        'Accept': 'application/json'
    },
    cache: 'no-store'
});
if (!alpacaRes.ok || alpacaJson?.code || alpacaJson?.error) {
    throw new Error(alpacaJson?.message || alpacaJson?.error || `Alpaca request failed for ${ticker} (${alpacaRes.status})`);
}
```

- [ ] **Step 4: Alpaca — newest bars first + pagination.** Replace URL construction (~1447) and raw-bar collection:

```js
let pageToken = '';
let rawBars = [];
let pages = 0;
do {
    const alpacaUrl = `https://data.alpaca.markets/v2/stocks/${encodeURIComponent(ticker)}/bars` +
        `?timeframe=${alpacaTimeframe}&start=${encodeURIComponent(startIso)}&end=${encodeURIComponent(endIso)}` +
        `&limit=${alpacaLimit}&sort=desc&feed=iex&adjustment=all` +
        (pageToken ? `&page_token=${encodeURIComponent(pageToken)}` : '');
    const { res: alpacaRes, json: alpacaJson } = await fetchJsonWithRetry(alpacaUrl, { headers: {...}, cache: 'no-store' });
    if (!alpacaRes.ok || alpacaJson?.code || alpacaJson?.error) {
        throw new Error(alpacaJson?.message || alpacaJson?.error || `Alpaca request failed for ${ticker} (${alpacaRes.status})`);
    }
    const pageBars = Array.isArray(alpacaJson.bars)
        ? alpacaJson.bars
        : (alpacaJson.bars && Array.isArray(alpacaJson.bars[ticker]) ? alpacaJson.bars[ticker] : []);
    rawBars.push(...pageBars);
    pageToken = alpacaJson.next_page_token || '';
} while (pageToken && rawBars.length < alpacaLimit && ++pages < 5);

// desc + pagination returns newest-first pages; normalize then restore ascending order.
const normalizedBars = normalizeMarketBars(rawBars.map(v => ({
    time: v.t ?? v.timestamp ?? v.time,
    open: v.o ?? v.open,
    high: v.h ?? v.high,
    low: v.l ?? v.low,
    close: v.c ?? v.close,
    volume: v.v ?? v.volume ?? 0
})));
```

Note: `normalizeMarketBars` sorts ascending internally and validates OHLC — this **replaces** the inline duplicated map/filter/sort (~1466–1478). Because pages arrive newest-first, reverse `rawBars` **before** normalize only if needed for readability; normalize's sort makes order irrelevant — drop the concern entirely (normalize sorts ascending).

Also remove `throw new Error([object Object])` risk (covered: `.message || .error || status`).

- [ ] **Step 5: Alpha Vantage loader** — isolation + canonical interval + helper dedupe:
- Replace inline interval map (~1523) with: `const interval = selectedTimeframeConfig.tdInterval;` (values `5min/15min/30min/1day` are valid AV intervals for intraday; 1Y uses `TIME_SERIES_DAILY` without interval).
- Replace the inline map/filter/sort (~1535–1547) with `const normalizedAlphaVantage = normalizeMarketBars(Object.entries(series).map(([timestamp, v]) => ({ time: timestamp, sessionDate: timestamp.slice(0, 10), open: v['1. open'], high: v['2. high'], low: v['3. low'], close: v['4. close'], volume: v['5. volume'] || 0 })));`
- Wrap per-ticker body in try/catch identical to Step 3 pattern (prefix `[ALPHAVANTAGE]`), abort loop on `isRateLimitError`.
- `fetchAlphaVantageJson` (~1413): switch to `fetchJsonWithRetry`:

```js
const fetchAlphaVantageJson = async (params) => {
    const query = new URLSearchParams({ ...params, apikey: alphaVantageKey });
    const { res, json } = await fetchJsonWithRetry(`https://www.alphavantage.co/query?${query.toString()}`, { cache: 'no-store' });
    if (!res.ok || json?.['Error Message'] || json?.Note || json?.Information) {
        const error = new Error(json?.['Error Message'] || json?.Note || json?.Information || `Alpha Vantage request failed (${res.status})`);
        if (RATE_LIMIT_PATTERN.test(error.message)) error.isRateLimit = true;
        throw error;
    }
    return json;
};
```

(`fetchAlphaVantageJson` is used by search/quote/fundamentals too — all gain retry.)

- [ ] **Step 6: Twelve Data loader** — chunking, `res.ok`, `cache: 'no-store'`:

```js
const TD_MAX_SYMBOLS = 8; // free-tier batch cap
for (let offset = 0; offset < watchlist.length; offset += TD_MAX_SYMBOLS) {
    const chunk = watchlist.slice(offset, offset + TD_MAX_SYMBOLS);
    const symbolsParam = chunk.join(',');
    const url = `https://api.twelvedata.com/time_series?symbol=${symbolsParam}&interval=${selectedInterval}&outputsize=${selectedTimeframeConfig.outputsize}&apikey=${apiKey}`;
    const { res, json: resJson } = await fetchJsonWithRetry(url, { cache: 'no-store' });
    if (!res.ok || resJson?.status === 'error') throw new Error(resJson?.message || `Twelve Data request failed (${res.status})`);
    for (let ticker of chunk) {
        const tickerData = chunk.length === 1 ? resJson : resJson[ticker];
        if (tickerData && tickerData.values) {
            const marketData = createMarketData(ticker, tickerData.values.map(v => ({
                time: v.datetime, open: v.open, high: v.high, low: v.low, close: v.close, volume: v.volume || 0
            })), 'TWELVEDATA');
            const visibleTwelveData = getTimeframeBars(marketData.bars, timeframe).slice(-selectedTimeframeConfig.maxCandles);
            const processedData = calculateTechnicalIndicators(visibleTwelveData);
            writeTimeframeCache('TWELVEDATA', ticker, timeframe, processedData);
        } else {
            console.warn(`[TWELVEDATA] no series returned for ${ticker}`);
        }
    }
}
```

(This also deletes the dead `cacheKey` line with the stale `v6` prefix.)

- [ ] **Step 7: Search path** — `searchLiveSymbols` Twelve Data fetch (~1963) → `fetchJsonWithRetry` keeping existing error handling; Alpha Vantage search already goes through `fetchAlphaVantageJson` ✓.

- [ ] **Step 8: Run standard verification** (demo mode — network paths not exercised) + add a **manual fetch-path sanity note** in the task output; live-API paths verified in Task 16 only if credentials exist (they don't — verify by code review + demo-path regression checks).

---

### Task 7: State correctness — stale closures, quote vs window return ordering, search re-render

**Goal:** Watchlist price rows and quote panel stop clobbering each other; effect calls the freshest pipeline; search keystrokes cause minimal renders.

**Files:**
- Modify: `index.html` (`fetchPipelineData` localPrices handling ~1599/1751/1928, pipeline effect ~1933, search effect ~2037–2093)

- [ ] **Step 1: Functional merge + quote applied last.**
- Change `const localPrices = { ...stockPrices };` (~1599) → `const localPrices = {};` (accumulator of *updated* tickers only).
- Change final `setStockPrices(localPrices);` (~1928) → `setStockPrices(prev => ({ ...prev, ...localPrices }));`
- Demo branch `setStockPrices(localPrices)` (~1630) → same functional merge.
- Quote write: capture instead of applying immediately. Before the try block add `let activeQuoteUpdate = null;` and change the quote application (~1749–1762) to:

```js
if (quoteJson && (!quoteJson.status || quoteJson.status !== 'error')) {
    setQuoteInfo(prev => ({ ...prev, [activeStock]: quoteJson }));
    activeQuoteUpdate = {
        price: Number(quoteJson.close || quoteJson.price || 0),
        change: Number(quoteJson.percent_change || 0),
        volume: quoteJson.volume || 'N/A'
    };
}
```

- After the cache loop's merge (`setStockPrices(prev => ({ ...prev, ...localPrices }))`), apply the quote last so the DAY CHANGE panel and watchlist agree for the active ticker:

```js
if (activeQuoteUpdate) {
    setStockPrices(prev => ({
        ...prev,
        [activeStock]: { ...(prev[activeStock] || {}), ...activeQuoteUpdate }
    }));
}
```

(React 18 batches both functional updates in order within the same async tick.)

- [ ] **Step 2: Call freshest pipeline via ref.** After `fetchPipelineData` definition:

```js
const fetchPipelineRef = React.useRef(fetchPipelineData);
fetchPipelineRef.current = fetchPipelineData;
```

Effect becomes:

```js
React.useEffect(() => {
    fetchPipelineRef.current?.().catch(error => {
        console.error('[PIPELINE] refresh failed:', error);
        setSyncStatusText('API ERROR: PIPELINE FAILURE');
    });
}, [activeStock, timeframe, apiKey, alphaVantageKey, alpacaKeyId, alpacaSecretKey, activeProvider, watchlist]);
```

(Keeps effect deps free of `fetchPipelineData`/`fundamentals`/`quoteInfo` — no loops — while always invoking the latest closure.)

- [ ] **Step 3: Search suggestions — bail out on identical content.** Add helper near other helpers (module scope):

```js
const sameSuggestionList = (a, b) =>
    a.length === b.length && a.every((item, index) => item?.ticker === b[index]?.ticker && item?.source === b[index]?.source);
```

In the search effect, wrap both `setSuggestions` calls (~2062 and the debounce body ~2080):

```js
setSuggestions(prev => sameSuggestionList(prev, localMatches.slice(0, 8)) ? prev : localMatches.slice(0, 8));
```

and for the merge path:

```js
setSuggestions(prev => {
    const merged = [...prev];
    liveMatches.forEach(item => {
        if (!merged.some(existing => existing.ticker === item.ticker)) merged.push(item);
    });
    const next = merged.slice(0, 8);
    return sameSuggestionList(prev, next) ? prev : next;
});
```

- [ ] **Step 4: Run standard verification** — 0 errors; `__APP_DEBUG__.stockPrices` populated in demo; typing in search (evaluate-driven) doesn't error.

---

### Task 8: Security — pin CDN versions with SRI, referrer policy, provider-key note

**Goal:** Supply-chain hardening for the four CDN scripts; document query-string key limitation.

**Files:**
- Modify: `index.html` head (~11–14), meta (~4–9), API panel note (~2254)

- [ ] **Step 1: Determine current pinned versions and compute SRI hashes:**

```bash
curl -sI https://unpkg.com/react@18/umd/react.production.min.js | grep -i location
# then for each pinned URL:
curl -s <URL> | openssl dgst -sha384 -binary | openssl base64 -A; echo
```

Pin exact versions (expected: react@18.3.1, react-dom@18.3.1; find current `@babel/standalone` and `@tailwindcss/browser@4` versions via unpkg/jsdelivr redirect headers).

- [ ] **Step 2: Update head:**

```html
<script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4.1.13" integrity="sha384-<HASH>" crossorigin="anonymous"></script>
<script src="https://unpkg.com/react@18.3.1/umd/react.production.min.js" integrity="sha384-<HASH>" crossorigin="anonymous"></script>
<script src="https://unpkg.com/react-dom@18.3.1/umd/react-dom.production.min.js" integrity="sha384-<HASH>" crossorigin="anonymous"></script>
<script src="https://unpkg.com/@babel/standalone@7.28.4/babel.min.js" integrity="sha384-<HASH>" crossorigin="anonymous"></script>
```

(exact versions/hashes from Step 1 — do NOT guess). Add `<meta name="referrer" content="no-referrer">` after the existing meta tags.

- [ ] **Step 3: API panel note** — extend the note text (~2254):

```jsx
<div className="text-[10px] font-mono text-zinc-500">If both Alpaca fields are provided, Alpaca is used. Otherwise Alpha Vantage is used when its key is present, then Twelve Data.<br />Credentials are stored in this browser's localStorage. Alpha Vantage and Twelve Data require the key in the request URL (provider limitation); Alpaca uses request headers.</div>
```

- [ ] **Step 4: Run standard verification** — CRITICAL: if SRI fails the app will not boot (integrity error in console). Expect clean load; if integrity error → re-check hash computation (hash the exact downloaded bytes, `curl -s` without `-L` mismatch).

---

### Task 9: Chart correctness + performance

**Goal:** Memoize expensive chart computation, consistent `chartData` indices, rAF-throttled hover, window-level mouseup, strict swing comparisons, stable chart key.

**Files:**
- Modify: `index.html` (`CandlestickChart` ~2483–2843, chart mount key ~2376)

**Interfaces:**
- `CandlestickChart({ vectorData, theme, showPatterns })` — now wrapped in `React.memo`. Internal: `hoverState = { node, index } | null`.

- [ ] **Step 1: Restructure with hooks before early returns** — replace the top of the component (~2483–2507):

```js
const CandlestickChart = React.memo(function CandlestickChart({ vectorData, theme, showPatterns }) {
    const [hoverState, setHoverState] = React.useState(null);
    const [rangeSelection, setRangeSelection] = React.useState(null);
    const [isSelectingRange, setIsSelectingRange] = React.useState(false);
    const hoverFrameRef = React.useRef(null);
    const pendingHoverRef = React.useRef(null);

    const chartData = React.useMemo(() => {
        if (!Array.isArray(vectorData)) return [];
        return vectorData
            .filter(d => [d.open, d.high, d.low, d.close].every(Number.isFinite))
            .map(d => ({
                ...d,
                rsi: Number.isFinite(d.rsi) ? d.rsi : 50,
                macd: Number.isFinite(d.macd) ? d.macd : 0,
                signal: Number.isFinite(d.signal) ? d.signal : 0,
                hist: Number.isFinite(d.hist) ? d.hist : 0
            }));
    }, [vectorData]);

    const chartModel = React.useMemo(() => buildChartModel(chartData), [chartData]);
    const identifiedPatterns = React.useMemo(
        () => (showPatterns ? detectCandlestickPatterns(chartData) : []),
        [showPatterns, chartData]
    );
    const levels = React.useMemo(
        () => computeSupportResistance(chartData, chartModel.priceDelta),
        [chartData, chartModel.priceDelta]
    );
    const { supportLevels, resistanceLevels } = levels;

    // Coalesce per-candle mouse moves into at most one state update per frame.
    const scheduleHover = React.useCallback((node, index) => {
        pendingHoverRef.current = { node, index };
        if (hoverFrameRef.current != null) return;
        hoverFrameRef.current = requestAnimationFrame(() => {
            hoverFrameRef.current = null;
            setHoverState(pendingHoverRef.current);
        });
    }, []);

    React.useEffect(() => () => {
        if (hoverFrameRef.current != null) cancelAnimationFrame(hoverFrameRef.current);
    }, []);

    // Releasing the mouse anywhere ends range selection, not only over a candle rect.
    React.useEffect(() => {
        if (!isSelectingRange) return undefined;
        const endSelection = () => setIsSelectingRange(false);
        window.addEventListener('mouseup', endSelection);
        return () => window.removeEventListener('mouseup', endSelection);
    }, [isSelectingRange]);

    if (!vectorData || vectorData.length === 0) {
        return <div className="h-full flex items-center justify-center font-mono text-xs text-zinc-700 animate-pulse">CONNECTING INTERFACE PIPELINES...</div>;
    }
    if (chartData.length === 0) {
        return <div className="h-full flex items-center justify-center font-mono text-xs text-zinc-700 animate-pulse">NO VALID MARKET DATA TO RENDER...</div>;
    }

    const hoverNode = hoverState?.node ?? null;
    const hoverIndex = hoverState?.index ?? -1;
    ...
```

- [ ] **Step 2: Extract pure helpers to module scope** (above `CandlestickChart`):

```js
const buildChartModel = (data) => {
    const highLowPrices = data.map(d => [d.high, d.low]).flat();
    const maxPrice = (highLowPrices.length ? Math.max(...highLowPrices) : 1) * 1.01;
    const minPrice = (highLowPrices.length ? Math.min(...highLowPrices) : 0) * 0.99;
    const priceDelta = maxPrice - minPrice || 1;
    const macdIndicators = data.map(d => [d.macd, d.signal, d.hist]).flat();
    const maxMacd = Math.max(...macdIndicators, 0.02);
    const minMacd = Math.min(...macdIndicators, -0.02);
    const macdDelta = maxMacd - minMacd || 1;
    const width = 700;
    const height = 450;
    const panelGap = 10;
    const candleAreaHeight = height * 0.47;
    const rsiAreaHeight = height * 0.20;
    const macdAreaHeight = height * 0.23;
    const rsiTop = candleAreaHeight + panelGap;
    const macdTop = rsiTop + rsiAreaHeight + panelGap;
    return { width, height, maxPrice, minPrice, priceDelta, maxMacd, minMacd, macdDelta,
        candleAreaHeight, rsiAreaHeight, macdAreaHeight, rsiTop, macdTop,
        horizontalStep: width / (data.length || 1) };
};

const computeSupportResistance = (data, priceDelta) => {
    const levelTolerance = Math.max(priceDelta * 0.015, 0.01);
    const swingPoints = [];
    for (let i = 2; i < data.length - 2; i++) {
        const d = data[i];
        const isSwingHigh =
            d.high > data[i - 1].high && d.high > data[i - 2].high &&
            d.high > data[i + 1].high && d.high > data[i + 2].high;
        const isSwingLow =
            d.low < data[i - 1].low && d.low < data[i - 2].low &&
            d.low < data[i + 1].low && d.low < data[i + 2].low;
        if (isSwingHigh) swingPoints.push({ price: d.high, type: 'resistance' });
        if (isSwingLow) swingPoints.push({ price: d.low, type: 'support' });
    }
    const clusterLevels = (points, type) => {
        const clusters = [];
        points.filter(p => p.type === type).forEach(point => {
            const existing = clusters.find(c => Math.abs(c.price - point.price) <= levelTolerance);
            if (existing) {
                existing.prices.push(point.price);
                existing.price = existing.prices.reduce((a, b) => a + b, 0) / existing.prices.length;
                existing.touches++;
            } else {
                clusters.push({ price: point.price, prices: [point.price], touches: 1, type });
            }
        });
        return clusters
            .filter(c => c.touches >= 2)
            .sort((a, b) => b.touches - a.touches || a.price - b.price)
            .slice(0, 3);
    };
    return { supportLevels: clusterLevels(swingPoints, 'support'), resistanceLevels: clusterLevels(swingPoints, 'resistance') };
};
```

Note: strict `>`/`<` comparisons (fixes plateau series marking every bar). Delete the in-component copies of `buildChartModel`, swing scan, and `clusterLevels`.

- [ ] **Step 3: Consistent indices — use `chartData` everywhere inside the chart.** Replace remaining `vectorData` usages:
- S/R scan: done (moved to `computeSupportResistance(chartData, ...)`).
- Hit rects (~2819): `chartData.map((d, i) => ...)` instead of `vectorData.map`.
- Handlers: `onMouseEnter={() => { scheduleHover(d, i); if (isSelectingRange) setRangeSelection(prev => prev ? { ...prev, end: i } : { start: i, end: i }); }}`, `onMouseLeave={() => scheduleHover(null, -1)}`, `onMouseDown` unchanged shape but `scheduleHover(d, i)` first, `onMouseUp` removed (window listener handles it; keep per-rect `onMouseUp` too is harmless — remove it to avoid duplication).
- Crosshair (~2733): use `hoverIndex` directly: `if (!isSelectingRange && hoverIndex >= 0) { const centerX = hoverIndex * horizontalStep + horizontalStep / 2; ... }` (drops the O(n) `indexOf`).
- Hover readout uses `hoverNode` (unchanged name via destructure).

- [ ] **Step 4: Chart mount key** (~2376) — stop remounting on every data refresh:

```jsx
<CandlestickChart
    key={`${activeProvider}-${activeStock}-${timeframe}`}
    vectorData={renderedChartData}
    theme={theme}
    showPatterns={showPatterns}
/>
```

- [ ] **Step 5: Run standard verification** — chart renders; hover still works (drive with Playwright mouse move over the chart area, screenshot); range selection still works (mousedown+move+mouseup over candles → RANGE readout appears; also mousedown then mouseup **outside** any candle → selection ends).

---

### Task 10: Dashboard render-loop reductions (telemetry, ticker lookup)

**Goal:** No guaranteed full re-render every 20 s; avoid repeated linear dictionary scans.

**Files:**
- Modify: `index.html` (telemetry effect ~1241–1263, watchlist row ~2310, other `TICKER_DICTIONARY.find` hot call sites ~1401/1620/1906/2120)

- [ ] **Step 1:** Telemetry — bail out when unchanged:

```js
setMarketTelemetry(prev => {
    const next = /* computed { isOpen, text } */;
    return prev.isOpen === next.isOpen && prev.text === next.text ? prev : next;
});
```

(Compute `next` first inside `trackHours`, then apply the bailing setter.)

- [ ] **Step 2:** Module-scope ticker index (avoids O(n) `find` in render loops):

```js
const TICKER_BY_SYMBOL = new Map(TICKER_DICTIONARY.map(item => [item.ticker, item]));
```

Replace `TICKER_DICTIONARY.find(i => i.ticker === ticker)` call sites with `TICKER_BY_SYMBOL.get(ticker)` (watchlist row ~2310, `createMarketData` ~1401, demo meta ~1620, cache meta ~1906, `activeCompanyMeta` ~2120, `generateFinancials` ~396, `priceAlerts` initializer unaffected — it uses `filter`, fine).

- [ ] **Step 3: Run standard verification** — unchanged behavior; 0 errors.

---

### Task 11: Pattern Lab demo-mode disclosure

**Goal:** Demo event-study statistics are labeled as synthetic (generator plants patterns + follow-through).

**Files:**
- Modify: `index.html` (`PatternLab` ~2845, mount ~2379)

- [ ] **Step 1:** Add prop: `function PatternLab({ data, theme, isDemo })` and mount:

```jsx
{showPatternLab && <PatternLab data={renderedChartData} theme={theme} isDemo={!hasLiveCredentials} />}
```

- [ ] **Step 2:** Inside `PatternLab`, right after the header row (~2876), add:

```jsx
{isDemo && (
    <div className="mb-3 px-2 py-1 border border-amber-500/40 bg-amber-500/10 text-amber-500 text-[9px] font-mono">
        DEMO MODE — synthetic candles: patterns and their follow-through are scripted by the data generator, so win rates are illustrative, not market evidence.
    </div>
)}
```

- [ ] **Step 3: Run standard verification** — open Pattern Lab (evaluate: click the PATTERN LAB button via Playwright), banner visible in demo mode; screenshot.

---

### Task 12: Structural extraction — provider loaders + `useMarketDataPipeline` hook

**Goal:** Provider adapters and the data pipeline move out of `Dashboard`; `Dashboard` shrinks and stops threading 10 dependencies through loader params.

**Files:**
- Modify: `index.html` (loaders ~1413–1592, `fetchPipelineData` ~1594–1946, `Dashboard` state/effects)

**Interfaces (Produces):**
- Module scope: `fetchAlphaVantageJson(apiKey, params)` (now takes key explicitly), `loadAlpacaMarketData({ watchlist, selectedTimeframeConfig, timeframe, activeStock, alpacaKeyId, alpacaSecretKey, setRenderedChartData })`, `loadAlphaVantageMarketData({ watchlist, selectedTimeframeConfig, timeframe, activeStock, alphaVantageKey, setRenderedChartData })`, `loadTwelveDataMarketData({ watchlist, selectedTimeframeConfig, timeframe, activeStock, apiKey, setRenderedChartData })`. Loaders now call module-scope `getTimeframeBars`, `normalizeMarketBars`, `createMarketData`, `calculateTechnicalIndicators`, `writeTimeframeCache` directly (no longer passed in).
- `useMarketDataPipeline(options) → { syncProgress, syncStatusText, renderedChartData, isRefreshing, stockPrices, quoteInfo, fundamentals, liveFinancials, handleManualRefresh, refreshNow }`

- [ ] **Step 1:** Move the three loaders + `fetchAlphaVantageJson` to module scope (place after `createMarketData`). Signature changes:
- `fetchAlphaVantageJson(alphaVantageKey, params)` — every internal call site updated (loaders, quote ~1726, fundamentals ~1819–1821, search ~1954).
- Loaders: drop params `selectedInterval` (Twelve Data computes `selectedInterval = selectedTimeframeConfig.tdInterval` internally), `writeTimeframeCache`, `getTimeframeBars`, `calculateTechnicalIndicators`, `createMarketData` (all module scope now). Keep credentials/setter params.
- Move the `setSyncStatusText` calls for rate-limit aborts out of loaders: loaders receive an optional `onStatus(text)` callback param instead of `setSyncStatusText` directly? Simpler: pass `setSyncStatusText` as part of options (it's just a setter) — keep `setSyncStatusText` in the options object. Decision: **pass `setSyncStatusText`**.

- [ ] **Step 2:** Create `useMarketDataPipeline` at module scope (after loaders, before `Dashboard`):

```js
const useMarketDataPipeline = ({ watchlist, activeStock, timeframe, activeProvider, hasLiveCredentials, apiKey, alphaVantageKey, alpacaKeyId, alpacaSecretKey }) => {
    const [syncProgress, setSyncProgress] = React.useState(100);
    const [syncStatusText, setSyncStatusText] = React.useState('SYSTEM_READY');
    const [renderedChartData, setRenderedChartData] = React.useState([]);
    const [isRefreshing, setIsRefreshing] = React.useState(false);
    const [stockPrices, setStockPrices] = React.useState({});
    const [fundamentals, setFundamentals] = React.useState({});
    const [quoteInfo, setQuoteInfo] = React.useState({});
    const [liveFinancials, setLiveFinancials] = React.useState({});

    const fetchPipelineData = React.useCallback(async (forceRefresh = false) => {
        ... existing body unchanged (uses the state/setters above via closure),
        plus module refs: timeframeConfig, getCacheKey..., loadXMarketData(...), getQuoteCacheKey...
    }, [watchlist, timeframe, apiKey, alphaVantageKey, alpacaKeyId, alpacaSecretKey,
        activeProvider, hasLiveCredentials, activeStock, fundamentals, quoteInfo]);

    const fetchPipelineRef = React.useRef(fetchPipelineData);
    fetchPipelineRef.current = fetchPipelineData;

    React.useEffect(() => {
        fetchPipelineRef.current?.().catch(error => {
            console.error('[PIPELINE] refresh failed:', error);
            setSyncStatusText('API ERROR: PIPELINE FAILURE');
        });
    }, [activeStock, timeframe, apiKey, alphaVantageKey, alpacaKeyId, alpacaSecretKey, activeProvider, watchlist]);

    const handleManualRefresh = React.useCallback(async () => {
        if (isRefreshing) return;
        setIsRefreshing(true);
        try {
            await fetchPipelineData(true);
        } catch (error) {
            console.error('[PIPELINE] manual refresh failed:', error);
            setSyncStatusText(`API ERROR: ${String(error?.message || 'UNKNOWN').slice(0, 20)}`);
        } finally {
            setIsRefreshing(false);
        }
    }, [fetchPipelineData, isRefreshing]);

    return { syncProgress, syncStatusText, renderedChartData, isRefreshing, stockPrices, quoteInfo, fundamentals, liveFinancials, handleManualRefresh };
};
```

- [ ] **Step 3:** Rewire `Dashboard`:
- Delete moved state: `syncProgress/syncStatusText/renderedChartData/isRefreshing/stockPrices/fundamentals/quoteInfo/liveFinancials` declarations, `fetchPipelineData`, the pipeline effect, `handleManualRefresh`.
- Destructure from the hook (keep `setStockPrices` exposure if any Dashboard-side writer exists — currently none after Task 7; the alerts effect only *reads* `stockPrices`).
- `__APP_DEBUG__` effect reads the destructured values (unchanged).
- Update `handleManualRefresh` button reference (unchanged name).

- [ ] **Step 4: Run standard verification** — demo pipeline still populates: `__APP_DEBUG__.chartData.length > 0`, `stockPrices` has 4 tickers, sync status shows `SIMULATION MODE BUFFERED`; refresh button works (click via Playwright); 0 errors.

---

### Task 13: Split Dashboard JSX into section components

**Goal:** `Dashboard`'s 317-line return becomes an assembly of named sections.

**Files:**
- Modify: `index.html` (Dashboard return ~2164–2481 → after Task 12 line numbers differ; locate by JSX markers)

**Design:** New components (defined at module scope above `Dashboard`, all receive explicit props; `theme` passed everywhere; children/JSX copied verbatim, only the wiring changes):

1. `HeaderBar({ theme, isRefreshing, onRefresh, language, onLanguageChange, onToggleTheme, onToggleApi, showApiInput, activeProvider, hasLiveCredentials, onClearApi })` — progress bar + header (~2167–2230 status strip included in `StatusBar` instead).
2. `StatusBar({ theme, syncStatusText, marketTelemetry, hasLiveCredentials })` (~2224–2230).
3. `ApiKeyPanel({ theme, values: { apiKey, alphaVantageKey, alpacaKeyId, alpacaSecretKey }, onSave, onClose })` — inputs + CONNECT button; the CONNECT inline handler becomes a function `saveApiKeys()` in Dashboard reading the four input ids, calling the four setters + `Storage.set` + `setShowApiInput(false)`.
4. `WatchlistSidebar({ theme, watchlist, setWatchlist, stockPrices, activeStock, setActiveStock, resolvedSymbols, searchQuery, onSearchChange, onSearchKeyDown, suggestions, onPickSuggestion })` (~2278–2334).
5. `ChartSection({ theme, ...chart header state })` — the `<main>` (~2336–2380): needs `timeframe/setTimeframe, showPatterns, setShowPatterns, showPatternLab, setShowPatternLab, isMaximized, setIsMaximized, activeStock, activeCompanyMeta, stockPrices, activeProvider, renderedChartData, hasLiveCredentials`.
6. `RightRail({ theme, ... })` — `<aside>` (~2382–2476): quantitative matrix, alerts, financials; needs `stockPrices, quoteInfo, activeStock, activeCompanyMeta, timeframe, windowHigh/Low, priceAlerts, setPriceAlerts, notificationStatus, requestNotificationPermission, showFinancials, financialView, setFinancialView, activeFinancialsData`.

Rules:
- Copy JSX **verbatim** (only `class`→ already stays as-is until Task 15; do not mix concerns).
- Components are plain function declarations; no state moved except where trivially local.
- Keep all cataloged strings untouched (Task 14 translates them).

- [ ] **Step 1:** Extract components one at a time (StatusBar → ApiKeyPanel → WatchlistSidebar → RightRail → ChartSection → HeaderBar), running standard verification after each extraction.
- [ ] **Step 2: Final standard verification** — pixel-compare screenshots before/after (take screenshot at Task 12 completion as baseline, compare at end): layout identical.

---

### Task 14: React-level i18n — context + `t()`, remove TreeWalker

**Goal:** Replace the racy DOM TreeWalker translation with React-context translation at render time.

**Files:**
- Modify: `index.html` (`uiTranslations` ~1106, `translateStaticText` ~1150, Dashboard language effect ~1175–1180, all cataloged string sites)

- [ ] **Step 1: Add context + hook** (after `uiTranslations`, replacing `translateStaticText`):

```js
const LanguageContext = React.createContext('en');
const getTranslationLanguage = language => language === 'pt-BR' ? 'pt' : language;
const translateText = (key, language) => {
    const entry = uiTranslations[key];
    const lang = getTranslationLanguage(language);
    return entry && entry[lang] !== undefined ? entry[lang] : key;
};
const useT = () => {
    const language = React.useContext(LanguageContext);
    return key => translateText(key, language);
};
```

Delete `translateStaticText` entirely. Language effect becomes:

```js
React.useEffect(() => {
    Storage.set(STORAGE_KEYS.LANGUAGE, language);
    document.documentElement.lang = language;
}, [language]);
```

- [ ] **Step 2:** Wrap Dashboard's returned tree: outermost element becomes

```jsx
<LanguageContext.Provider value={language}>
    <div className={...}> ... </div>
</LanguageContext.Provider>
```

- [ ] **Step 3: Convert every cataloged string site.** For each key in `uiTranslations`, `grep -n '<key literal>' index.html` and wrap JSX text occurrences in `t('...')` (add `const t = useT();` at the top of any component that uses it: Dashboard, HeaderBar, StatusBar, ApiKeyPanel, WatchlistSidebar, RightRail, ChartSection, PatternLab, CandlestickChart).

Key list (from `uiTranslations`): `'TRADING TERMINAL // NASDAQ'`, `'Stock Dashboard //'`, `'DataStream'`, `'CLEAR API'`, `'DATA PROVIDER'`, `'Market Data Provider'`, `'ALPACA KEY ID'`, `'ALPACA SECRET KEY'`, `'ALPHAVANTAGE API KEY'`, `'TWELVEDATA API KEY'`, `'FINANCIAL STATEMENTS'`, `'PRICE ALERTS'`, `'PATTERN LAB'`, `'PATTERN LAB // HISTORICAL EVENT STUDY'`, `'PRICE ACTION // OHLC'`, `'RSI (14) // 30 / 70'`, `'MACD (12,26,9)'`, `'QUANTITATIVE DATA MATRIX'`, `'Watchlist Matrix Grid'`, `'TIMEFRAME'`, `'SIGNALS'`, `'STREAM LOGS //'`, `'TIMELINE TRACKER ACTIVE'`, `'CONNECT'`, `'REMOVE'`, `'NOTIFY'`, `'ABOVE'`, `'BELOW'`, `'BULLISH REVERSAL'`, `'BEARISH REVERSAL'`, `'Current / incomplete signals'`, `'Individual occurrences'`, `'Pattern results'`, `'Performance Metrics (REV vs NET)'`, `'No candlestick patterns detected in the current dataset.'`, `'No completed pattern occurrences in the current dataset.'`, `'No historical data segments loaded.'`.
- `'TRADING TERMINAL // PRO'` is the document `<title>` — set it in the language effect instead: `document.title = translateText('TRADING TERMINAL // PRO', language);`
- Keys `'BULLISH '`/`'BEARISH '` (trailing space — previously dead): normalize object keys to `'BULLISH'`/`'BEARISH'` and apply to PatternLab's `{r.type}` occurrences (open-signals list + type column context) — check each `{r.type}` render and wrap: `t(r.type)`.
- Dynamic texts (`syncStatusText`, provider button labels like `ALPACA API`) are not in the catalog — leave.
- `<option>` value labels (PT-BR/EN/ES/FR) — leave.

- [ ] **Step 4:** After conversion, verify no cataloged English literal remains unwrapped in JSX: for each key, `grep -n ">\s*<key>" index.html` should show no raw JSX text matches (only the catalog definition + `t('...')` calls).

- [ ] **Step 5: Run standard verification**; then set language to `pt-BR` via the `<select>` (Playwright select option) and verify header shows `TERMINAL DE TRADING // NASDAQ` (evaluate `document.body.innerText.includes('TERMINAL DE TRADING')`), switch back to EN.

---

### Task 15: Cleanup — dead code, merge artifacts, `className`, color constants, repo hygiene

**Files:**
- Modify: `index.html`, create `.gitignore`

- [ ] **Step 1: Dead code removal** (verify zero references by grep before each delete):
- `Icons.TrendingUp` / `Icons.TrendingDown` (~189–190).
- `getPatternStatistics` + `getPatternStatisticsSafe` aliases (~1102–1103): change PatternLab's `getPatternStatisticsSafe(data)` call to `analyzePatternLab(data)`, delete both aliases.
- `TICKER_DICTIONARY` fields `high2y`/`low2y` (grep first — remove only if 0 usages).
- Any remaining unused `cacheKey` locals.

- [ ] **Step 2: Fix merged-line/indentation artifacts** — split these known double-statements onto separate lines (locate by content, lines from plan-time HEAD): the `selectedInterval` line (~1598), `processedData...cacheKey` (~1588, removed by Task 6 anyway), `put()` doubles in `generateForcedFakeData` (~630), `} }, 300);` (~2087), `if (exactSuggestion) { addSearchSymbol...` (~2144), `<g> <line ...` (~2663), the `'PRICE ACTION // OHLC'` catalog line (~1120), `const analyzePatternLab` (~1065), `const selectedTimeframeConfig...` — and fix the stray indentation of `function Dashboard() {` / `analyzePatternLab` / `uiTranslations` to match the file's 8-space module-scope convention. Reformat any line > 200 chars touched by these edits.

- [ ] **Step 3: `class` → `className` sweep (JSX only):**
```bash
# first confirm no string literals contain 'class=' inside the script block:
grep -n "class=" index.html | grep -v "className" | head -50   # review every hit
sed -i '56,/<\/script>/s/\bclass=/className=/g' index.html      # script block only; adjust range to actual script end line
grep -c "className=" index.html
grep -n "[^Name]class=" index.html   # expect only the <body class=...> at line 51 (outside script)
```

- [ ] **Step 4: Color constants for SVG/inline props** — add module scope:

```js
const COLORS = Object.freeze({
    up: '#00FF66',
    down: '#ef4444',
    rsiLine: '#a855f7',
    macdLine: '#3b82f6',
    signalLine: '#eab308',
    crosshair: 'rgba(0,255,102,0.15)'
});
```

Replace matching **attribute** literals in `CandlestickChart` (`fill="#00FF66"` etc. → `fill={COLORS.up}`; `stroke="#a855f7"` → `stroke={COLORS.rsiLine}`; dashed zero-line / hover crosshair strokes as applicable). **Do not** touch Tailwind class strings like `text-[#00FF66]` (Tailwind needs literals).

- [ ] **Step 5: Repo hygiene** — create `.gitignore`:

```
.playwright-mcp/
```

- [ ] **Step 6: Run standard verification** — full pass; visually compare screenshots (theme toggle light/dark) against baseline.

---

### Task 16: Version bump, docs, final verification

**Files:**
- Modify: `index.html` meta version, `README.md`, `agent.md`, `skills.md`

- [ ] **Step 1:** `index.html` `<meta name="version" content="1.10.0">`.
- [ ] **Step 2: Docs** — update the three docs where they describe architecture/behavior:
- README: version → 1.10.0; add a "v1.10.0 fixes" section summarizing: indicator warm-up correctness (null warm-up, Wilder ATR), demo indicators, session bucketing in Eastern time, quarter labels, rate-limit backoff + partial refetch, Alpaca newest-first pagination, SRI-pinned CDN, memoized chart, React-level i18n, `useMarketDataPipeline` extraction.
- README technical-analysis list: ATR described as Wilder ATR.
- agent.md / skills.md: update architecture notes to match (module-scope providers, pipeline hook, `t()` i18n via context, `window.__APP_DEBUG__` test hook, in-page test suites: core/indicator/candle-pattern).
- Do not invent new features not implemented.
- [ ] **Step 3: Final full verification:**
  1. Standard verification (fresh load): `[REGRESSION CHECKS] 10/10`, `[INDICATOR CHECKS] 7/7`, `[CANDLE PATTERN TESTS] 11/11`, zero console errors.
  2. Interactions via Playwright: timeframe buttons (1W/1M/1Y), theme toggle, pattern overlay toggle, Pattern Lab open, watchlist switch ticker, search input typing, manual refresh, range drag select. Zero errors after each.
  3. Screenshots: dark demo default, light theme, Pattern Lab — saved for the summary.
  4. `git status` / `git diff --stat` review — only intended files changed.
- [ ] **Step 4:** Report completion summary with diff stats. **Do not commit.**

## Self-Review Notes

- All review findings mapped: EMA warmup (T2), demo indicators (T4), Q1 label (T1), session timezone (T1), rate limits (T6), Alpaca sort (T6), rejections (T3/T5), SRI (T8), tweezers/doji (T3), methodology SMA/BB/ATR/RSI (T2), provider inconsistencies AV interval (T6), fallbacks (T1), stale closures (T7), Pattern Lab circularity → disclosure (T11), perf (T9/T10), structural (T12/T13), i18n (T14), dead code/artifacts/class/colors (T15), docs (T16).
- Deliberately **not** done (documented decisions): no CSP header (too risky for Babel-standalone inline scripts), no full palette rewrite of Tailwind class strings (Tailwind requires literals), detector's internal ATR stays a simple TR average (fixtures pin its behavior; module ATR is Wilder), keys stay in query string where providers require it (documented in UI).
- Type consistency: loaders consistently take `{ watchlist, selectedTimeframeConfig, timeframe, activeStock, ...credentials, setRenderedChartData, setSyncStatusText }` after T12; `fetchAlphaVantageJson(apiKey, params)` everywhere.
