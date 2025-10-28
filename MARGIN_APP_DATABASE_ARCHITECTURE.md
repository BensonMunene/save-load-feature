# Margin App Database Architecture
## Complete Guide with Flowcharts & Diagrams

**Document Version:** 1.0  
**Last Updated:** October 28, 2025  
**Total Pages:** 3

---

# 📊 PAGE 1: DATABASE OVERVIEW & ARCHITECTURE

## Executive Summary

The Margin App uses **ONE SQLite database** (`db.sqlite3`) containing **4 database tables** across 3 Django apps. The database stores margin calculations, backtest sessions, temporary cache, and user-saved backtest results.

### Database Count: **1 Database**
- **Name:** `db.sqlite3`
- **Location:** `margin_app/db.sqlite3`
- **Engine:** SQLite3 (Development) / PostgreSQL (Production-ready)
- **Size:** ~5-50 MB (varies with saved results)

---

## 🏗️ Complete Database Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          MARGIN APP DATABASE                                │
│                         File: db.sqlite3                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  TABLE 1: margin_calculator_margincalculation                         │ │
│  │  App: margin_calculator                                               │ │
│  │  Purpose: Store margin calculation history                            │ │
│  │  ─────────────────────────────────────────────────────────────────    │ │
│  │  Fields:                                                               │ │
│  │  • id (PK)                    - Primary key                            │ │
│  │  • user_id (FK)               - User who ran calculation (nullable)   │ │
│  │  • ticker                     - Stock symbol (e.g., "SPY")            │ │
│  │  • investment_amount          - Dollar amount invested                 │ │
│  │  • leverage                   - Leverage used (1.0 - 7.0x)            │ │
│  │  • current_price              - Stock price at calculation            │ │
│  │  • account_type               - "reg_t" or "portfolio"                │ │
│  │  • position_type              - "Long" or "Short"                     │ │
│  │  • interest_rate              - Annual interest % (1-10%)             │ │
│  │  • holding_period             - Months (1-60)                         │ │
│  │  • results (JSON)             - Complete calculation results          │ │
│  │  • created_at                 - Timestamp                             │ │
│  │  • updated_at                 - Last modified                         │ │
│  │                                                                         │ │
│  │  Indexes: [user, created_at], [ticker, created_at]                    │ │
│  │  Size per record: ~1-2 KB                                             │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  TABLE 2: historical_backtest_backtestcache                           │ │
│  │  App: historical_backtest                                             │ │
│  │  Purpose: Temporary cache for backtest results (performance)          │ │
│  │  ─────────────────────────────────────────────────────────────────    │ │
│  │  Fields:                                                               │ │
│  │  • id (PK)                    - Primary key                            │ │
│  │  • cache_key (UNIQUE)         - MD5 hash of parameters                │ │
│  │  • etf                        - ETF ticker (SPY, VTI)                 │ │
│  │  • mode                       - Backtest mode                          │ │
│  │  • parameters (JSON)          - Input parameters                       │ │
│  │  • results (JSON)             - Complete backtest results             │ │
│  │  • created_at                 - When cached                           │ │
│  │  • expires_at                 - Cache expiration (24 hrs)             │ │
│  │                                                                         │ │
│  │  Indexes: [etf, mode, created_at], [expires_at]                       │ │
│  │  TTL: 24 hours (auto-deleted)                                         │ │
│  │  Size per record: ~2-4 MB                                             │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  TABLE 3: historical_backtest_backtestsession                         │ │
│  │  App: historical_backtest                                             │ │
│  │  Purpose: Track backtest execution sessions (analytics)               │ │
│  │  ─────────────────────────────────────────────────────────────────    │ │
│  │  Fields:                                                               │ │
│  │  • id (PK)                    - Primary key                            │ │
│  │  • session_id (UNIQUE)        - UUID for session                       │ │
│  │  • mode                       - liquidation_reentry, etc.             │ │
│  │  • etf                        - SPY or VTI                            │ │
│  │  • start_date                 - Backtest start                        │ │
│  │  • end_date                   - Backtest end                          │ │
│  │  • parameters (JSON)          - All backtest parameters                │ │
│  │  • status                     - pending, running, completed, error    │ │
│  │  • created_at                 - Session start time                    │ │
│  │  • completed_at               - Session end time                      │ │
│  │  • execution_time_ms          - Processing duration                   │ │
│  │                                                                         │ │
│  │  Indexes: [session_id], [created_at], [status]                        │ │
│  │  Size per record: ~500 bytes                                          │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  TABLE 4: historical_backtest_savedbacktestresult                     │ │
│  │  App: historical_backtest.saved_results                               │ │
│  │  Purpose: User's permanently saved backtest results                   │ │
│  │  ─────────────────────────────────────────────────────────────────    │ │
│  │  Fields:                                                               │ │
│  │  • id (PK)                    - Primary key                            │ │
│  │  • result_id (UNIQUE)         - UUID for external reference           │ │
│  │  • user_id (INDEXED)          - Owner from umbrella_app               │ │
│  │  • user_email                 - User's email                          │ │
│  │  • name                       - User-friendly name                    │ │
│  │  • description                - Optional notes                        │ │
│  │  • mode                       - Backtest mode                          │ │
│  │  • ticker                     - ETF symbol                            │ │
│  │  • start_date                 - Backtest period start                 │ │
│  │  • end_date                   - Backtest period end                   │ │
│  │  • leverage                   - Leverage used                         │ │
│  │  • account_type               - reg_t or portfolio                    │ │
│  │  • initial_investment         - Starting equity                       │ │
│  │  • profit_threshold           - Rebalance threshold (nullable)        │ │
│  │  • results_data (JSON)        - COMPLETE backtest output (2-4 MB)    │ │
│  │  • created_at                 - Save timestamp                        │ │
│  │  • updated_at                 - Last modified                         │ │
│  │                                                                         │ │
│  │  Indexes: [user_id, created_at], [user_id, ticker], [user_id, mode]  │ │
│  │  Quota: 50 results per user                                           │ │
│  │  Size per record: 2-4 MB                                              │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 📈 Database Size Estimates

