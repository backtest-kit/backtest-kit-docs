---
title: article/01_look_ahead_bias
group: article
---

# 🧿 How I Made Look-Ahead Bias Architecturally Impossible in Trading Backtests

![Prophet](../../assets/prophet.png)

## The Problem That Breaks 90% of Trading Bots

You write a trading strategy. Backtest shows amazing returns. You deploy it live. It loses money.

**Why?** Look-ahead bias. Your backtest accidentally used future data.

I built [backtest-kit](https://github.com/tripolskypetr/backtest-kit) to make this **architecturally impossible** using Node.js `AsyncLocalStorage`.

---

## What is Look-Ahead Bias?

Look-ahead bias happens when your backtest uses information that wouldn't have been available at the time of the trade.

**Classic example:**

```javascript
// ❌ WRONG: This peeks into the future!
function shouldBuy(candles, currentIndex) {
  const currentPrice = candles[currentIndex].close;
  const nextPrice = candles[currentIndex + 1].close; // Future data!
  
  return nextPrice > currentPrice; // "Buy if price will go up"
}
```

Obvious, right? But most look-ahead bias is subtle:

- Loading indicators without filtering by timestamp
- Accidentally including one extra candle in calculations
- Using data that "leaked" from the future

---

## The Traditional Approach (That Doesn't Work)

Most frameworks tell you: "Just be careful!"

```javascript
// Traditional: You must remember to filter
getSignal: async (candles, currentTime) => {
  const validCandles = candles.filter(c => c.timestamp <= currentTime);
  const rsi = calculateRSI(validCandles);
  // What if you forget to filter? Backtest won't catch it.
}
```

**The problem:** This relies on discipline. One mistake ruins everything.

---

## The Solution: AsyncLocalStorage as Temporal Context

I used Node.js `AsyncLocalStorage` to create an immutable temporal context that flows through all async operations.

**How it works:**

```javascript
import { AsyncLocalStorage } from 'async_hooks';

// Framework creates a temporal context storage
const backtestContext = new AsyncLocalStorage();

// For each backtest tick, we establish the current time
async function processTick(timestamp, symbol) {
  const context = {
    currentTime: timestamp,  // This is "now" for this tick
  };
  
  // Everything inside run() operates in this temporal context
  await backtestContext.run(context, async () => {
    const signal = await strategy.getSignal(symbol);
    await processSignal(signal);
  });
}
```

When you request data, it **automatically** respects the temporal context:

```javascript
async function getCandles(symbol, interval, limit) {
  const context = backtestContext.getStore();
  
  // ALWAYS fetches data UP TO context.currentTime
  // Future data is architecturally impossible to access
  return await exchange.getCandles(
    symbol,
    interval,
    context.currentTime,  // From backtest context
    limit
  );
}
```

**Result:** Your strategy can't accidentally use future data. The architecture prevents it.

---

## Multi-Timeframe Analysis: Automatic Synchronization

Traditional frameworks make multi-timeframe analysis error-prone:

```javascript
// ❌ Traditional: Easy to mess up
getSignal: async (currentTime) => {
  const candles1h = await getCandles('1h', currentTime, 100);
  const candles15m = await getCandles('15m', currentTime, 100);
  // Are these synchronized? Did I pass currentTime everywhere?
}
```

With AsyncLocalStorage, **all timeframes are automatically synchronized**:

```javascript
// ✅ backtest-kit: Impossible to mess up
getSignal: async (symbol) => {
  // All these use the SAME timestamp from context
  const candles1h = await getCandles(symbol, '1h', 100);
  const candles15m = await getCandles(symbol, '15m', 100);
  const candles5m = await getCandles(symbol, '5m', 100);
  
  // Data is perfectly synchronized to current backtest tick
  // No way to accidentally include future data
}
```

**No timestamp parameters. No manual filtering. Just works.**

---

## Same Code for Backtest AND Live Trading

Here's the magic: **identical strategy code** works for both modes.

## Backtest Mode (from demo/backtest/src/index.mjs)

```javascript
import { Backtest, listenSignalBacktest, listenBacktestProgress } from "backtest-kit";

// Run backtest on historical data
Backtest.background("BTCUSDT", {
    strategyName: "test_strategy",
    exchangeName: "test_exchange",
    frameName: "test_frame",
});

// Monitor progress
listenBacktestProgress((event) => {
    console.log(`Progress: ${(event.progress * 100).toFixed(2)}%`);
    console.log(`Processed: ${event.processedFrames} / ${event.totalFrames}`);
});

// Handle signals
listenSignalBacktest((event) => {
    console.log(event);
});
```

## Live Mode (from demo/live/src/index.mjs)

```javascript
import { Live, listenSignalLive } from "backtest-kit";

// Same strategy, now running live!
Live.background("BTCUSDT", {
    strategyName: "test_strategy",
    exchangeName: "test_exchange",
    frameName: "test_frame",
});

// Handle live signals
listenSignalLive(async (event) => {
    if (event.action === "opened") {
        console.log("Open position");
    }
    if (event.action === "closed") {
        console.log("Close position");
        await Live.dump(event.symbol, event.strategyName);
    }
});
```

**The only difference?**

- Backtest: `context.currentTime` comes from historical data
- Live: `context.currentTime = Date.now()`

**Your strategy doesn't change at all.**

---

## Real Production Setup (Actual Demo Code)

Here's the complete setup from my demo project:

```javascript
import ccxt from "ccxt";
import { addExchange, addStrategy, addFrame, addRisk } from "backtest-kit";
import { v4 as uuid } from "uuid";
import { json } from "./utils/json.mjs";
import { getMessages } from "./utils/messages.mjs";

// 1. Configure exchange (CCXT integration)
addExchange({
    exchangeName: "test_exchange",
    getCandles: async (symbol, interval, since, limit) => {
        const exchange = new ccxt.binance();
        const ohlcv = await exchange.fetchOHLCV(
            symbol, 
            interval, 
            since.getTime(), 
            limit
        );
        return ohlcv.map(([timestamp, open, high, low, close, volume]) => ({
            timestamp, open, high, low, close, volume
        }));
    },
    formatPrice: async (symbol, price) => price.toFixed(2),
    formatQuantity: async (symbol, quantity) => quantity.toFixed(8),
});

// 2. Risk management rules
addRisk({
    riskName: "demo_risk",
    validations: [
        {
            validate: ({ pendingSignal, currentPrice }) => {
                const { priceOpen = currentPrice, priceTakeProfit, position } = pendingSignal;
                if (!priceOpen) return;
                
                // Calculate TP distance percentage
                const tpDistance = position === "long"
                    ? ((priceTakeProfit - priceOpen) / priceOpen) * 100
                    : ((priceOpen - priceTakeProfit) / priceOpen) * 100;
                if (tpDistance < 1) {
                    throw new Error(`TP distance ${tpDistance.toFixed(2)}% < 1%`);
                }
            },
            note: "TP distance must be at least 1%",
        },
        {
            validate: ({ pendingSignal, currentPrice }) => {
                const { priceOpen = currentPrice, priceTakeProfit, priceStopLoss, position } = pendingSignal;
                if (!priceOpen) return;
                
                // Calculate reward (TP distance)
                const reward = position === "long"
                    ? priceTakeProfit - priceOpen
                    : priceOpen - priceTakeProfit;
                    
                // Calculate risk (SL distance)
                const risk = position === "long"
                    ? priceOpen - priceStopLoss
                    : priceStopLoss - priceOpen;
                    
                if (risk <= 0) {
                    throw new Error("Invalid SL: risk must be positive");
                }
                
                const rrRatio = reward / risk;
                if (rrRatio < 2) {
                    throw new Error(`RR ratio ${rrRatio.toFixed(2)} < 2:1`);
                }
            },
            note: "Risk-Reward ratio must be at least 1:2",
        },
    ],
});

// 3. Define timeframe
addFrame({
    frameName: "test_frame",
    interval: "1m",
    startDate: new Date("2025-12-01T00:00:00.000Z"),
    endDate: new Date("2025-12-01T23:59:59.000Z"),
});

// 4. Strategy logic
addStrategy({
    strategyName: "test_strategy",
    interval: "5m",
    riskName: "demo_risk",
    getSignal: async (symbol) => {
        // getMessages internally calls getCandles
        // which automatically respects temporal context
        const messages = await getMessages(symbol);
        
        const resultId = uuid();
        // Creates a trading signal using Ollama
        const result = await json(messages);
        await dumpSignal(resultId, messages, result);
        
        result.id = resultId;
        return result;
    },
});
```

Notice how `getSignal` has **no timestamp parameters**. The temporal context flows automatically through `getMessages` → `getCandles`.

---

## Production Features from Demo

## 1. Progress Monitoring

```javascript
import { listenBacktestProgress, listenDoneBacktest } from "backtest-kit";

listenBacktestProgress((event) => {
    console.log(`Progress: ${(event.progress * 100).toFixed(2)}%`);
    console.log(`Processed: ${event.processedFrames} / ${event.totalFrames}`);
});

listenDoneBacktest(async (event) => {
    console.log("Backtest completed:", event.symbol);
    await Backtest.dump(event.symbol, event.strategyName);
});
```

## 2. Partial Profit/Loss Management

```javascript
import { listenPartialProfit, listenPartialLoss, Constant } from "backtest-kit";

listenPartialProfit(({ symbol, price, level }) => {
  console.log(`${symbol} reached ${level}% profit at ${price}`);
  
  if (level === Constant.TP_LEVEL3) {
    console.log("Close 33% at 90% profit");
  }
  if (level === Constant.TP_LEVEL2) {
    console.log("Close 33% at 60% profit");
  }
  if (level === Constant.TP_LEVEL1) {
    console.log("Close 34% at 30% profit");
  }
});

listenPartialLoss(({ symbol, price, level }) => {
  console.log(`${symbol} reached -${level}% loss at ${price}`);
  
  if (level === Constant.SL_LEVEL2) {
    console.log("Close 50% at -80% loss");
  }
  if (level === Constant.SL_LEVEL1) {
    console.log("Close 50% at -40% loss");
  }
});
```

## 3. Risk Validation Events

```javascript
import { listenRisk, listenError } from "backtest-kit";

listenRisk(async (event) => {
    // Risk validation failed
    await Risk.dump(event.symbol, event.strategyName);
});

listenError((error) => {
    console.error("Error occurred:", error);
});
```

## 4. Live Trading State Persistence

```javascript
import { listenSignalLive } from "backtest-kit";

listenSignalLive(async (event) => {
    if (event.action === "opened") {
        console.log("Open position");
    }
    
    if (event.action === "closed") {
        console.log("Close position");
        // Atomic dump to disk for crash recovery
        await Live.dump(event.symbol, event.strategyName);
        await Partial.dump(event.symbol, event.strategyName);
    }
    
    if (event.action === "scheduled") {
        // Limit order scheduled
        await Schedule.dump(event.symbol, event.strategyName);
    }
    
    if (event.action === "cancelled") {
        // Scheduled order cancelled
        await Schedule.dump(event.symbol, event.strategyName);
    }
});
```

---

## Why This Matters

**Look-ahead bias is silent.** Your backtest won't warn you. It just shows inflated returns.

Then you deploy live and lose money.

**Traditional solutions rely on discipline.** One mistake, one forgotten filter, and your months of work are worthless.

**backtest-kit makes it architectural.** You literally cannot access future data, even if you try.

---

## Why AsyncLocalStorage is Perfect for This

**Traditional approach:** Pass context explicitly

```javascript
// Verbose and error-prone
async function getSignal(symbol, context) {
  const candles = await getCandles(symbol, context.currentTime);
  const rsi = await calculateRSI(candles, context.currentTime);
  // What if you forget to pass context somewhere?
}
```

**AsyncLocalStorage approach:** Context flows automatically

```javascript
// Clean and foolproof
async function getSignal(symbol) {
  const candles = await getCandles(symbol);
  const rsi = await calculateRSI(candles);
  // Context is always available, always correct
}
```

**Key benefits:**

1. ✅ **Zero boilerplate** — No context parameters to pass around
2. ✅ **Type-safe** — TypeScript knows what's available
3. ✅ **Impossible to forget** — Context is always there
4. ✅ **Works across async boundaries** — Even through `Promise.all()`

---

## How AsyncLocalStorage Handles Edge Cases

## Works Through Promise.all

```javascript
getSignal: async (symbol) => {
  // Context preserved across concurrent operations
  const [c1h, c15m, c5m] = await Promise.all([
    getCandles(symbol, '1h', 100),
    getCandles(symbol, '15m', 100),
    getCandles(symbol, '5m', 100)
  ]);
  
  // All synchronized to the same timestamp
}
```

## Error Handling

```javascript
listenError((error) => {
    console.error("Error occurred:", error);
});

listenRisk(async (event) => {
    // Validation failed - log for analysis
    await Risk.dump(event.symbol, event.strategyName);
});
```

---

## Key Takeaways

1. **Architecture > Discipline** — Don't rely on being careful. Make mistakes impossible.
2. **Same Code, Same Results** — If backtest and live use different code paths, you're doing it wrong.
3. **AsyncLocalStorage is Underrated** — Perfect for this pattern, criminally underused.
4. **Temporal Context Flows** — No timestamp parameters needed. Context propagates automatically.
5. **Production-Ready Matters** — Crash recovery, atomic persistence, and validation aren't optional.

---

## Conclusion

Look-ahead bias breaks most trading bots. Traditional solutions rely on developers being careful.

**backtest-kit makes carelessness impossible** by using AsyncLocalStorage to create an immutable temporal context.

The result? Strategies that work the same in backtest and live trading.

---

**Try it yourself:**

```bash
npm install backtest-kit ccxt
```

**Resources:**

- 📚 [Documentation](https://backtest-kit.github.io/documents/design_01_overview.html)
- 💻 [GitHub Repository](https://github.com/tripolskypetr/backtest-kit)
- 🎯 [Demo Projects](https://github.com/tripolskypetr/backtest-kit/tree/master/demo)
- 📖 [Understanding Signals](https://backtest-kit.github.io/documents/example_03_understanding_signals.html)

---

