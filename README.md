# Backtest Kit Documentation

## Introduction

Backtest Kit is a production-ready TypeScript framework for algorithmic trading that solves the fundamental challenges of building reliable trading systems. Unlike research-focused libraries, it provides battle-tested solutions for the real-world problems that emerge when moving from backtesting to live trading.

This documentation explains the core concepts, design decisions, and problem-solving approaches that make Backtest Kit suitable for production trading systems.

---

## Core Concepts

### Signal Lifecycle State Machine

Every trading position follows a strict lifecycle: **idle → opened → active → closed**. This state machine is enforced through TypeScript discriminated unions, making invalid states unrepresentable at compile time.

**Problem Solved:** Traditional trading systems often suffer from state corruption where positions exist in ambiguous states (partially opened, unclear if closed, etc.). Backtest Kit eliminates this class of bugs entirely.

**Key Benefits:**
- Impossible to access closed position data
- Impossible to modify opened position as if it were active
- Type-safe access to position properties based on current state
- Automatic validation of state transitions

### Execution Context Propagation

The framework uses Node.js AsyncLocalStorage to maintain temporal context throughout the entire execution stack. Every function call automatically knows:
- Current simulation timestamp (backtest) or real time (live)
- Trading symbol being processed
- Strategy and exchange configuration
- Whether running in backtest or live mode

**Problem Solved:** Manual timestamp passing creates error-prone code where developers accidentally use wrong time references, leading to look-ahead bias or incorrect calculations.

**Key Benefits:**
- Zero boilerplate for time context
- Impossible to accidentally use future data
- Same code works identically in backtest and live modes
- Automatic synchronization across multiple timeframes

### Crash-Safe Persistence

All state mutations are written atomically to disk before being considered complete. If the process crashes mid-operation, the system recovers to the last consistent state on restart.

**Problem Solved:** Trading systems that crash during position updates often leave corrupted state, causing duplicate trades, missed exits, or phantom positions.

**Key Solved Scenarios:**
- Process killed during order placement → position state unchanged
- Network failure during exchange API call → automatic retry on next tick
- Power outage during state save → recovery from last atomic write
- Out of memory error → graceful shutdown with state preserved

---

## Signal Lifecycle

### Signal Creation and Validation

Signals are created through the `getSignal` function and undergo comprehensive validation before being accepted:

**Validation Checks:**
- Take-profit and stop-loss prices are positive
- TP/SL relationship is correct (TP > entry for long, TP < entry for short)
- Risk/reward ratio meets minimum thresholds
- Timestamps are valid and not in the future
- Signal doesn't violate interval throttling rules

**Problem Solved:** Invalid signals cause undefined behavior, incorrect PnL calculations, and potential financial losses. Validation catches these issues before they reach the execution layer.

### Signal States and Transitions

Signals progress through well-defined states:

1. **Idle:** No active signal, strategy is monitoring
2. **Scheduled:** Signal created but waiting for entry price
3. **Pending:** Entry conditions met, waiting for execution
4. **Opened:** Position established, monitoring for exit
5. **Active:** Position in profit/loss, trailing stops active
6. **Closed:** Position exited (take-profit, stop-loss, manual, or timeout)

Each state has specific allowed operations and accessible data. Attempting invalid operations (like modifying a closed position) results in compile-time errors.

### Signal Cancellation and Timeout

Signals can be cancelled before execution due to:
- User intervention
- Risk management rejection
- Timeout (configurable wait period)
- Conflicting signal generation

Cancelled signals are tracked separately from closed positions, allowing analysis of signal quality and rejection patterns.

---

## Position Management

### Dollar Cost Averaging (DCA)

The framework supports sophisticated DCA strategies with automatic entry tracking:

**Features:**
- Multiple entry levels with individual cost basis
- Harmonic mean calculation for effective entry price
- Automatic rejection of averaging into winning positions (configurable)
- Partial close history affects future DCA calculations

**Problem Solved:** Manual DCA tracking requires complex bookkeeping. The framework automatically maintains entry history, calculates blended cost basis, and prevents common mistakes like averaging up.

**Key Mechanics:**
- Each DCA entry records price, cost, and timestamp
- Effective price recalculates after each entry
- Partial closes adjust cost basis proportionally
- Entry overlap detection prevents duplicate entries at similar prices

### Partial Profit and Loss

Positions can be partially closed at profit or loss milestones:

**Capabilities:**
- Close percentage of position at specific profit levels (10%, 20%, 30%, etc.)
- Close percentage at loss levels to limit exposure
- Track remaining cost basis after partial closes
- Calculate PnL for each partial independently

**Problem Solved:** All-or-nothing exits miss opportunities to lock in gains or limit losses. Partial closes allow fine-grained position management while maintaining exposure.

**Advanced Features:**
- Partial close history preserved for analysis
- Remaining position continues with adjusted cost basis
- PnL calculations account for all partials and remaining position
- Overlap detection prevents duplicate partials at similar prices

### Trailing Stop and Take Profit

Dynamic adjustment of exit levels based on price movement:

**Trailing Stop:**
- Stop-loss moves in favorable direction only
- Configurable distance from current price
- Automatic adjustment on each tick
- Prevents stop from moving backward

**Trailing Take:**
- Take-profit adjusts as price moves favorably
- Locks in gains while allowing upside
- Configurable trailing percentage
- Independent from initial take-profit

**Problem Solved:** Static exit levels either exit too early (missing trends) or too late (giving back gains). Trailing exits adapt to market conditions automatically.

### Breakeven Protection

Automatic adjustment of stop-loss to entry price once position reaches profitability threshold:

**Mechanism:**
- Calculates threshold to cover fees and slippage
- Moves stop-loss to entry price when threshold reached
- Prevents winning trade from becoming loser
- One-time adjustment per signal

**Problem Solved:** Traders often move stop-loss to breakeven manually, missing the optimal moment or forgetting entirely. Automation ensures protection is applied consistently.

---

## Risk Management

### Portfolio-Level Risk Checks

Risk validation operates across all strategies and symbols simultaneously:

**Capabilities:**
- Track total number of open positions across portfolio
- Enforce maximum concurrent position limits
- Custom validation functions for complex rules
- Rejection tracking with detailed reasons

**Problem Solved:** Individual strategy risk checks miss portfolio-level exposure. A trader might have 10 strategies each risking 10%, creating 100% total exposure unintentionally.

**Key Features:**
- Atomic check-and-reserve prevents race conditions
- Multiple strategies can share same risk profile
- Rejection events logged for analysis
- Configurable callbacks for rejection handling

### Custom Validation Rules

Developers define validation functions that receive:
- Pending signal details (entry, TP, SL, direction)
- Current market price
- Active position count
- Portfolio state