```
┌──────────────────────────────────────────────────────────────┐
│  TABLE SIZE BREAKDOWN                                        │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  margin_calculator_margincalculation                         │
│  • Per record: 1-2 KB                                        │
│  • Typical: 100 calculations = 100-200 KB                    │
│                                                              │
│  historical_backtest_backtestcache                           │
│  • Per record: 2-4 MB                                        │
│  • Typical: 10 cached results = 20-40 MB                     │
│  • Auto-cleanup: Expires after 24 hours                      │
│                                                              │
│  historical_backtest_backtestsession                         │
│  • Per record: ~500 bytes                                    │
│  • Typical: 1000 sessions = 500 KB                           │
│                                                              │
│  historical_backtest_savedbacktestresult                     │
│  • Per record: 2-4 MB                                        │
│  • Maximum: 50 results/user × 100 users = 10-20 GB           │
│  • Typical: 10 results/user × 10 users = 200-400 MB          │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  TOTAL ESTIMATED DATABASE SIZE                               │
│  • Development (10 users): 500 MB - 1 GB                     │
│  • Production (100 users): 10 GB - 20 GB                     │
└──────────────────────────────────────────────────────────────┘
```

---

# 📊 PAGE 2: DATA FLOW DIAGRAMS

## Complete Data Flow: From User Action to Database Storage

### Flow 1: Margin Calculator - Save Calculation

```
┌─────────────────────────────────────────────────────────────────────────┐
│  USER PERFORMS MARGIN CALCULATION                                       │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 1: User Input (Browser)                                           │
│  ──────────────────────────────────────────────────────────────────     │
│  Form Fields:                                                            │
│  • Ticker: "SPY"                                                         │
│  • Investment: $100,000                                                  │
│  • Leverage: 2.0x                                                        │
│  • Account Type: "reg_t"                                                 │
│  • Position: "Long"                                                      │
│  • Interest Rate: 5.5%                                                   │
│  • Holding Period: 24 months                                             │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │ POST /margin-calculator/calculate/
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 2: Backend Processing (views.py)                                  │
│  ──────────────────────────────────────────────────────────────────     │
│  1. Validate inputs                                                      │
│  2. Fetch current price from FMP API                                     │
│  3. Calculate:                                                           │
│     • Cash required = $100,000 / 2.0 = $50,000                          │
│     • Margin loan = $50,000                                              │
│     • Shares = $100,000 / $price                                         │
│     • Margin call price = Calculate based on account type               │
│     • Interest costs = Calculate daily/monthly/annual                   │
│  4. Create results JSON                                                  │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 3: Save to Database                                               │
│  ──────────────────────────────────────────────────────────────────     │
│  MarginCalculation.objects.create(                                       │
│      user_id = 123,              # If logged in                          │
│      ticker = "SPY",                                                     │
│      investment_amount = 100000,                                         │
│      leverage = 2.0,                                                     │
│      current_price = 450.25,                                             │
│      account_type = "reg_t",                                             │
│      position_type = "Long",                                             │
│      interest_rate = 5.5,                                                │
│      holding_period = 24,                                                │
│      results = {                                                         │
│          "cash_required": 50000,                                         │
│          "margin_loan": 50000,                                           │
│          "shares": 222,                                                  │
│          "margin_call_price": 300.17,                                    │
│          "daily_interest": 7.53,                                         │
│          ... (50+ calculated fields)                                     │
│      }                                                                   │
│  )                                                                       │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  DATABASE: margin_calculator_margincalculation                           │
│  ──────────────────────────────────────────────────────────────────     │
│  New Row Inserted:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ID: 42                                                           │   │
│  │ User: 123                                                        │   │
│  │ Ticker: SPY                                                      │   │
│  │ Investment: $100,000                                             │   │
│  │ Leverage: 2.0x                                                   │   │
│  │ Results: {JSON with 50+ fields}                                  │   │
│  │ Created: 2025-10-28 14:30:00                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Size: ~1.5 KB                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### Flow 2: Historical Backtest - Run & Cache

```
┌─────────────────────────────────────────────────────────────────────────┐
│  USER RUNS HISTORICAL BACKTEST                                           │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 1: User Input                                                      │
│  ──────────────────────────────────────────────────────────────────     │
│  • ETF: SPY                                                              │
│  • Mode: Profit Threshold                                                │
│  • Dates: 2010-01-01 to 2024-10-28                                       │
│  • Leverage: 2.0x                                                        │
│  • Account: reg_t                                                        │
│  • Initial Investment: $100,000                                          │
│  • Profit Threshold: 100%                                                │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │ POST /historical-backtest/api/run/
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 2: Create Session Tracking                                        │
│  ──────────────────────────────────────────────────────────────────     │
│  BacktestSession.objects.create(                                         │
│      session_id = "uuid-12345...",                                       │
│      mode = "profit_threshold",                                          │
│      etf = "SPY",                                                        │
│      start_date = "2010-01-01",                                          │
│      end_date = "2024-10-28",                                            │
│      parameters = {all params},                                          │
│      status = "running"                                                  │
│  )                                                                       │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 3: Check Cache First                                              │
│  ──────────────────────────────────────────────────────────────────     │
│  cache_key = md5(etf + mode + dates + leverage + ...)                   │
│  cached = BacktestCache.get_cached_result(cache_key)                     │
│                                                                          │
│  IF cached AND not expired:                                              │
│      ↓ Return cached results (instant!)                                 │
│  ELSE:                                                                   │
│      ↓ Run backtest calculation                                         │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │ Cache MISS → Run backtest
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 4: Run Backtest Calculation (5-10 seconds)                        │
│  ──────────────────────────────────────────────────────────────────     │
│  1. Fetch market data (SPY prices 2010-2024)                             │
│  2. Simulate day-by-day trading:                                         │
│     • Calculate daily portfolio value                                   │
│     • Check for margin calls                                            │
│     • Detect liquidation events                                         │
│     • Apply profit threshold rebalancing                                │
│  3. Generate 15 Plotly charts                                            │
│  4. Calculate 500+ performance metrics                                   │
│  5. Create comprehensive results JSON (2-4 MB)                           │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 5: Save to Cache (for future requests)                            │
│  ──────────────────────────────────────────────────────────────────     │
│  BacktestCache.save_result(                                              │
│      cache_key = "md5-hash-abc123",                                      │
│      etf = "SPY",                                                        │
│      mode = "profit_threshold",                                          │
│      parameters = {all params},                                          │
│      results = {complete 2-4 MB JSON},                                   │
│      ttl_hours = 24                                                      │
│  )                                                                       │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 6: Update Session Status                                          │
│  ──────────────────────────────────────────────────────────────────     │
│  session.status = "completed"                                            │
│  session.completed_at = now()                                            │
│  session.execution_time_ms = 8543                                        │
│  session.save()                                                          │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  DATABASE STATE AFTER BACKTEST                                           │
│  ──────────────────────────────────────────────────────────────────     │
│                                                                          │
│  TABLE: historical_backtest_backtestcache                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ cache_key: "md5-hash-abc123"                                     │   │
│  │ etf: "SPY"                                                       │   │
│  │ mode: "profit_threshold"                                         │   │
│  │ results: {2.5 MB JSON with all charts, metrics, data}           │   │
│  │ expires_at: 2025-10-29 14:30:00 (24 hrs from now)               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  TABLE: historical_backtest_backtestsession                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ session_id: "uuid-12345..."                                      │   │
│  │ status: "completed"                                              │   │
│  │ execution_time_ms: 8543                                          │   │
│  │ completed_at: 2025-10-28 14:30:08                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### Flow 3: Save Backtest Results - Permanent Storage