**Common Validations:**
- Minimum risk/reward ratio
- Maximum position size
- Time-of-day restrictions
- Correlation checks between symbols
- Volatility filters

**Problem Solved:** Hard-coded risk rules don't adapt to strategy-specific requirements. Custom validators allow arbitrary logic while maintaining framework guarantees.

### Risk Rejection Handling

When a signal violates risk rules:
- Signal is rejected before execution
- Rejection reason logged with full context
- Strategy continues operating normally
- Rejection statistics tracked for analysis

**Problem Solved:** Silent risk violations lead to unexpected exposure. Explicit rejection with logging provides visibility into risk management effectiveness.

---

## Data Integrity

### Look-Ahead Bias Prevention

The framework enforces strict temporal boundaries to prevent accidental use of future data:

**Mechanisms:**
- All candle requests aligned to interval boundaries
- Pending (incomplete) candles excluded from results
- Timestamp validation on every data access
- AsyncLocalStorage provides automatic context

**Problem Solved:** Look-ahead bias is the most common cause of unrealistic backtest results. Manual timestamp management inevitably leads to errors.

**Guarantees:**
- Impossible to request data beyond current simulation time
- Pending candles never included in calculations
- Multi-timeframe data automatically synchronized
- Same code produces identical results in backtest and live

### Candle Timestamp Alignment

All timestamps aligned down to interval boundaries:

**Example:**
- 15-minute interval
- Request at 00:17 → aligned to 00:15
- Request at 00:44 → aligned to 00:30

**Problem Solved:** Misaligned timestamps cause off-by-one errors, duplicate data, or missing candles. Automatic alignment ensures consistency.

**Key Rules:**
- First candle timestamp equals aligned start time
- Exactly requested number of candles returned
- Sequential timestamps with fixed interval
- Last candle timestamp calculated deterministically

### Order Book and Trade Data

Similar temporal protection for order book and aggregated trades:

**Order Book:**
- Configurable time offset window
- Aligned to offset boundaries
- Adapter receives time range for historical queries

**Aggregated Trades:**
- Always aligned to 1-minute boundaries
- Pagination for large requests
- Automatic slicing to requested limit

**Problem Solved:** Order book data without temporal context leads to using future snapshots. Time windows ensure only historical data accessible.

---

## Persistence and Storage

### Atomic File Writes

Default persistence uses atomic file operations:

**Mechanism:**
- Write to temporary file
- Rename to final location (atomic on most filesystems)
- Verify write completion
- Recover from incomplete writes on restart

**Problem Solved:** Partial writes during crashes corrupt state files, making recovery impossible. Atomic writes ensure all-or-nothing semantics.

### Pluggable Storage Adapters

Framework supports multiple storage backends through adapter pattern:

**Built-in Adapters:**
- File-based (default, zero configuration)
- In-memory (testing, fast iteration)
- MongoDB (production, scalable)
- Redis (caching, O(1) lookups)

**Problem Solved:** Different deployment scenarios require different storage strategies. Hard-coded storage limits flexibility and scalability.

**Adapter Capabilities:**
- 15 separate persistence contracts (signals, candles, logs, etc.)
- Each adapter implements all contracts
- Zero strategy code changes when switching backends
- Automatic adapter selection based on mode

### MongoDB and Redis Integration

Production-grade storage with advanced features:

**MongoDB Features:**
- Unique compound indexes prevent duplicates
- Atomic upserts with read-after-write consistency
- Queryable signal history
- Survives process restarts

**Redis Features:**
- O(1) lookups via context-key to ID mapping
- Cache miss fallback to MongoDB
- Automatic cache invalidation on writes
- Eliminates B-tree scans for repeated reads

**Problem Solved:** File-based storage becomes bottleneck at scale. Database backends provide durability, queryability, and performance for production deployments.

### Soft Delete Mechanism

Certain records use soft delete instead of physical deletion:

**Applies To:**
- Measure data (LLM responses, API results)
- Interval markers (processed time periods)
- Memory entries (agent context)

**Mechanism:**
- Set `removed: true` flag
- Listing operations filter on `removed: false`
- Physical file preserved for audit trail

**Problem Solved:** Physical deletion loses audit trail and can cause race conditions. Soft delete preserves history while hiding from active use.

---

## Broker Integration

### Transactional Commits

Every broker operation follows transactional pattern:

**Sequence:**
1. Framework prepares commit payload
2. Broker adapter executes exchange operation
3. If success → internal state updated
4. If failure → state unchanged, retry on next tick

**Problem Solved:** Exchange failures during position updates cause state desynchronization. Transactional commits ensure framework state matches exchange state.

### Rollback on Failure

When broker adapter throws exception:

**Behavior:**
- Internal state mutation skipped
- Position remains in previous state
- Error logged with full context
- Automatic retry on next tick

**Problem Solved:** Partial state updates leave system in inconsistent state. Rollback ensures atomicity of position changes.

**Example Scenarios:**
- Exchange API timeout → retry automatically
- Insufficient margin → reject and log
- Network failure → recover when connection restored
- Exchange rejection → state unchanged, strategy continues

### Retry Mechanism

Failed commits automatically retried:

**Mechanism:**
- Exception caught by framework
- State rollback completed
- Same commit attempted on next tick
- Continues until success or strategy cancelled

**Problem Solved:** Transient exchange failures shouldn't require manual intervention. Automatic retry handles temporary issues transparently.

**Guarantees:**
- No duplicate commits (state unchanged until success)
- No lost commits (retry continues indefinitely)
- Strategy can cancel pending commit
- Full audit trail of attempts

### Exchange Rejection Handling

Broker adapters can reject operations:

**Rejection Scenarios:**
- Insufficient balance
- Invalid order parameters
- Exchange-specific restrictions
- Rate limiting

**Handling:**
- Adapter throws exception with reason
- Framework rolls back state
- Error surfaced to user interface
- Strategy decides whether to retry or cancel

**Problem Solved:** Silent exchange rejections cause confusion about position state. Explicit rejection with error messages provides clarity.

---

## Reporting and Analytics

### Markdown Reports

Automatic generation of human-readable reports:

**Report Types:**
- Backtest summary (total PnL, win rate, Sharpe ratio)
- Signal lifecycle (all state transitions)
- Risk rejections (all blocked signals)
- Partial closes (all profit/loss milestones)
- Maximum drawdown (all peak-to-trough events)

**Problem Solved:** Manual report generation is time-consuming and error-prone. Automatic reports provide instant visibility into strategy performance.

**Features:**
- Customizable columns
- Configurable output format (file or JSONL)
- Organized by symbol, strategy, exchange
- Searchable and filterable

### JSONL Logging

Machine-readable logs for programmatic analysis:

**Capabilities:**
- Incremental writes (one event per line)
- Metadata included (symbol, strategy, timestamp)
- Searchable by multiple criteria
- Append-only (no overwrites)

**Problem Solved:** Binary log formats require custom parsers. JSONL provides human-readable, tool-friendly format for analysis.

**Use Cases:**
- Post-trade analysis
- Strategy optimization
- Anomaly detection
- Compliance auditing

### Performance Metrics

Comprehensive calculation of trading metrics:

**Metrics Calculated:**
- Total PnL (absolute and percentage)
- Win rate and loss rate
- Average win and loss size
- Sharpe ratio (risk-adjusted returns)
- Sortino ratio (downside risk-adjusted)
- Maximum drawdown (peak-to-trough decline)
- Recovery factor (profit / max drawdown)
- Expectancy (average profit per trade)

**Problem Solved:** Manual metric calculation is error-prone and incomplete. Built-in calculations ensure accuracy and consistency.

**Advanced Features:**
- Pooled metrics across multiple symbols
- Time-weighted returns
- Consecutive win/loss streaks
- Buyer/seller pressure analysis
- Trend classification with confidence

### Heatmap Visualization

Portfolio-wide performance visualization:

**Features:**
- Performance by symbol
- Cross-strategy comparison
- Risk-adjusted metrics
- Time-period analysis

**Problem Solved:** Tabular reports don't reveal patterns across symbols. Heatmaps provide instant visual identification of strengths and weaknesses.

---

## AI and LLM Integration

### Memory Adapters

Persistent storage for LLM context and reasoning:

**Capabilities:**
- Store conversation history per signal
- Search memory using BM25 algorithm
- List all memories up to timestamp
- Soft delete for audit trail

**Problem Solved:** LLM context lost between sessions prevents learning and continuity. Memory adapters preserve reasoning across restarts.

**Use Cases:**
- Store LLM reasoning for each trade
- Retrieve relevant past decisions
- Build knowledge base over time
- Debug LLM decision-making

### Session Management

Temporary storage for calculations and intermediate results:

**Features:**
- Scoped to symbol, strategy, exchange, timeframe
- Automatic cleanup on signal close
- Persistent across process restarts
- Prevents look-ahead bias

**Problem Solved:** Complex calculations repeated on every tick waste resources. Session storage caches results while maintaining temporal correctness.

**Example Scenarios:**
- Cache indicator calculations
- Store LLM inference results
- Maintain running statistics
- Track signal-specific metrics

### Agent Answer Dumping

Complete preservation of LLM interactions:

**Capabilities:**
- Store full conversation history
- Include system prompts and tool calls
- Attach to specific trading signals
- Searchable and exportable

**Problem Solved:** LLM decisions are opaque without full context. Complete conversation dumps enable debugging and improvement.

**Features:**
- Messages with roles (system, user, assistant, tool)
- Reasoning content preserved
- Tool call details included
- Images and attachments supported

### Multi-Provider Support

Unified interface for multiple LLM providers:

**Supported Providers:**
- OpenAI, Claude, DeepSeek, Grok
- Mistral, Perplexity, Cohere, Alibaba
- Hugging Face, Ollama (local)

**Features:**
- Automatic API key rotation
- Structured output enforcement
- Trading-specific prompts
- Fallback chains

**Problem Solved:** Different LLM providers have incompatible APIs. Unified interface allows easy switching and comparison.

---

## Developer Tools

### Command-Line Interface

Zero-boilerplate execution of strategies:

**Modes:**
- `--backtest`: Historical simulation
- `--paper`: Live prices, no real orders
- `--live`: Real trading with exchange
- `--walker`: A/B strategy comparison

**Features:**
- Automatic candle caching
- Web dashboard integration
- Telegram notifications
- Graceful shutdown on SIGINT

**Problem Solved:** Manual setup for each mode requires significant boilerplate. CLI provides one-command execution with all infrastructure handled.

### Web Dashboard

Real-time monitoring and manual control:

**Pages:**
- Main overview (all strategies and symbols)
- Signal status (pending, active, closed)
- Manual control (open, close, average, breakeven)
- Pine Script editor (interactive development)

**Problem Solved:** Console-only monitoring lacks visibility. Web dashboard provides real-time insights and manual intervention capabilities.

**Features:**
- Live signal updates
- Multi-timeframe charts
- Risk rejection tracking
- Position management buttons

### Telegram Notifications

Formatted alerts for trading events:

**Notifications:**
- Signal opened/closed
- Partial profit/loss
- Risk rejections
- Breakeven reached
- Trailing adjustments

**Problem Solved:** Missing trading events due to lack of monitoring. Telegram alerts ensure immediate awareness of important events.

**Features:**
- Price charts included
- Customizable templates
- Filter by event type
- HTML formatting

### Docker Support

Containerized deployment with automatic restarts:

**Capabilities:**
- Pre-configured docker-compose
- Environment variable configuration
- Automatic container restart
- Zero-downtime updates

**Problem Solved:** Manual deployment and process management is error-prone. Docker provides consistent, reliable execution environment.

**Features:**
- Separate containers for app, MongoDB, Redis
- Configurable via environment variables
- Health checks and restart policies
- Log aggregation

---

## Advanced Features

### Walker (A/B Testing)

Systematic comparison of multiple strategies:

**Workflow:**
1. Define multiple strategy variants
2. Run all on same historical data
3. Calculate metrics for each
4. Rank by optimization criterion
5. Generate comparison report

**Problem Solved:** Manual strategy comparison is time-consuming and inconsistent. Walker automates the entire process with statistical rigor.

**Features:**
- Configurable optimization metric (Sharpe, PnL, etc.)
- Progress tracking during execution
- Markdown comparison reports
- JSON export for further analysis

### Cron Scheduler

Periodic and fire-once job execution:

**Capabilities:**
- Periodic jobs (every hour, daily, etc.)
- Fire-once jobs (single execution)
- Symbol-specific or global scope
- Coordination across parallel backtests

**Problem Solved:** Manual scheduling leads to missed executions or duplicate runs. Cron ensures reliable, coordinated job execution.

**Features:**
- Virtual time alignment (backtest mode)
- Automatic retry on failure
- Generation tracking for re-registrations
- Mutex semantics for parallel execution

**Example Use Cases:**
- Fetch funding rates hourly
- Parse Telegram signals every 15 minutes
- Warm cache on startup
- Daily portfolio rebalancing

### Pine Script Support

Run TradingView indicators directly in Node.js:

**Features:**
- Pine Script v5/v6 compatibility
- 60+ built-in indicators
- File or code string input
- Plot extraction to signals

**Problem Solved:** Rewriting Pine Script indicators in JavaScript is time-consuming. Direct execution preserves existing work.