```
┌─────────────────────────────────────────────────────────────────────────┐
│  USER SAVES BACKTEST RESULTS                                             │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 1: User Clicks "💾 SAVE RESULTS"                                   │
│  ──────────────────────────────────────────────────────────────────     │
│  Modal opens with form:                                                  │
│  • Name: "SPY 2x Conservative 2010-2024"                                 │
│  • Description: "Testing profit threshold mode"                          │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │ POST /historical-backtest/save-result/
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 2: Backend Validation                                             │
│  ──────────────────────────────────────────────────────────────────     │
│  1. Get user_id from session: 123                                        │
│  2. Check quota: user has 15/50 saved results ✓                         │
│  3. Validate name is not empty ✓                                        │
│  4. Extract parameters from results JSON                                 │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  STEP 3: Save to Database                                               │
│  ──────────────────────────────────────────────────────────────────     │
│  SavedBacktestResult.objects.create(                                     │
│      result_id = uuid.uuid4(),                                           │
│      user_id = 123,                                                      │
│      user_email = "user@example.com",                                    │
│      name = "SPY 2x Conservative 2010-2024",                             │
│      description = "Testing profit threshold mode",                      │
│      mode = "profit_threshold",                                          │
│      ticker = "SPY",                                                     │
│      start_date = "2010-01-01",                                          │
│      end_date = "2024-10-28",                                            │
│      leverage = 2.0,                                                     │
│      account_type = "reg_t",                                             │
│      initial_investment = 100000,                                        │
│      profit_threshold = 100.0,                                           │
│      results_data = {                                                    │
│          "status": "success",                                            │
│          "metrics": {500+ metrics},                                      │
│          "charts": {15 Plotly charts},                                   │
│          "dataframe": {3650 rows of daily data},                         │
│          ... ENTIRE 2.5 MB OUTPUT ...                                    │
│      }                                                                   │
│  )                                                                       │
└───────────────────────┬─────────────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  DATABASE: historical_backtest_savedbacktestresult                       │
│  ──────────────────────────────────────────────────────────────────     │
│  New Row Inserted:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ ID: 16                                                           │   │
│  │ result_id: "a4f3c2d1-8b9e-4f3a-9d2c-1e5f6a7b8c9d"               │   │
│  │ user_id: 123                                                     │   │
│  │ name: "SPY 2x Conservative 2010-2024"                            │   │
│  │ ticker: SPY                                                      │   │
│  │ mode: profit_threshold                                           │   │
│  │ results_data: {2.5 MB COMPLETE JSON}                             │   │
│  │ created_at: 2025-10-28 14:35:00                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  Size: 2.5 MB                                                            │
│  User now has: 16/50 saved results                                       │
└──────────────────────────────────────────────────────────────────────────┘
```

---

# 📊 PAGE 3: DATABASE RELATIONSHIPS & QUERIES