**Capabilities:**
- Native TradingView syntax
- Automatic indicator calculation
- Flexible signal mapping
- Cached execution for performance

### Multi-Timeframe Analysis

Synchronized data across multiple intervals:

**Mechanism:**
- Automatic alignment of all timeframes
- Consistent temporal boundaries
- No look-ahead bias possible
- Same code for all timeframes

**Problem Solved:** Manual multi-timeframe synchronization is error-prone. Automatic alignment ensures consistency.

**Supported Intervals:**
- 1m, 3m, 5m, 15m, 30m
- 1h, 2h, 4h, 6h, 8h
- 12h, 1d, 3d, 1w, 1M

---

## Performance Optimization

### Async Generators

Memory-efficient streaming for large datasets:

**Benefits:**
- No array accumulation
- Early termination support
- Constant memory usage
- Backpressure handling

**Problem Solved:** Loading entire history into memory causes out-of-memory errors. Streaming allows processing unlimited data.

**Use Cases:**
- Year-long backtests on 1-minute data
- Multi-symbol parallel execution
- Real-time signal processing
- Large portfolio analysis

### Memoization

Cache expensive calculations automatically:

**Capabilities:**
- Client instances cached by schema name
- Validation results memoized
- Connection reuse
- Time-based cache invalidation

**Problem Solved:** Repeated calculations waste CPU cycles. Memoization eliminates redundant work.

**Features:**
- Automatic cache key generation
- Configurable TTL
- Manual cache clearing
- Memory-efficient storage

### Prototype Methods

Memory-efficient method definitions:

**Mechanism:**
- Methods defined on prototype, not instances
- Shared across all instances of class
- Reduced memory footprint
- Faster instantiation

**Problem Solved:** Arrow functions in constructors create new function per instance. Prototype methods share single function across all instances.

**Impact:**
- 50-70% memory reduction for large object collections
- Faster garbage collection
- Better CPU cache utilization

### Bounded Queues

Prevent memory leaks from unbounded event accumulation:

**Mechanism:**
- Maximum queue size enforced
- Oldest events dropped when full
- Backpressure signals to producers
- Graceful degradation under load

**Problem Solved:** Unbounded queues grow indefinitely during high event rates, causing out-of-memory crashes. Bounded queues maintain stability.

**Configuration:**
- Configurable maximum size
- Drop policy (oldest, newest, etc.)
- Monitoring of queue depth
- Alerts on threshold breach

---

## Architecture

### Clean Architecture Layers

Strict separation of concerns:

**Client Layer:**
- Pure business logic
- No dependency injection
- Prototype methods for efficiency
- Direct data manipulation

**Service Layer:**
- Dependency injection container
- Organized by responsibility
- Schema, validation, connection services
- Global wrappers for public API

**Persistence Layer:**
- Atomic file operations
- Pluggable adapters
- Crash-safe writes
- Recovery mechanisms

**Event Layer:**
- Subject-based emitters
- Queued async processing
- Filter predicates
- Once listeners

**Problem Solved:** Monolithic architecture makes testing and modification difficult. Layered architecture enables independent development and testing.

### Dependency Injection

Custom DI container with Symbol-based tokens:

**Features:**
- Type-safe injection
- Lazy instantiation
- Scope management
- Test-friendly mocking

**Problem Solved:** Hard-coded dependencies prevent testing and flexibility. DI enables easy substitution and testing.

**Benefits:**
- Easy mocking for tests
- Flexible configuration
- Clear dependency graph
- Reduced coupling

### Context Propagation

Nested contexts using scoped services:

**Contexts:**
- ExecutionContext (symbol, timestamp, mode)
- MethodContext (strategy, exchange, frame)
- Automatic propagation through async calls

**Problem Solved:** Manual context passing creates verbose, error-prone code. Automatic propagation ensures correct context everywhere.

**Mechanism:**
- AsyncLocalStorage for execution context
- DI scopes for method context
- Automatic inheritance
- Isolation between parallel executions

### Registry Pattern

Centralized configuration management:

**Registries:**
- Strategy schemas
- Exchange schemas
- Frame schemas
- Risk schemas
- Sizing schemas

**Problem Solved:** Scattered configuration makes validation and discovery difficult. Centralized registries provide single source of truth.

**Features:**
- Type-safe registration
- Shallow validation on add
- Runtime existence checks
- Memoized validation results

---

## Testing and Reliability

### Comprehensive Test Coverage

775+ unit and integration tests:

**Test Categories:**
- Exchange helper functions
- Event listener system
- Signal validation logic
- PnL calculation accuracy
- Signal lifecycle verification
- Strategy callbacks
- Report generation

**Problem Solved:** Untested trading systems contain hidden bugs that cause financial losses. Comprehensive testing ensures reliability.

**Testing Patterns:**
- Unique names per test (prevent cross-contamination)
- Mock candle generators
- Async coordination utilities
- Background execution with event detection

### Validation Services

Runtime validation with memoization:

**Validations:**
- Strategy existence and configuration
- Exchange schema correctness
- Frame definition validity
- Risk profile completeness
- Sizing method parameters

**Problem Solved:** Invalid configurations cause runtime errors or incorrect behavior. Validation catches issues before execution.

**Features:**
- Shallow validation on registration
- Deep validation before execution
- Memoized results (performance)
- Clear error messages

### Error Handling

Graceful degradation on failures:

**Strategies:**
- ActionProxy wraps all handlers
- Errors logged, execution continues
- Rollback on state mutations
- Automatic retry on transient failures

**Problem Solved:** Unhandled exceptions crash trading systems. Graceful error handling maintains operation through failures.

**Mechanisms:**
- Try-catch at every boundary
- Error events for monitoring
- State rollback on failure
- Retry with backoff

---

## Deployment Scenarios

### Single Strategy Development

Quick iteration on individual strategies:

**Workflow:**
1. Create strategy file
2. Run backtest with CLI
3. Analyze reports
4. Iterate on logic
5. Deploy to paper trading
6. Move to live when validated

**Problem Solved:** Complex setup slows development. Simple workflow enables rapid iteration.

### Multi-Strategy Monorepo

Manage multiple strategies in single repository:

**Structure:**
- Shared packages (brokers, signals, utilities)
- Individual strategy directories
- Isolated resources per strategy
- Shared infrastructure (MongoDB, Redis)

**Problem Solved:** Separate repositories for each strategy create maintenance burden. Monorepo enables code sharing and consistent tooling.

**Features:**
- Automatic import aliases
- Per-strategy environment variables
- Isolated dump directories
- Shared broker adapters

### Parallel Symbol Execution

Run same strategy across multiple symbols simultaneously:

**Capabilities:**
- Single Node.js process
- Shared event loop
- Common infrastructure
- Independent state per symbol

**Problem Solved:** Separate processes for each symbol waste resources. Parallel execution maximizes hardware utilization.

**Performance:**
- ~6,300× real-time aggregate speed
- ~703× per-symbol replay speed
- ~103 events/second in hot loops
- Constant memory usage

### Production Deployment

Reliable live trading with monitoring:

**Components:**
- Docker containers for isolation
- MongoDB for persistence
- Redis for caching
- Web dashboard for monitoring
- Telegram for alerts

**Problem Solved:** Ad-hoc deployments are unreliable. Structured deployment ensures stability and observability.

**Features:**
- Automatic restart on failure
- Health checks and monitoring
- Log aggregation
- Zero-downtime updates

---

## Common Patterns

### Signal Generation

Typical signal generation workflow:

1. Fetch multi-timeframe data
2. Calculate indicators
3. Apply entry logic
4. Validate signal parameters
5. Return signal or null

**Best Practices:**
- Use framework data fetching (prevents look-ahead bias)
- Validate all parameters before return
- Include descriptive notes
- Handle errors gracefully

### Position Management

Common position management patterns:

**Partial Profit:**
- Close percentage at profit milestones
- Lock in gains while maintaining exposure
- Adjust remaining cost basis

**DCA Entry:**
- Add to position on favorable moves
- Track multiple entry levels
- Calculate blended entry price

**Trailing Exit:**
- Adjust stop/take as price moves
- Lock in gains dynamically
- Prevent giving back profits

### Risk Management

Typical risk management setup:

1. Define risk profile with validations
2. Attach profile to strategy
3. Framework checks before every signal
4. Rejections logged automatically
5. Statistics available for analysis

**Common Validations:**
- Minimum risk/reward ratio
- Maximum position size
- Time-of-day restrictions
- Volatility filters

---

## Learning Curve: A "Fool-Proof" Architecture

```typescript
import {
  addStrategySchema,
  listenError,
  listenActivePing,
  Log,
  Position,
  commitClosePending,
  getPositionPnlPercent,
  getPositionEntryOverlap,
  getPositionEntries,
  commitAverageBuy,
} from "backtest-kit";
import { errorData, getErrorMessage, str } from "functools-kit";

const HARD_STOP = 25.0;
const TARGET_PROFIT = 3;

const LADDER_STEP_COST = 100;
const LADDER_UPPER_STEP = 5;
const LADDER_LOWER_STEP = 1;

const LADDER_MAX_STEPS = 10;

addStrategySchema({
  strategyName: "apr_2026_strategy",
  getSignal: async (symbol, when, currentPrice) => {
    return {
      position: "long",
      ...Position.moonbag({
        position: "long",
        currentPrice,
        percentStopLoss: HARD_STOP,
      }),
      minuteEstimatedTime: Infinity,
      cost: LADDER_STEP_COST,
    };
  },
});

listenActivePing(async ({ symbol, currentPrice }) => {
  const { length: steps } = await getPositionEntries(symbol);
  if (steps >= LADDER_MAX_STEPS) {
    return;
  }
  const hasOverlap = await getPositionEntryOverlap(symbol, currentPrice, {
    upperPercent: LADDER_UPPER_STEP,
    lowerPercent: LADDER_LOWER_STEP,
  });
  if (hasOverlap) {
    return;
  }
  await commitAverageBuy(symbol, LADDER_STEP_COST);
});


listenActivePing(async ({ symbol, data, timestamp }) => {
  console.log(new Date(timestamp));
  const currentProfit = await getPositionPnlPercent(symbol);
  if (currentProfit < TARGET_PROFIT) {
    return;
  }
  Log.info("position closed due to the target pnl reached", {
    symbol,
    data,
  });
  await commitClosePending(symbol, {
    id: "unknown",
    note: str.newline(
      "# Closed by target pnl",
    ),
  });
});

listenError((error) => {
  console.log(error);
  Log.debug("error", {
    error: errorData(error),
    message: getErrorMessage(error),
  });
});
```

A common critique of trading frameworks is their steep learning curve and the ease with which a developer can introduce catastrophic bugs. Backtest Kit is intentionally designed with a **"fool-proof" API surface**. It restricts dangerous primitives and forces the developer into the "pit of success," making it practically impossible to shoot yourself in the foot.

### 1. Ambient Temporal Context (No More Manual Time Passing)
In traditional frameworks, you must manually pass the `currentDate` or `timestamp` through every function call. If you miss a parameter, you accidentally leak future data into your indicators, ruining the backtest. 
Backtest Kit uses Node.js `AsyncLocalStorage` to create an **ambient execution context**. When you request candles or order book data, the engine automatically knows exactly what "now" is. You cannot accidentally query future data because the engine physically blocks it at the adapter level.

### 2. Type-Safe State Machines
Trading signals have a strict lifecycle: *Idle → Scheduled → Pending → Opened → Active → Closed*. 
Using TypeScript's Discriminated Unions, the framework makes invalid states unrepresentable. You cannot accidentally call a `closePosition` method on a signal that is already closed, nor can you modify the entry price of an active trade. The compiler enforces the lifecycle, eliminating entire classes of runtime state-corruption bugs.

### 3. Guarded Dollar-Cost Averaging (DCA)
Manually implementing DCA is a mathematical nightmare. Developers often accidentally "average up" (buying at a higher price than the current effective entry), which destroys the position's profitability. Backtest Kit's DCA engine uses a **harmonic mean** calculation and physically rejects any `commitAverageBuy` call that would worsen the effective entry price. 

### 4. Transactional Broker Commits (The "No-Try-Catch" Rule)
In live trading, if the exchange rejects an order (e.g., insufficient margin, rate limit), your internal bot state and the exchange state become desynchronized. Usually, developers write complex `try/catch` blocks to manually roll back variables. 
In Backtest Kit, **the broker adapter intercepts every mutation before the internal state changes**. If the exchange throws an error, the framework automatically rolls back the internal state and retries on the next tick. You never have to write rollback logic.

### 5. Automatic Signal Validation
Before a signal even reaches the execution engine, it passes through a rigorous validation pipeline. The framework checks if Take-Profit (TP) and Stop-Loss (SL) prices are logically sound, if the Risk/Reward ratio meets your criteria, and if the signal violates any interval throttling rules. Invalid signals are rejected silently or logged, preventing them from crashing the strategy.

---

## How Backtest Kit Compares to the Competition

Most trading frameworks force you to choose between research speed and production reliability. Backtest Kit bridges this gap by providing a unified engine that handles everything from historical simulation to live execution. Here is how it stacks up against the industry standards.