## Entity Relationship Diagram (ERD)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MARGIN APP DATABASE RELATIONSHIPS                     │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────┐
│  UMBRELLA APP (External)         │
│  umbrella_db.sqlite3             │
│  ┌────────────────────────────┐  │
│  │  User Table                │  │
│  │  ─────────────────────     │  │
│  │  • id (PK)                 │  │
│  │  • email                   │  │
│  │  • password                │  │
│  │  • is_active               │  │
│  └────────────────────────────┘  │
└───────────────┬──────────────────┘
                │
                │ Foreign Key (optional)
                │ Session data (user_id)
                ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  MARGIN APP DATABASE (db.sqlite3)                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  margin_calculator_margincalculation                           │    │
│  │  ──────────────────────────────────────────────────────────    │    │
│  │  • id (PK)                                                     │    │
│  │  • user_id (FK → umbrella.User, nullable)   ←─────────────────┼────┘
│  │  • ticker                                                      │
│  │  • investment_amount                                           │
│  │  • leverage                                                    │
│  │  • results (JSON)                                              │
│  │  • created_at                                                  │
│  └────────────────────────────────────────────────────────────────┘
│        ↑
│        │ INDEPENDENT TABLES (No FK relationships)
│        ↓
│  ┌────────────────────────────────────────────────────────────────┐
│  │  historical_backtest_backtestcache                             │
│  │  ──────────────────────────────────────────────────────────    │
│  │  • id (PK)                                                     │
│  │  • cache_key (UNIQUE)                                          │
│  │  • etf                                                         │
│  │  • mode                                                        │
│  │  • results (JSON - 2-4 MB)                                     │
│  │  • expires_at                                                  │
│  └────────────────────────────────────────────────────────────────┘
│        ↑
│        │ INDEPENDENT TABLES (No FK relationships)
│        ↓
│  ┌────────────────────────────────────────────────────────────────┐
│  │  historical_backtest_backtestsession                           │
│  │  ──────────────────────────────────────────────────────────    │
│  │  • id (PK)                                                     │
│  │  • session_id (UNIQUE)                                         │
│  │  • mode                                                        │
│  │  • etf                                                         │
│  │  • status                                                      │
│  │  • execution_time_ms                                           │
│  └────────────────────────────────────────────────────────────────┘
│        ↑
│        │ INDEPENDENT TABLES (No FK relationships)
│        ↓
│  ┌────────────────────────────────────────────────────────────────┐
│  │  historical_backtest_savedbacktestresult                       │
│  │  ──────────────────────────────────────────────────────────    │
│  │  • id (PK)                                                     │
│  │  • result_id (UNIQUE)                                          │
│  │  • user_id (INDEXED, from umbrella session) ←─────────────────┼────┐
│  │  • name                                                        │    │
│  │  • ticker                                                      │    │
│  │  • mode                                                        │    │
│  │  • results_data (JSON - 2-4 MB)                                │    │
│  │  • created_at                                                  │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
        │
        │ Note: user_id in SavedBacktestResult is stored as integer
        │ Not a true FK constraint, linked via session data
        └──────────────────────────────────────────────────────────────────┘
```

### Key Relationships:
- **NO direct foreign key constraints between tables**
- `margin_calculator_margincalculation.user_id` → Optional link to umbrella User
- `historical_backtest_savedbacktestresult.user_id` → Stored from session (no FK)
- All tables are **independent** for performance and flexibility

---

## Common Database Queries

### Query 1: Get User's Calculation History
```sql
SELECT id, ticker, investment_amount, leverage, created_at
FROM margin_calculator_margincalculation
WHERE user_id = 123
ORDER BY created_at DESC
LIMIT 10;
```

### Query 2: Check for Cached Backtest
```sql
SELECT results
FROM historical_backtest_backtestcache
WHERE cache_key = 'md5-hash-abc123'
  AND expires_at > CURRENT_TIMESTAMP;
```

### Query 3: Get User's Saved Backtest Results
```sql
SELECT result_id, name, ticker, mode, created_at
FROM historical_backtest_savedbacktestresult
WHERE user_id = 123
ORDER BY created_at DESC;
```

### Query 4: Get Backtest Performance Stats
```sql
SELECT 
    mode,
    COUNT(*) as total_runs,
    AVG(execution_time_ms) as avg_time,
    MAX(execution_time_ms) as max_time
FROM historical_backtest_backtestsession
WHERE status = 'completed'
  AND created_at > DATE('now', '-30 days')