### vs. Backtrader (Python)
**The Paradigm Shift: From Fragile Scripts to Type-Safe Systems**
Backtrader is the grandfather of Python backtesting, but it relies heavily on dynamic typing and global state. This makes it notoriously easy to introduce look-ahead bias or state corruption, as the framework rarely stops you from making logical errors. 
* **The Backtest Kit Advantage:** Backtest Kit enforces a strict, type-safe state machine (`idle → opened → active → closed`) using TypeScript's discriminated unions. The compiler physically prevents you from accessing closed position data or modifying an active trade as if it were closed. Furthermore, Backtest Kit's ambient temporal context automatically blocks look-ahead bias at the adapter level.

### vs. VectorBT (Python)
**The Paradigm Shift: From Static Simulation to Live Execution**
VectorBT is incredibly fast for matrix-based, single-pass historical simulations. However, it is fundamentally a research tool. It lacks a built-in execution engine for live trading, meaning you have to build a completely separate, custom system to actually deploy your strategy to an exchange.
* **The Backtest Kit Advantage:** Backtest Kit is not just a simulator; it is a unified execution engine. The exact same code that runs a historical backtest can be deployed to live trading with a single configuration change. Its crash-safe persistence, atomic file writes, and transactional broker commits ensure that your live bot can recover from network failures or process crashes without losing track of open positions.

### vs. MetaTrader / MQL5
**The Paradigm Shift: From Platform Lock-in to a Modern, Open Stack**
MetaTrader is a closed ecosystem. MQL5 is a legacy language, and your strategies are trapped inside a Windows-centric desktop application. Integrating with modern data sources, AI models, or custom REST/WebSocket APIs requires painful workarounds and external bridges.
* **The Backtest Kit Advantage:** Backtest Kit is built on a modern, open-source stack (Node.js/TypeScript). You own the entire infrastructure. You can seamlessly integrate local LLMs (like Ollama), fetch data from any custom API, or deploy your bot in a Docker container on a Linux server. There is no vendor lock-in, no platform fees, and no reliance on a proprietary desktop GUI.

### vs. QuantConnect / Lean (C#)
**The Paradigm Shift: From Cloud Dependency to Self-Hosted Sovereignty**
QuantConnect (and its open-source Lean engine) is powerful, but it heavily pushes you toward their cloud infrastructure and data ecosystem. C# adds compilation overhead, and debugging complex multi-timeframe strategies in a web IDE can be frustrating.
* **The Backtest Kit Advantage:** Backtest Kit is 100% self-hosted and has zero dependencies on third-party cloud platforms. Your code, your data, and your execution environment stay entirely on your own machines. The TypeScript ecosystem allows for rapid iteration without compilation steps, and the built-in Web UI, CLI, and Telegram notifications provide a superior, modern developer experience for debugging and monitoring live bots.

### vs. Freqtrade (Python)
**The Paradigm Shift: From Simple Grids to Complex Position Management**
Freqtrade is an excellent open-source crypto trading bot, but its strategy logic is relatively flat. It excels at simple entry/exit signals but struggles with complex, multi-layered position management.
* **The Backtest Kit Advantage:** Backtest Kit treats position management as a first-class citizen. It natively supports complex Dollar-Cost Averaging (DCA) ladders, partial profit/loss taking, trailing stops, and breakeven protection—all with mathematically accurate cost-basis tracking. If your strategy requires dynamically adjusting a position over time rather than just "buying and holding until a stop-loss hits," Backtest Kit provides the enterprise-grade math to handle it flawlessly.

### The Bottom Line
If you are building a quick research prototype or running a simple moving-average crossover in Python, tools like VectorBT or Backtrader are hard to beat for raw speed. 

But the moment you need to **deploy a strategy to production**, manage **complex position sizing**, integrate **AI agents**, or guarantee that a **network outage won't desync your bot from the exchange**, Backtest Kit provides the architectural guardrails that legacy frameworks simply do not have. It is the bridge between a Jupyter Notebook experiment and a resilient, commercial-grade trading desk.

---

## Troubleshooting

### Common Issues

**Look-Ahead Bias:**
- Symptom: Unrealistic backtest results
- Cause: Using future data accidentally
- Solution: Use framework data fetching functions

**State Desynchronization:**
- Symptom: Framework state doesn't match exchange
- Cause: Exchange failure during state update
- Solution: Use transactional broker commits

**Memory Leaks:**
- Symptom: Increasing memory usage over time
- Cause: Unbounded event accumulation
- Solution: Use bounded queues, dispose resources

**Performance Degradation:**
- Symptom: Slow execution over time
- Cause: Repeated expensive calculations
- Solution: Use memoization, cache results

### Debugging Techniques

**Enable Verbose Logging:**
- Set log level to debug
- Review all state transitions
- Check validation errors

**Use Web Dashboard:**
- Monitor signals in real-time
- Inspect position details
- Trigger manual operations

**Analyze Reports:**
- Review Markdown reports
- Check JSONL logs
- Examine heatmap visualizations

---

## Migration Guide

### From Other Frameworks

**Key Differences:**
- Strict state machine (no ambiguous states)
- Automatic temporal context (no manual timestamps)
- Transactional commits (no partial updates)
- Comprehensive validation (no invalid signals)

**Migration Steps:**
1. Identify signal generation logic
2. Map to `getSignal` function
3. Configure risk management
4. Set up broker adapter
5. Test in paper mode
6. Deploy to live

**Benefits:**
- Eliminate state corruption bugs
- Prevent look-ahead bias
- Ensure exchange synchronization
- Improve code maintainability

---

## Universal LLM Inference Adapter

Building AI-powered trading strategies typically requires integrating with multiple LLM providers, each with its own API format, authentication method, and response structure. Developers end up writing boilerplate code to handle OpenAI's format, then Claude's format, then DeepSeek's format — and managing API key rotation, structured output enforcement, and error handling across all of them.

**The Backtest Kit Solution:** `@backtest-kit/ollama` provides a **universal inference adapter** that abstracts away provider differences, letting you switch between 10+ LLM providers with a single line of code while maintaining type-safe, structured output.

### Multi-Provider Abstraction
Instead of learning and maintaining separate SDKs for each provider, you use a unified Higher-Order Function (HOF) API. Wrap your signal generation function with `deepseek()`, `claude()`, `gpt5()`, or `ollama()`, and the adapter handles provider-specific formatting, authentication, and error handling automatically.

**Supported Providers:**
- OpenAI (GPT-5, GPT-4o)
- Anthropic Claude
- DeepSeek
- Grok (xAI)
- Mistral
- Perplexity
- Cohere
- Alibaba (Qwen)
- Hugging Face
- Ollama (local models)
- GLM-4 (Zhipu AI)

**Problem Solved:** Provider lock-in and API fragmentation. You can test your strategy with DeepSeek during development, switch to Claude for production, and fall back to local Ollama if cloud APIs are unavailable — all without changing your strategy code.

### Structured Output Enforcement
Trading signals require specific fields: position direction, entry price, take-profit, stop-loss, risk notes. Raw LLM outputs are unpredictable — they might return markdown, JSON with missing fields, or hallucinated values.

The adapter integrates with `agent-swarm-kit` to enforce **Zod schemas** or JSON schema validation. You define the expected structure once, and the adapter guarantees every LLM response conforms to it before reaching your strategy logic.

**Key Features:**
- Zod schema validation with automatic retry on malformed output
- Custom validation rules (e.g., "stop-loss must be below entry for LONG")
- Trading-specific field descriptions injected into prompts
- Type-safe TypeScript interfaces generated from schemas

**Problem Solved:** Unstructured LLM outputs crash strategies or generate invalid trades. Schema enforcement ensures every signal is valid before execution.

### Token Rotation and Fallback Chains
Cloud LLM APIs have rate limits, and single API keys can exhaust quotas during intensive backtesting or live trading. The adapter supports **automatic token rotation** — pass an array of API keys, and the system rotates through them to distribute load.

For production resilience, you can configure **fallback chains**: if DeepSeek is unavailable, automatically retry with Claude; if Claude fails, fall back to local Ollama. This ensures your strategy never stops generating signals due to provider outages.

**Problem Solved:** Rate limiting and provider downtime halt trading. Token rotation and fallback chains maintain continuous operation.

### Userspace Prompts and Memoization
Prompt engineering is iterative — you need to tweak system prompts, add context, and test variations. The adapter loads prompts from `.cjs` modules in your `config/prompt/` directory, making them easy to edit without touching strategy code.

To avoid redundant API calls during backtesting (where the same candle data might be analyzed multiple times), the adapter **memoizes prompt modules** using `functools-kit`. Identical inputs return cached responses, dramatically reducing API costs and latency.

**Problem Solved:** Prompt management scattered across strategy files, and redundant API calls during backtesting. Centralized prompt modules with memoization solve both issues.

### Trading-Specific Context Injection
Unlike generic LLM wrappers, `@backtest-kit/ollama` is designed for trading. System prompts automatically receive context about the current symbol, strategy name, exchange, timeframe, and whether you're in backtest or live mode. This allows your prompts to adapt dynamically without manual string interpolation.

**Problem Solved:** Generic LLM adapters don't understand trading context. Trading-specific prompts produce more relevant signals and risk assessments.

---

## Production Strategy Examples

Most trading frameworks provide toy examples — moving average crossovers on daily data that work in tutorials but fail in production. Backtest Kit includes **real-world strategy examples** that demonstrate advanced position management, multi-timeframe analysis, AI integration, and alternative signal sources. Each example is fully documented with backtest results, Sharpe ratios, and implementation details.

### Neural Network Strategy (October 2021)
**Signal Source:** TensorFlow feed-forward neural network trained on normalized candle patterns

**How It Works:**
- Every 8 hours, the strategy fetches 58 candles (50 for training, 8 for prediction)
- A neural network (8→6→4→1 architecture) is trained on normalized data, where each candle's close is mapped to [0,1] representing its position within the high-low range
- Every 15 minutes, if the current price is below the predicted price, a LONG position is opened with a 1% hard stop
- Positions exit via trailing take-profit: when profit retraces 1% from its peak (e.g., position hits +3%, closes at +2%)

**Results:** +18.26% PNL, Sharpe Ratio 0.31

**Key Techniques Demonstrated:**
- Real-time model training within strategy execution
- Normalization for stable neural network inputs
- Trailing take-profit for dynamic exits
- Integration with TensorFlow.js in Node.js

**Problem Solved:** Static technical indicators fail to adapt to changing market regimes. Neural networks can learn non-linear patterns, but integrating them into a trading system requires careful handling of training data, prediction timing, and position management.

---

### Pine Script Range Breakout (December 2025)
**Signal Source:** TradingView Pine Script indicator executed via `@backtest-kit/pinets`

**How It Works:**
- Every hour, the strategy runs `btc_dec2025_range.pine` on 1h candles
- The Pine Script extracts: Bollinger Bands, range boundaries, signal direction (±1), ranging flag, and volume spike detection
- A signal fires on `signal === 1` (LONG) or `signal === -1` (SHORT), but only if the price hasn't already moved past the signal close and the market isn't ranging
- Each position uses a fixed ±2% bracket (TP and SL), no DCA, no trailing

**Results:** +2.40% PNL, Sharpe Ratio 0.06

**Key Techniques Demonstrated:**
- Running TradingView Pine Script directly in Node.js
- Extracting multiple outputs from a single Pine indicator
- Filtering signals based on market state (ranging vs. trending)
- Fixed bracket exits for mean-reversion strategies

**Problem Solved:** Traders have thousands of hours invested in Pine Script indicators. Rewriting them in JavaScript is time-consuming and error-prone. Direct Pine Script execution preserves existing work while enabling backtesting and live deployment.

---

### Signal Inversion Strategy (January 2026)
**Signal Source:** Real Telegram channel signals (Crypto Yoda), inverted