GROUP BY mode;
```

### Query 5: Clean Up Expired Cache
```sql
DELETE FROM historical_backtest_backtestcache
WHERE expires_at < CURRENT_TIMESTAMP;
```

---

## Database Maintenance Operations

### Auto-Cleanup Process

```
┌─────────────────────────────────────────────────────────────────────┐
│  AUTOMATIC DATABASE MAINTENANCE                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Cache Expiration (BacktestCache)                                │
│     ────────────────────────────────────────────────────────        │
│     • Trigger: Before each cache lookup                             │
│     • Action: Delete rows where expires_at < now()                  │
│     • Frequency: On-demand                                          │
│     • Impact: Frees 2-4 MB per expired entry                        │
│                                                                      │
│  2. Quota Enforcement (SavedBacktestResult)                         │
│     ─────────────────────────────────────────────────────────────   │
│     • Trigger: Before saving new result                             │
│     • Action: Check COUNT(*) WHERE user_id = X                      │
│     • Limit: 50 results per user                                    │
│     • Response: Reject save if quota exceeded                       │
│                                                                      │
│  3. Session Cleanup (BacktestSession)                               │
│     ─────────────────────────────────────────────────────────────   │
│     • Trigger: Manual or cron job                                   │
│     • Action: Archive sessions > 90 days old                        │
│     • Frequency: Weekly                                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Migration to Production (PostgreSQL)

### Development vs Production Database

```
┌──────────────────────────────────────────────────────────────────────┐
│  DEVELOPMENT (Current)              PRODUCTION (AWS)                 │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  SQLite (db.sqlite3)                PostgreSQL RDS                   │
│  ────────────────────               ───────────────                  │
│  • File-based                       • Server-based                   │
│  • Single file                      • Distributed storage            │
│  • 5-50 MB typical                  • 10-100 GB scalable             │
│  • No concurrent writes             • 1000+ concurrent connections   │
│  • Manual backups                   • Automatic daily backups        │
│  • Local only                       • Network accessible             │
│  • Free                             • ~$15-30/month                  │
│                                                                       │
│  Migration Command:                                                  │
│  ──────────────────                                                  │
│  1. Update settings.py with PostgreSQL config                        │
│  2. pip install psycopg2-binary                                      │
│  3. python manage.py migrate                                         │
│  4. python manage.py dumpdata > data.json (optional)                 │
│  5. python manage.py loaddata data.json                              │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Summary Statistics

```
┌──────────────────────────────────────────────────────────────────────┐
│  MARGIN APP DATABASE - KEY METRICS                                   │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Total Databases:           1 (db.sqlite3)                           │
│  Total Tables:              4                                        │
│  Tables with Data:          4                                        │
│  Total Indexes:             11                                       │
│                                                                       │
│  Data Types Stored:                                                  │
│  • Margin calculations      ✓                                        │
│  • Backtest cache           ✓                                        │
│  • Session tracking         ✓                                        │
│  • Saved backtest results   ✓                                        │
│                                                                       │
│  Largest Data Storage:                                               │
│  • SavedBacktestResult: 2-4 MB per record                            │
│  • BacktestCache: 2-4 MB per record (temporary)                      │
│                                                                       │
│  Performance Features:                                               │
│  • 11 database indexes for fast queries                              │
│  • Cache system reduces database hits                                │
│  • Automatic cache expiration                                        │
│  • User-based data isolation                                         │
│                                                                       │
│  Data Retention:                                                     │
│  • Calculations: Permanent                                           │
│  • Cache: 24 hours                                                   │
│  • Sessions: Permanent (recommend 90-day cleanup)                    │
│  • Saved Results: Permanent (50 per user max)                        │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

**End of Document**

---

## Quick Reference Card

**Database File:** `margin_app/db.sqlite3`

**Tables:**
1. `margin_calculator_margincalculation` - Margin calculations
2. `historical_backtest_backtestcache` - Temporary cache (24hr TTL)
3. `historical_backtest_backtestsession` - Session tracking
4. `historical_backtest_savedbacktestresult` - User's saved results

**Largest Data:** SavedBacktestResult (2-4 MB per record)

**Total Quota:** 50 saved results per user

**Production Migration:** SQLite → PostgreSQL RDS on AWS