**How It Works:**
- Signals are loaded from `assets/entry.jsonl` — 11 real posts from a Telegram channel, exported verbatim
- On each candle, `getSignal` checks if `publishedAt` matches the current minute and whether `closePrice` falls inside `entry.from..entry.to`
- The strategy enters **counter-trend** (inverts the channel's direction) with trailing take-profit and no fixed TP
- Stop-loss is set to -0.5%

**Results:** +8.58% PNL, **Sharpe Ratio 1.14** (highest in the example set)

**Key Techniques Demonstrated:**
- Exploiting predictable signal sources (channels with algorithmic posting)
- Counter-trend entry based on liquidity harvesting
- Trailing take-profit for maximizing gains
- Loading external signal data into backtest

**Problem Solved:** Many Telegram channels publish signals with poor risk-reward ratios and algorithmic timing. By reverse-engineering the algorithm and entering counter-trend, you can harvest the liquidity created by followers blindly copying the signals.

---

### AI News Sentiment Strategy (February 2026)
**Signal Source:** LLM analysis of live crypto/macro news via Tavily + Ollama

**How It Works:**
- Every 4–8 hours, a Tavily search fetches the latest Bitcoin and macro headlines
- The raw news text is passed to a local Ollama model, which returns one of `bullish`, `bearish`, or `wait`
- `getSignal` opens a LONG on `bullish`, SHORT on `bearish`, and skips on `wait`
- A conflicting forecast while a position is open triggers `commitClosePending` (sentiment flip)
- Positions exit on trailing take-profit (1% drawdown from peak) or stop-loss (1% from entry)

**Results:** +16.99% PNL, Sharpe Ratio 0.25 (during a month where BTC fell -16.4%)

**Key Techniques Demonstrated:**
- Integrating LLM inference into signal generation
- Using external news APIs (Tavily) for real-time data
- Sentiment-based position flipping
- Trailing exits for trending markets

**Problem Solved:** Technical indicators lag price action. News sentiment can provide leading signals, but parsing news manually is impractical. LLMs can analyze headlines in real-time and generate directional bias, enabling news-driven strategies.

---

### SHORT DCA Ladder (March 2026)
**Signal Source:** Fixed SHORT signal with dynamic Dollar-Cost Averaging

**How It Works:**
- `getSignal` opens a SHORT on every new pending signal via `Position.moonbag` with a 25% hard stop and $100 cost
- While active, `commitAverageBuy` fires on each ping if the current price moves outside a ±1–5% band around the last entry and fewer than 10 rungs have been added
- The position closes as soon as blended portfolio PNL reaches +0.5% via `commitClosePending`

**Results:** +37.83% PNL, Sharpe Ratio 0.35

**Key Techniques Demonstrated:**
- Dynamic DCA ladder with overlap detection
- Blended cost basis tracking across multiple entries
- Target-based exit (close at specific PNL threshold)
- Short-biased mean-reversion strategy

**Problem Solved:** Single-entry SHORT positions in volatile markets often get stopped out before the reversal. DCA ladders allow you to average into the position as price spikes, lowering the blended entry and increasing the probability of hitting a small profit target on the reversal.

---

### LONG DCA Ladder (April 2026)
**Signal Source:** Fixed LONG signal with dynamic Dollar-Cost Averaging

**How It Works:**
- Same mechanics as SHORT version but LONG-biased with a 3% profit target
- Deployed 2.4 entries per trade on average
- Achieved +67.85% PNL on deployed capital with improved percentage drawdown (-2.59% vs -3.99% without DCA)

**Results:** +67.85% PNL, Sharpe Ratio 0.12

**Key Techniques Demonstrated:**
- Long-biased DCA in trending bull markets
- Comparison of DCA vs. single-entry performance
- Drawdown analysis with and without averaging

**Problem Solved:** In trending markets, single entries miss opportunities to add to winning positions. DCA ladders allow you to scale into trends while maintaining a favorable risk-reward profile.

---

### Python EMA Crossover (February 2021)
**Signal Source:** Python-based EMA crossover executed via WebAssembly (WASI)

**How It Works:**
- Every 8 hours, `Cache.fn` runs the Python indicator (`strategy.py`) on 8h candles to calculate EMA(9) and EMA(21)
- A signal fires based on EMA crossover and 4h range midpoint confirmation: if EMA(9) > EMA(21), open LONG; otherwise SELL
- Each signal opens a $100 bracket position via `Position.bracket` with ±2% take-profit and stop-loss
- The strategy deployed $3,300 across 33 trades (all LONG), achieving +$5.52 (+0.17%) with a 63.6% win rate

**Results:** +5.52% PNL, Sharpe Ratio 0.09

**Key Techniques Demonstrated:**
- Running Python indicators in Node.js via WebAssembly
- Multi-timeframe confirmation (8h EMA + 4h range)
- Bracket orders for fixed risk-reward
- Integration with Python ecosystem from TypeScript

**Problem Solved:** Python has a rich ecosystem of technical analysis libraries (pandas, ta-lib, scikit-learn), but integrating them into Node.js trading systems is difficult. WebAssembly execution allows you to run Python code directly in your TypeScript strategies without rewriting logic.

---

### Polymarket Δprob Strategy (April 2024)
**Signal Source:** Prediction market probability shifts from Polymarket

**How It Works:**
- `loadPolySignals` reads `assets/polymarket-backtest-result.json` once via `singleshot`, aggregates to one signal per day (max `|dprob|` across all crypto-prices markets), and strips `entryPrice`/`exitPrice` (future-data fields)
- `getSignal` picks the most recent signal with `timestamp ≤ when` and rejects it if older than 1h or `|dprob| < 0.10`
- Positive Δprob → LONG, negative → SHORT
- Entry via `Position.moonbag` at market with a 1% hard stop and no fixed TP
- `listenActivePing` closes on 1% trailing drawdown from peak profit, or on the 24h timeout

**Results:** 10 trades, 70% WR, Sharpe +0.065 — three SL hits (one per LONG on the April top) nearly cancel seven trailing-take SHORT wins on the recovery slope

**Key Techniques Demonstrated:**
- Using prediction markets as leading indicators
- Aggregating signals across multiple markets
- Time-based signal expiration
- Trailing exits for asymmetric risk-reward

**Problem Solved:** Prediction markets aggregate crowd wisdom about future events. Sharp shifts in probability can reflect retail sentiment flow that precedes spot price movement. By tracking Δprob (change in probability), you can exploit this leading indicator without look-ahead bias.

---

### What These Examples Demonstrate

Each strategy example showcases a different aspect of Backtest Kit's capabilities:

**Advanced Position Management:**
- DCA ladders with overlap detection and blended cost basis
- Partial profit/loss with dynamic cost adjustment
- Trailing take-profit and stop-loss
- Breakeven protection

**Multi-Source Signal Generation:**
- Technical indicators (EMA, Bollinger Bands, RSI)
- Neural networks and machine learning
- LLM-powered sentiment analysis
- External data sources (Telegram, Polymarket, news APIs)
- Pine Script indicators from TradingView

**Realistic Backtesting:**
- Look-ahead bias prevention
- Accurate PNL with fees and slippage
- Multi-timeframe synchronization
- Crash-safe persistence

**Production Deployment:**
- Same code for backtest and live
- Transactional broker commits
- Risk management and validation
- Event-driven monitoring

These examples are not toy demonstrations — they are **production-quality strategies** that have been backtested on real historical data with documented results. You can clone the repository, run the backtests, and see the exact performance metrics for yourself.

## Conclusion

Backtest Kit solves the fundamental challenges of building reliable trading systems through:

- **Type-safe state machines** that eliminate state corruption
- **Automatic temporal context** that prevents look-ahead bias
- **Transactional commits** that ensure exchange synchronization
- **Comprehensive validation** that catches errors before execution
- **Pluggable architecture** that adapts to different deployment scenarios
- **Production-grade persistence** that survives crashes and restarts

The framework is battle-tested through 775+ tests, real-world trading deployments, and continuous iteration based on user feedback. It provides the reliability and correctness required for production trading while maintaining the flexibility needed for strategy research and development.

Whether building a single strategy or a multi-symbol trading desk, Backtest Kit provides the tools and guarantees needed to move from research to production with confidence.