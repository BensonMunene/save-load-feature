# 💾 Save & Load Backtest Results Feature
## Complete Technical Documentation

---

## 📋 Table of Contents
1. [Feature Overview](#feature-overview)
2. [Architecture & Data Flow](#architecture--data-flow)
3. [Database Schema](#database-schema)
4. [API Endpoints](#api-endpoints)
5. [User Interface Design](#user-interface-design)
6. [Implementation Flow](#implementation-flow)
7. [Security & Access Control](#security--access-control)
8. [Storage & Scalability](#storage--scalability)
9. [Testing Strategy](#testing-strategy)

---

## 1. Feature Overview

### Purpose
Enable users to save complete historical backtest results to their account and load them from any device, providing:
- **Cross-device accessibility** - Access results from any laptop
- **Historical record keeping** - Maintain analysis history
- **Quick reference** - Reload complex backtests instantly without re-running
- **Organization** - Tag and categorize saved results

### Key Capabilities
- ✅ Save complete backtest results (all charts, metrics, data)
- ✅ Load results with perfect reproduction
- ✅ Browse saved results library
- ✅ Search, filter, and organize
- ✅ Edit metadata (name, description, tags)
- ✅ Delete unwanted results
- ✅ Quota management (50 results per user)

---

## 2. Architecture & Data Flow

### System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER'S BROWSER                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Historical Backtest UI (index.html)                      │  │
│  │  - Run backtest forms                                     │  │
│  │  - Results display area                                   │  │
│  │  - Save/Load buttons                                      │  │
│  │  - Modals (Save Modal, Load Modal)                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↕                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  JavaScript (historical_backtest_v2.js)                   │  │
│  │  - saveBacktestResults()                                  │  │
│  │  - loadSavedResult()                                      │  │
│  │  - deleteSavedResult()                                    │  │
│  │  - displaySavedResults()                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            ↕ HTTP/JSON
┌─────────────────────────────────────────────────────────────────┐
│                    UMBRELLA APP (Authentication)                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Proxy Layer (margin_integration/views.py)                │  │
│  │  - Authenticates user                                     │  │
│  │  - Injects session data:                                  │  │
│  │    • umbrella_user_id                                     │  │
│  │    • umbrella_user_email                                  │  │
│  │    • umbrella_user_name                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                    MARGIN APP (Backend)                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Django Views (historical_backtest/views.py)              │  │
│  │  - save_backtest_result()                                 │  │
│  │  - list_saved_results()                                   │  │
│  │  - load_saved_result()                                    │  │
│  │  - delete_saved_result()                                  │  │
│  │  - update_saved_result()                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ↕                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Django ORM (models.py)                                   │  │
│  │  - SavedBacktestResult model                              │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE (SQLite/PostgreSQL)                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  saved_backtest_result table                              │  │
│  │  - User's saved results                                   │  │
│  │  - Indexed by user_id for fast queries                    │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Complete User Flow: Save Results

```
┌─────────────┐
│ User runs   │
│ backtest    │
└──────┬──────┘
       │
       ↓
┌─────────────────────────────────┐
│ Backtest completes successfully │
│ Results displayed on screen     │
└──────┬──────────────────────────┘
       │
       ↓
┌──────────────────────────┐
│ "💾 SAVE RESULTS" button │
│ appears                  │
└──────┬───────────────────┘
       │
       ↓ User clicks
┌──────────────────────────┐
│ Save Modal opens         │
│ - Name input (required)  │
│ - Description (optional) │
│ - Tags (optional)        │
└──────┬───────────────────┘
       │
       ↓ User fills form & submits
┌─────────────────────────────────────┐
│ JavaScript validation               │
│ - Check name is not empty           │
│ - Check currentBacktestResults exist│
└──────┬──────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────┐
│ POST /historical-backtest/save-result/│
│ Body: {                              │
│   name: "...",                       │
│   description: "...",                │
│   tags: [...],                       │
│   results: { entire JSON }           │
│ }                                    │
└──────┬──────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────┐
│ Backend: save_backtest_result()     │
│ 1. Get user_id from session         │
│ 2. Validate inputs                  │
│ 3. Check quota (< 50 results)       │
│ 4. Extract parameters from results  │
│ 5. Create SavedBacktestResult       │
└──────┬──────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────┐
│ Database INSERT                     │
│ New row in saved_backtest_result    │
└──────┬──────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────┐
│ Response: 200 OK                    │
│ { status: "success",                │
│   result_id: "abc123...",           │
│   name: "..." }                     │
└──────┬──────────────────────────────┘
       │
       ↓
┌─────────────────────────────────────┐
│ JavaScript shows success message    │
│ "✓ Backtest saved successfully!"    │
│ Modal closes                        │
└─────────────────────────────────────┘
```

### Complete User Flow: Load Results

```
┌─────────────────┐
│ User clicks     │
│ "📂 LOAD SAVED  │
│    RESULTS"     │
└────────┬────────┘
         │
         ↓
┌────────────────────────────────────┐
│ Load Modal opens                   │
│ Shows "Loading..."                 │
└────────┬───────────────────────────┘
         │
         ↓
┌────────────────────────────────────┐
│ GET /historical-backtest/          │
│     list-saved-results/            │
└────────┬───────────────────────────┘
         │
         ↓
┌────────────────────────────────────┐
│ Backend: list_saved_results()      │
│ 1. Get user_id from session        │
│ 2. Query: WHERE user_id = X        │
│ 3. Return list of results metadata │
└────────┬───────────────────────────┘
         │
         ↓
┌────────────────────────────────────┐
│ Response: List of saved results    │
│ [                                  │
│   {result_id, name, ticker, ...},  │
│   {result_id, name, ticker, ...},  │
│   ...                              │
│ ]                                  │
└────────┬───────────────────────────┘
         │
         ↓
┌────────────────────────────────────┐
│ JavaScript: displaySavedResults()  │
│ Renders cards for each result      │
│ - Name, mode badge                 │
│ - Parameters (ticker, dates, etc)  │
│ - Tags                             │
│ - LOAD and DELETE buttons          │
└────────┬───────────────────────────┘
         │
         ↓ User clicks LOAD button
┌────────────────────────────────────┐
│ GET /historical-backtest/          │
│     load-saved-result/?            │
│     result_id=abc123...            │
└────────┬───────────────────────────┘
         │
         ↓
┌────────────────────────────────────┐
│ Backend: load_saved_result()       │
│ 1. Get user_id from session        │
│ 2. Query: WHERE result_id = X      │
│            AND user_id = Y          │
│ 3. Return results_data JSON        │
└────────┬───────────────────────────┘
         │
         ↓
┌────────────────────────────────────┐
│ Response: Complete results data    │
│ {                                  │
│   status: "success",               │
│   result: { ... entire JSON ... }, │
│   metadata: { name, saved_at, ...} │
│ }                                  │
└────────┬───────────────────────────┘
         │
         ↓
┌────────────────────────────────────┐
│ JavaScript: loadSavedResult()      │
│ 1. Set currentBacktestResults      │
│ 2. Call displayBacktestResults()   │
│ 3. Close modal                     │
│ 4. Show success message            │
└────────────────────────────────────┘
         │
         ↓
┌────────────────────────────────────┐
│ Results render on screen           │
│ - All charts appear                │
│ - All metrics display              │
│ - All tables populate              │
│ EXACTLY as if backtest just ran    │
└────────────────────────────────────┘
```

---

## 3. Database Schema

### SavedBacktestResult Model

```sql
CREATE TABLE saved_backtest_result (
    -- Primary Key
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    result_id VARCHAR(100) UNIQUE NOT NULL,  -- UUID for external reference
    
    -- User Identification (from Umbrella App)
    user_id INTEGER NOT NULL,                -- Links to umbrella_app User
    user_email VARCHAR(254) NOT NULL,        -- Email for reference
    
    -- User-Provided Metadata
    name VARCHAR(255) NOT NULL,              -- e.g., "SPY 2x Conservative 2010-2024"
    description TEXT,                        -- Optional notes
    tags JSON,                               -- ["SPY", "conservative", "long-term"]
    
    -- Backtest Parameters (for quick filtering/display)
    mode VARCHAR(50) NOT NULL,               -- liquidation_reentry, fresh_capital, profit_threshold
    ticker VARCHAR(10) NOT NULL,             -- SPY, VTI, etc.
    start_date DATE NOT NULL,                -- 2010-01-01
    end_date DATE NOT NULL,                  -- 2024-10-26
    leverage FLOAT NOT NULL,                 -- 2.0
    account_type VARCHAR(20) NOT NULL,       -- reg_t, portfolio
    initial_investment FLOAT NOT NULL,       -- 100000
    profit_threshold FLOAT,                  -- 100.0 (for profit_threshold mode)
    
    -- Complete Results Data (ENTIRE JSON response)
    results_data JSON NOT NULL,              -- { status, metrics, charts, ... }
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Indexes for performance
    INDEX idx_user_created (user_id, created_at),
    INDEX idx_user_ticker (user_id, ticker),
    INDEX idx_user_mode (user_id, mode)
);
```

### Sample Database Row

```json
{
  "id": 1,
  "result_id": "a4f3c2d1-8b9e-4f3a-9d2c-1e5f6a7b8c9d",
  "user_id": 123,
  "user_email": "john.doe@example.com",
  "name": "SPY 2x Leverage Conservative Strategy 2010-2024",
  "description": "Testing profit threshold mode with 100% rebalancing threshold. Good performance during bull markets.",
  "tags": ["SPY", "conservative", "long-term", "tested"],
  "mode": "profit_threshold",
  "ticker": "SPY",
  "start_date": "2010-01-01",
  "end_date": "2024-10-26",
  "leverage": 2.0,
  "account_type": "reg_t",
  "initial_investment": 100000,
  "profit_threshold": 100.0,
  "results_data": {
    "status": "success",
    "mode": "profit_threshold",
    "ticker": "SPY",
    "metrics": { /* all performance metrics */ },
    "charts": { /* all Plotly charts */ },
    "dashboard": { /* dashboard data */ },
    "cushion_analysis": { /* margin cushion */ },
    "round_analysis": { /* round-by-round */ },
    "dataframe": { /* daily data */ }
  },
  "created_at": "2024-10-26 14:30:00",
  "updated_at": "2024-10-26 14:30:00"
}
```

---

## 4. API Endpoints

### 1. Save Backtest Result

**Endpoint:** `POST /historical-backtest/save-result/`

**Request:**
```json
{
  "name": "SPY 2x Conservative 2010-2024",
  "description": "Testing profit threshold mode",
  "tags": ["SPY", "conservative", "long-term"],
  "results": {
    /* Entire backtest results object from run_backtest */
  }
}
```

**Response (Success):**
```json
{
  "status": "success",
  "message": "Backtest saved successfully",
  "result_id": "a4f3c2d1-8b9e-4f3a-9d2c-1e5f6a7b8c9d",
  "name": "SPY 2x Conservative 2010-2024"
}
```

**Response (Error - Quota Exceeded):**
```json
{
  "error": "Maximum limit of 50 saved results reached. Please delete some old results first.",
  "status": 400
}
```

### 2. List Saved Results

**Endpoint:** `GET /historical-backtest/list-saved-results/`

**Response:**
```json
{
  "status": "success",
  "total_count": 15,
  "results": [
    {
      "result_id": "a4f3c2d1...",
      "name": "SPY 2x Conservative 2010-2024",
      "description": "Testing profit threshold mode",
      "tags": ["SPY", "conservative"],
      "mode": "profit_threshold",
      "ticker": "SPY",
      "start_date": "2010-01-01",
      "end_date": "2024-10-26",
      "leverage": 2.0,
      "account_type": "reg_t",
      "created_at": "2024-10-26T14:30:00Z",
      "updated_at": "2024-10-26T14:30:00Z"
    },
    /* ... more results ... */
  ]
}
```

### 3. Load Saved Result

**Endpoint:** `GET /historical-backtest/load-saved-result/?result_id={id}`

**Response:**
```json
{
  "status": "success",
  "result": {
    /* Complete backtest results data */
    "status": "success",
    "mode": "profit_threshold",
    "metrics": { /* ... */ },
    "charts": { /* ... */ },
    /* ... everything ... */
  },
  "metadata": {
    "name": "SPY 2x Conservative 2010-2024",
    "description": "Testing profit threshold mode",
    "tags": ["SPY", "conservative"],
    "saved_at": "2024-10-26T14:30:00Z"
  }
}
```

### 4. Delete Saved Result

**Endpoint:** `DELETE /historical-backtest/delete-saved-result/`

**Request:**
```json
{
  "result_id": "a4f3c2d1-8b9e-4f3a-9d2c-1e5f6a7b8c9d"
}
```

**Response:**
```json
{
  "status": "success",
  "message": "Result deleted successfully"
}
```

### 5. Update Saved Result Metadata

**Endpoint:** `PUT /historical-backtest/update-saved-result/`

**Request:**
```json
{
  "result_id": "a4f3c2d1...",
  "name": "SPY 2x Updated Name",
  "description": "Updated description",
  "tags": ["SPY", "updated", "test"]
}
```

**Response:**
```json
{
  "status": "success",
  "message": "Result updated successfully"
}
```

---

## 5. User Interface Design

### Save Modal UI

```
┌─────────────────────────────────────────────────────────┐
│  💾 SAVE BACKTEST RESULTS                        [X]    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  NAME (REQUIRED)                                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │ SPY 2x Leverage 2010-2024                          │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  DESCRIPTION (OPTIONAL)                                  │
│  ┌────────────────────────────────────────────────────┐ │
│  │ Testing profit threshold mode with 100%            │ │
│  │ rebalancing. Performance looks good during         │ │
│  │ bull markets but need to test bear scenarios.      │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
│  TAGS (OPTIONAL)                                         │
│  ┌────────────────────────────────────────────────────┐ │
│  │ SPY, conservative, long-term, tested               │ │
│  └────────────────────────────────────────────────────┘ │
│  Comma-separated tags for organization                  │
│                                                          │
│                                      [CANCEL]  [SAVE]   │
└─────────────────────────────────────────────────────────┘
```

### Load Modal UI (With Results)

```
┌──────────────────────────────────────────────────────────────────────┐
│  📂 LOAD SAVED RESULTS                                        [X]    │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ SPY 2x Conservative 2010-2024        [PROFIT_THRESHOLD]       │ │
│  │                                                                │ │
│  │ 📊 SPY  📅 2010-01-01 to 2024-10-26  ⚡ 2.0x  💼 REG-T        │ │
│  │                                                                │ │
│  │ Testing profit threshold mode with 100% rebalancing.          │ │
│  │                                                                │ │
│  │ [SPY] [conservative] [long-term] [tested]                     │ │
│  │                                                                │ │
│  │ Saved: 10/26/2024                                             │ │
│  │                                                                │ │
│  │                                           [LOAD]  [DELETE]    │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ VTI 4x Aggressive Portfolio Margin   [LIQUIDATION_REENTRY]   │ │
│  │                                                                │ │
│  │ 📊 VTI  📅 2015-01-01 to 2024-10-26  ⚡ 4.0x  💼 PORTFOLIO    │ │
│  │                                                                │ │
│  │ High risk strategy testing. Multiple liquidations occurred.   │ │
│  │                                                                │ │
│  │ [VTI] [aggressive] [portfolio-margin]                         │ │
│  │                                                                │ │
│  │ Saved: 10/25/2024                                             │ │
│  │                                                                │ │
│  │                                           [LOAD]  [DELETE]    │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │ SPY 3x Fresh Capital Restart         [FRESH_CAPITAL]          │ │
│  │                                                                │ │
│  │ 📊 SPY  📅 2008-01-01 to 2024-10-26  ⚡ 3.0x  💼 PORTFOLIO    │ │
│  │                                                                │ │
│  │ [SPY] [moderate] [multi-round]                                │ │
│  │                                                                │ │
│  │ Saved: 10/20/2024                                             │ │
│  │                                                                │ │
│  │                                           [LOAD]  [DELETE]    │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│                                                         [CLOSE]       │
└──────────────────────────────────────────────────────────────────────┘
```

### Results Page with Save/Load Buttons

```
┌──────────────────────────────────────────────────────────────┐
│ BACKTEST RESULTS                                             │
│                                                              │
│ PROFIT THRESHOLD BACKTEST COMPLETE: ANALYZED 3,650 TRADING  │
│ DAYS WITH 5 LIQUIDATION EVENTS                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ [💾 SAVE RESULTS]  [📂 LOAD SAVED RESULTS]                  │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ 🔶 PERFORMANCE DASHBOARD                                     │
│                                                              │
│ ┌─────────────┬─────────────┬─────────────┬─────────────┐  │
│ │ TOTAL RETURN│    CAGR     │ FINAL EQUITY│MAX DRAWDOWN │  │
│ │   456.8%    │   12.5%     │  $556,800   │  -23.4%     │  │
│ │  CAGR 12.5% │             │             │             │  │
│ └─────────────┴─────────────┴─────────────┴─────────────┘  │
│                                                              │
│ ... (rest of results) ...                                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Implementation Flow

### Phase 1: Backend Setup

1. **Create Database Model** (10 min)
   - Add `SavedBacktestResult` to `models.py`
   - Define fields, indexes, meta

2. **Create Migrations** (5 min)
   - Run `makemigrations`
   - Run `migrate`

3. **Add API Views** (30 min)
   - `save_backtest_result()`
   - `list_saved_results()`
   - `load_saved_result()`
   - `delete_saved_result()`
   - `update_saved_result()`

4. **Add URL Routes** (5 min)
   - Map endpoints to views

### Phase 2: Frontend UI

5. **HTML Modal Structure** (20 min)
   - Save modal with form
   - Load modal with results list

6. **CSS Styling** (15 min)
   - Modal styles
   - Result card styles
   - Buttons and animations

7. **JavaScript Functions** (45 min)
   - Modal open/close handlers
   - Save form submission
   - Load results list
   - Display saved results
   - Delete confirmation
   - API fetch calls

### Phase 3: Integration

8. **Connect Save Button** (10 min)
   - Show button after backtest completes
   - Wire up click handler

9. **Connect Load Button** (10 min)
   - Always visible
   - Opens load modal

### Phase 4: Testing

10. **End-to-End Testing** (30 min)
    - Save a result
    - Load it back
    - Verify all data appears
    - Delete result
    - Test quota limits

**Total Estimated Time:** ~3 hours

---

## 7. Security & Access Control

### Authentication Flow

```
User Request
    ↓
Umbrella App Proxy (margin_integration)
    ↓
Check: Is user authenticated?
    ├─ NO → Redirect to login
    └─ YES → Continue
        ↓
    Inject session data:
    - umbrella_user_id
    - umbrella_user_email
        ↓
Margin App receives request
    ↓
Extract user_id from session
    ↓
Query: WHERE user_id = X
    ↓
Return ONLY user's data
```

### Security Measures

1. **User Isolation**
   - All queries filtered by `user_id`
   - User A cannot access User B's results
   - Enforced at database query level

2. **Session Validation**
   ```python
   user_id = request.session.get('umbrella_user_id')
   if not user_id:
       return JsonResponse({'error': 'Authentication required'}, status=401)
   ```

3. **Data Ownership Verification**
   ```python
   # Load query ensures ownership
   saved_result = SavedBacktestResult.objects.get(
       result_id=result_id,
       user_id=user_id  # Must match logged-in user
   )
   ```

4. **Quota Enforcement**
   ```python
   # Prevent abuse
   MAX_SAVED_RESULTS_PER_USER = 50
   count = SavedBacktestResult.objects.filter(user_id=user_id).count()
   if count >= MAX_SAVED_RESULTS_PER_USER:
       return JsonResponse({'error': 'Quota exceeded'}, status=400)
   ```

---

## 8. AWS Deployment Architecture

### Current Local Development Setup

```
┌────────────────────────────────────────────────────────┐
│              YOUR LAPTOP (Development)                  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐ │
│  │  Umbrella App                                     │ │
│  │  - Port: 8000                                     │ │
│  │  - Database: umbrella_db.sqlite3                  │ │
│  │  - Handles: Authentication, User Management       │ │
│  └──────────────────────────────────────────────────┘ │
│                         ↓ Proxy                        │
│  ┌──────────────────────────────────────────────────┐ │
│  │  Margin App                                       │ │
│  │  - Port: 8001                                     │ │
│  │  - Database: db.sqlite3                           │ │
│  │  - Handles: Backtest calculations, saved results  │ │
│  └──────────────────────────────────────────────────┘ │
│                                                         │
│  ⚠️ Problem: Data lost if laptop crashes/resets       │
│  ⚠️ Problem: Only accessible from this laptop          │
└────────────────────────────────────────────────────────┘
```

### AWS Production Deployment (Recommended)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AWS CLOUD INFRASTRUCTURE                          │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                    VPC (Virtual Private Cloud)                     │   │
│  │                                                                     │   │
│  │  ┌──────────────────────────────────────────────────────────────┐ │   │
│  │  │  EC2 Instance 1 (Umbrella App)                               │ │   │
│  │  │  ┌────────────────────────────────────────────────────┐      │ │   │
│  │  │  │ Docker Container                                   │      │ │   │
│  │  │  │ - Django App (umbrella_app)                        │      │ │   │
│  │  │  │ - Gunicorn (WSGI server)                           │      │ │   │
│  │  │  │ - Port: 8000                                       │      │ │   │
│  │  │  └────────────────────────────────────────────────────┘      │ │   │
│  │  └───────────────────────────┬──────────────────────────────────┘ │   │
│  │                              │                                     │   │
│  │                              │ Session data                        │   │
│  │                              ↓ (user_id, email)                   │   │
│  │  ┌──────────────────────────────────────────────────────────────┐ │   │
│  │  │  EC2 Instance 2 (Margin App)                                 │ │   │
│  │  │  ┌────────────────────────────────────────────────────┐      │ │   │
│  │  │  │ Docker Container                                   │      │ │   │
│  │  │  │ - Django App (margin_app)                          │      │ │   │
│  │  │  │ - Gunicorn (WSGI server)                           │      │ │   │
│  │  │  │ - Port: 8001                                       │      │ │   │
│  │  │  │ - Historical Backtest Module                       │      │ │   │
│  │  │  │ - Save/Load Results Feature ⭐                     │      │ │   │
│  │  │  └────────────────────────────────────────────────────┘      │ │   │
│  │  └───────────────────────────┬──────────────────────────────────┘ │   │
│  │                              │                                     │   │
│  │                              │ Database Queries                    │   │
│  │                              ↓                                     │   │
│  │  ┌──────────────────────────────────────────────────────────────┐ │   │
│  │  │  RDS PostgreSQL Instance                                     │ │   │
│  │  │  ┌────────────────────────────────────────────────────┐      │ │   │
│  │  │  │ Database: margin_production_db                     │      │ │   │
│  │  │  │                                                     │      │ │   │
│  │  │  │ Tables:                                            │      │ │   │
│  │  │  │ ├─ saved_backtest_result ⭐ NEW                   │      │ │   │
│  │  │  │ ├─ backtest_session                                │      │ │   │
│  │  │  │ ├─ backtest_cache                                  │      │ │   │
│  │  │  │ └─ (other tables...)                               │      │ │   │
│  │  │  │                                                     │      │ │   │
│  │  │  │ Features:                                          │      │ │   │
│  │  │  │ ✓ Automatic backups (daily)                        │      │ │   │
│  │  │  │ ✓ Multi-AZ deployment (high availability)          │      │ │   │
│  │  │  │ ✓ Encryption at rest                               │      │ │   │
│  │  │  │ ✓ Point-in-time recovery                           │      │ │   │
│  │  │  └────────────────────────────────────────────────────┘      │ │   │
│  │  └──────────────────────────────────────────────────────────────┘ │   │
│  │                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                   ↑                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │  Application Load Balancer (ALB)                                     │ │
│  │  - HTTPS (port 443)                                                  │ │
│  │  - SSL/TLS Certificate                                               │ │
│  │  - Routes traffic to EC2 instances                                   │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                   ↑                                         │
└───────────────────────────────────┼─────────────────────────────────────────┘
                                    │
                    INTERNET        │
                                    │
┌───────────────────────────────────┼─────────────────────────────────────────┐
│                                   ↓                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │  Route 53 (DNS)                                                      │ │
│  │  - Domain: margin-analytics.yourdomain.com                           │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│                        USER'S LAPTOP / DEVICE                               │
│                        (Any laptop, anywhere)                               │
│                                                                             │
│  1. User logs in from Laptop A (New York)                                  │
│  2. Runs backtest, saves results to RDS PostgreSQL                         │
│  3. User travels to different city                                         │
│  4. Logs in from Laptop B (California)                                     │
│  5. Loads saved results - Data retrieved from RDS                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### AWS Services Used

#### 1. **EC2 Instances** (Virtual Servers)

```
┌─────────────────────────────────────┐
│  EC2 Instance Type: t3.medium       │
│  - 2 vCPUs                          │
│  - 4 GB RAM                         │
│  - 30 GB SSD Storage                │
│  - Cost: ~$30/month                 │
│                                     │
│  Running:                           │
│  - Ubuntu 22.04 LTS                 │
│  - Docker                           │
│  - Django App                       │
│  - Gunicorn                         │
└─────────────────────────────────────┘
```

#### 2. **RDS PostgreSQL** (Database)

```
┌─────────────────────────────────────┐
│  RDS Instance: db.t3.micro          │
│  - PostgreSQL 15                    │
│  - 2 vCPUs                          │
│  - 1 GB RAM                         │
│  - 20 GB SSD Storage                │
│  - Cost: ~$15-20/month              │
│                                     │
│  Features:                          │
│  ✓ Automatic backups (7-day)        │
│  ✓ Automatic updates                │
│  ✓ Multi-AZ (optional, +$15/mo)     │
│  ✓ Encrypted storage                │
│                                     │
│  Connection String:                 │
│  postgresql://user:pass@            │
│    db-instance.region.rds.aws.com:  │
│    5432/margin_db                   │
└─────────────────────────────────────┘
```

#### 3. **S3 Bucket** (Static Files - Optional)

```
┌─────────────────────────────────────┐
│  S3 Bucket: margin-app-static       │
│  - CSS, JavaScript, Images          │
│  - Cost: ~$1-3/month                │
│  - CloudFront CDN (optional)        │
└─────────────────────────────────────┘
```

### Data Flow: User Saves Backtest Results

```
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 1: User Runs Backtest (Laptop in New York)                    │
└──────────────────────────────────────────────────────────────────────┘
                            ↓
User in browser: https://margin-analytics.yourdomain.com
                            ↓
                    ┌───────────────┐
                    │  Route 53     │
                    │  (DNS)        │
                    └───────┬───────┘
                            ↓
                    ┌───────────────┐
                    │  ALB          │
                    │  (HTTPS)      │
                    └───────┬───────┘
                            ↓
         ┌──────────────────┴──────────────────┐
         ↓                                     ↓
┌────────────────┐                  ┌────────────────┐
│ EC2: Umbrella  │                  │ EC2: Margin    │
│ Port 8000      │ ─── Proxy ────→  │ Port 8001      │
│                │                  │                │
│ Authenticates  │                  │ Runs Backtest  │
│ user           │                  │                │
│                │                  │ User clicks    │
│ Sets session:  │                  │ "Save Results" │
│ - user_id=123  │                  │                │
│ - email=...    │                  │                │
└────────────────┘                  └────────┬───────┘
                                             ↓
                                    POST /save-result/
                                    {
                                      name: "...",
                                      results: {...}
                                    }
                                             ↓
                                    ┌────────────────┐
                                    │  Django View   │
                                    │  (views.py)    │
                                    │                │
                                    │ 1. Get user_id │
                                    │    from session│
                                    │ 2. Validate    │
                                    │ 3. Create row  │
                                    └────────┬───────┘
                                             ↓
                                    INSERT INTO saved_backtest_result
                                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│  RDS PostgreSQL (Persistent Storage on AWS)                         │
│                                                                      │
│  saved_backtest_result table:                                       │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ result_id | user_id | name              | results_data      │  │
│  ├──────────────────────────────────────────────────────────────┤  │
│  │ abc-123   | 123     | "SPY 2x 2010-24"  | {...3MB JSON...}  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ✓ Data is now PERSISTENT                                           │
│  ✓ Survives EC2 restarts                                            │
│  ✓ Automatic backups                                                │
│  ✓ Accessible from anywhere                                         │
└─────────────────────────────────────────────────────────────────────┘
                                             ↓
                                    Response: Success
                                             ↓
                            User sees: "✓ Saved!"


┌──────────────────────────────────────────────────────────────────────┐
│  STEP 2: User Loads Results (Different Laptop in California)        │
└──────────────────────────────────────────────────────────────────────┘
                            ↓
User on new laptop: https://margin-analytics.yourdomain.com
                            ↓
                    Logs in with same account
                    (user_id = 123)
                            ↓
                    Clicks "Load Saved Results"
                            ↓
         ┌──────────────────┴──────────────────┐
         ↓                                     ↓
┌────────────────┐                  ┌────────────────┐
│ EC2: Umbrella  │                  │ EC2: Margin    │
│                │ ─── Proxy ────→  │                │
│ Authenticates  │                  │ GET /list-     │
│ Same user      │                  │  saved-results/│
│ user_id=123    │                  │                │
└────────────────┘                  └────────┬───────┘
                                             ↓
                                    SELECT * FROM
                                    saved_backtest_result
                                    WHERE user_id = 123
                                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│  RDS PostgreSQL                                                      │
│                                                                      │
│  Returns all saved results for user_id=123:                         │
│  [                                                                   │
│    {result_id: "abc-123", name: "SPY 2x 2010-24", ...},            │
│    {result_id: "def-456", name: "VTI 4x Aggressive", ...},         │
│    ...                                                              │
│  ]                                                                  │
└─────────────────────────────────────────────────────────────────────┘
                                             ↓
                            User selects result to load
                                             ↓
                                    GET /load-saved-result/
                                    ?result_id=abc-123
                                             ↓
                                    SELECT results_data
                                    FROM saved_backtest_result
                                    WHERE result_id='abc-123'
                                      AND user_id=123
                                             ↓
┌─────────────────────────────────────────────────────────────────────┐
│  RDS PostgreSQL                                                      │
│                                                                      │
│  Returns complete results JSON (3 MB):                              │
│  {                                                                   │
│    metrics: {...},                                                  │
│    charts: {...},                                                   │
│    dataframe: {...},                                                │
│    ... everything ...                                               │
│  }                                                                  │
└─────────────────────────────────────────────────────────────────────┘
                                             ↓
                            Results render on screen
                            - All charts appear
                            - All metrics display
                            - Exact same as saved
                            
                    ✓ Cross-device access works!
```

### Cost Breakdown (Monthly)

```
┌────────────────────────────────────────────────────┐
│  AWS Monthly Costs (Estimated)                     │
├────────────────────────────────────────────────────┤
│  EC2 t3.medium (2 instances)       $30 × 2 = $60  │
│  RDS db.t3.micro (PostgreSQL)              = $18  │
│  Application Load Balancer                 = $16  │
│  Data Transfer (est.)                      = $5   │
│  S3 Storage (static files)                 = $2   │
│  Route 53 (DNS)                            = $1   │
├────────────────────────────────────────────────────┤
│  TOTAL:                                    ~$102/mo│
└────────────────────────────────────────────────────┘

Cost Optimization Options:
- Use t3.small instead of t3.medium: Save $20/mo
- Single EC2 (combined apps): Save $30/mo
- Reserved Instances (1-year): Save 30-40%

Minimal Setup: ~$50-60/month
```

### Database Migration: SQLite → PostgreSQL

```bash
# Step 1: On AWS, create RDS PostgreSQL instance
# Via AWS Console or Terraform

# Step 2: Update Django settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', 'margin_db'),
        'USER': os.environ.get('DB_USER', 'postgres'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST', 'your-rds-endpoint.rds.amazonaws.com'),
        'PORT': os.environ.get('DB_PORT', '5432'),
    }
}

# Step 3: Install PostgreSQL adapter
pip install psycopg2-binary

# Step 4: Run migrations on PostgreSQL
python manage.py migrate

# Step 5: (Optional) Migrate existing data
python manage.py dumpdata > data.json  # From SQLite
python manage.py loaddata data.json    # Into PostgreSQL
```

### Security Best Practices

```
1. Database Security
   ✓ RDS in private subnet (not internet-accessible)
   ✓ Security groups allow only EC2 → RDS
   ✓ SSL/TLS for database connections
   ✓ Encryption at rest enabled
   ✓ Automatic backups enabled

2. Application Security
   ✓ HTTPS only (SSL certificate via AWS ACM)
   ✓ Session data encrypted
   ✓ User isolation enforced by user_id
   ✓ CSRF protection enabled
   ✓ SQL injection prevention (Django ORM)

3. Network Security
   ✓ VPC with public/private subnets
   ✓ Security groups (firewall rules)
   ✓ No direct database access from internet
   ✓ Bastion host for admin access (optional)
```

### Complete Database Data Flow Diagram

#### Development Environment (SQLite)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      DEVELOPMENT (Your Laptop)                          │
│                                                                         │
│  USER SAVES BACKTEST RESULT:                                           │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  Browser (Chrome/Firefox)                                              │
│       │                                                                 │
│       │ Click "Save Results"                                           │
│       ↓                                                                 │
│  JavaScript (historical_backtest_v2.js)                                │
│       │                                                                 │
│       │ saveBacktestResults()                                          │
│       │ - Collect: name, description, tags                             │
│       │ - Package: currentBacktestResults (full JSON)                  │
│       ↓                                                                 │
│  POST /historical-backtest/save-result/                                │
│  ┌──────────────────────────────────────────────┐                      │
│  │ Request Body:                                │                      │
│  │ {                                            │                      │
│  │   name: "SPY 2x Conservative",               │                      │
│  │   description: "Testing...",                 │                      │
│  │   tags: ["SPY", "conservative"],             │                      │
│  │   results: {                                 │                      │
│  │     status: "success",                       │                      │
│  │     mode: "profit_threshold",                │                      │
│  │     ticker: "SPY",                           │                      │
│  │     metrics: {...3000 lines...},             │                      │
│  │     charts: {...Plotly JSON...},             │                      │
│  │     dataframe: {...15 years data...}         │                      │
│  │   }                                          │                      │
│  │ }                                            │                      │
│  └──────────────────────────────────────────────┘                      │
│       │                                                                 │
│       ↓                                                                 │
│  Django View (views.py)                                                │
│  ┌──────────────────────────────────────────────┐                      │
│  │ save_backtest_result():                      │                      │
│  │ 1. Get user_id from session (e.g., 123)      │                      │
│  │ 2. Validate inputs                           │                      │
│  │ 3. Extract parameters from results           │                      │
│  │ 4. Call Django ORM:                          │                      │
│  │    SavedBacktestResult.objects.create(...)   │                      │
│  └──────────────────────────────────────────────┘                      │
│       │                                                                 │
│       ↓                                                                 │
│  Django ORM (models.py)                                                │
│  ┌──────────────────────────────────────────────┐                      │
│  │ Translates to SQL:                           │                      │
│  │                                              │                      │
│  │ INSERT INTO saved_backtest_result (          │                      │
│  │   result_id,                                 │                      │
│  │   user_id,                                   │                      │
│  │   user_email,                                │                      │
│  │   name,                                      │                      │
│  │   description,                               │                      │
│  │   tags,                                      │                      │
│  │   mode,                                      │                      │
│  │   ticker,                                    │                      │
│  │   start_date,                                │                      │
│  │   end_date,                                  │                      │
│  │   leverage,                                  │                      │
│  │   account_type,                              │                      │
│  │   initial_investment,                        │                      │
│  │   profit_threshold,                          │                      │
│  │   results_data,                              │                      │
│  │   created_at,                                │                      │
│  │   updated_at                                 │                      │
│  │ ) VALUES (                                   │                      │
│  │   'a4f3c2d1-8b9e...',                        │                      │
│  │   123,                                       │                      │
│  │   'user@example.com',                        │                      │
│  │   'SPY 2x Conservative',                     │                      │
│  │   'Testing...',                              │                      │
│  │   '["SPY", "conservative"]',                 │                      │
│  │   'profit_threshold',                        │                      │
│  │   'SPY',                                     │                      │
│  │   '2010-01-01',                              │                      │
│  │   '2024-10-26',                              │                      │
│  │   2.0,                                       │                      │
│  │   'reg_t',                                   │                      │
│  │   100000,                                    │                      │
│  │   100.0,                                     │                      │
│  │   '{...ENTIRE 3MB JSON...}',                 │                      │
│  │   '2024-10-26 14:30:00',                     │                      │
│  │   '2024-10-26 14:30:00'                      │                      │
│  │ );                                           │                      │
│  └──────────────────────────────────────────────┘                      │
│       │                                                                 │
│       ↓                                                                 │
│  SQLite Database (db.sqlite3)                                          │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │  FILE: D:\...\margin_app\db.sqlite3                             │  │
│  │  SIZE: Growing with each save (~2-4 MB per result)              │  │
│  │                                                                  │  │
│  │  TABLE: saved_backtest_result                                   │  │
│  │  ┌────┬──────┬───────┬─────────────┬─────────────────────────┐ │  │
│  │  │ ID │UID │Ticker │    Name      │   results_data (JSON)   │ │  │
│  │  ├────┼──────┼───────┼─────────────┼─────────────────────────┤ │  │
│  │  │ 1  │ 123  │ SPY   │SPY 2x Cons. │{status:"success",...}   │ │  │
│  │  │ 2  │ 123  │ VTI   │VTI 4x Agg.  │{status:"success",...}   │ │  │
│  │  │ 3  │ 456  │ SPY   │SPY Test     │{status:"success",...}   │ │  │
│  │  └────┴──────┴───────┴─────────────┴─────────────────────────┘ │  │
│  │                                                                  │  │
│  │  ⚠️  LIMITATION: File stored on laptop disk                     │  │
│  │  ⚠️  Lost if laptop crashes or disk fails                       │  │
│  │  ⚠️  Not accessible from other devices                          │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│       │                                                                 │
│       │ Write confirmed                                                │
│       ↓                                                                 │
│  Django ORM returns Python object                                      │
│       │                                                                 │
│       ↓                                                                 │
│  Django View creates JSON response                                     │
│  ┌──────────────────────────────────────────────┐                      │
│  │ {                                            │                      │
│  │   status: "success",                         │                      │
│  │   message: "Backtest saved successfully",    │                      │
│  │   result_id: "a4f3c2d1-8b9e...",             │                      │
│  │   name: "SPY 2x Conservative"                │                      │
│  │ }                                            │                      │
│  └──────────────────────────────────────────────┘                      │
│       │                                                                 │
│       ↓                                                                 │
│  JavaScript receives response                                          │
│       │                                                                 │
│       ↓                                                                 │
│  Browser shows: "✓ Backtest saved successfully!"                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Production Environment (PostgreSQL on AWS RDS)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PRODUCTION (AWS Cloud)                               │
│                                                                         │
│  USER SAVES BACKTEST RESULT FROM ANY DEVICE:                           │
│  ═══════════════════════════════════════════════════════════════════   │
│                                                                         │
│  User's Laptop/Phone/Tablet (New York)                                 │
│  Browser: https://margin-analytics.yourdomain.com                      │
│       │                                                                 │
│       │ Click "Save Results"                                           │
│       ↓                                                                 │
│  ┌────────────────────────────────────────────┐                        │
│  │ HTTPS Request (encrypted)                  │                        │
│  │ POST /historical-backtest/save-result/     │                        │
│  │ Headers: Cookie (session_id)               │                        │
│  │ Body: { name, results: {...3MB JSON...} }  │                        │
│  └────────────────────────────────────────────┘                        │
│       │                                                                 │
│       ↓ Through Internet                                               │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      AWS REGION (us-east-1)                     │   │
│  │                                                                 │   │
│  │  Route 53 (DNS)                                                 │   │
│  │  margin-analytics.yourdomain.com → ALB IP                       │   │
│  │       ↓                                                          │   │
│  │                                                                 │   │
│  │  Application Load Balancer (ALB)                                │   │
│  │  - Terminates SSL/TLS                                           │   │
│  │  - Routes to healthy EC2 instance                               │   │
│  │       ↓                                                          │   │
│  │                                                                 │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │ EC2 Instance: Margin App (172.31.10.5)                   │  │   │
│  │  │                                                           │  │   │
│  │  │ Django Application (Port 8001)                            │  │   │
│  │  │  ↓                                                        │  │   │
│  │  │ views.save_backtest_result()                              │  │   │
│  │  │  - Extract user_id from session: 123                      │  │   │
│  │  │  - Validate data                                          │  │   │
│  │  │  - Prepare database insert                                │  │   │
│  │  │  ↓                                                        │  │   │
│  │  │ Django ORM:                                               │  │   │
│  │  │ SavedBacktestResult.objects.create(                       │  │   │
│  │  │   user_id=123,                                            │  │   │
│  │  │   name="SPY 2x Conservative",                             │  │   │
│  │  │   results_data={...3MB JSON...}                           │  │   │
│  │  │ )                                                         │  │   │
│  │  │  ↓                                                        │  │   │
│  │  │ SQL Generator (psycopg2 adapter)                          │  │   │
│  │  │  ↓                                                        │  │   │
│  │  │ ┌─────────────────────────────────────────────────────┐  │  │   │
│  │  │ │ SQL Query (sent to PostgreSQL):                     │  │  │   │
│  │  │ │                                                     │  │  │   │
│  │  │ │ INSERT INTO saved_backtest_result                   │  │  │   │
│  │  │ │ (result_id, user_id, name, ticker, mode,            │  │  │   │
│  │  │ │  start_date, end_date, leverage, results_data,      │  │  │   │
│  │  │ │  created_at, updated_at)                            │  │  │   │
│  │  │ │ VALUES                                              │  │  │   │
│  │  │ │ ('a4f3...', 123, 'SPY 2x Conservative', 'SPY',      │  │  │   │
│  │  │ │  'profit_threshold', '2010-01-01', '2024-10-26',    │  │  │   │
│  │  │ │  2.0, '{"status":"success",...3MB JSON...}',        │  │  │   │
│  │  │ │  NOW(), NOW())                                      │  │  │   │
│  │  │ │ RETURNING id;                                       │  │  │   │
│  │  │ └─────────────────────────────────────────────────────┘  │  │   │
│  │  │                                                           │  │   │
│  │  └───────────────────────┬───────────────────────────────────┘  │   │
│  │                          │                                       │   │
│  │                          │ Database Connection                   │   │
│  │                          │ (SSL/TLS encrypted)                   │   │
│  │                          ↓                                       │   │
│  │                                                                 │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │ RDS PostgreSQL Instance (Private Subnet)                 │  │   │
│  │  │                                                           │  │   │
│  │  │ Endpoint: margin-db.xxx.us-east-1.rds.amazonaws.com      │  │   │
│  │  │ Port: 5432                                               │  │   │
│  │  │ Database: margin_production_db                           │  │   │
│  │  │                                                           │  │   │
│  │  │ PostgreSQL Engine                                        │  │   │
│  │  │  ↓                                                       │  │   │
│  │  │ Query Planner                                            │  │   │
│  │  │  - Parse SQL                                             │  │   │
│  │  │  - Validate constraints                                  │  │   │
│  │  │  - Check user_id index                                   │  │   │
│  │  │  ↓                                                       │  │   │
│  │  │ Execution Engine                                         │  │   │
│  │  │  - Lock table row                                        │  │   │
│  │  │  - Insert data into table                                │  │   │
│  │  │  - Update indexes                                        │  │   │
│  │  │  ↓                                                       │  │   │
│  │  │ Storage Layer                                            │  │   │
│  │  │  ┌────────────────────────────────────────────────────┐ │  │   │
│  │  │  │ Physical Storage (EBS Volume)                      │ │  │   │
│  │  │  │                                                    │ │  │   │
│  │  │  │ saved_backtest_result table                       │ │  │   │
│  │  │  │ ┌──────────────────────────────────────────────┐  │ │  │   │
│  │  │  │ │ Data Pages (8KB blocks):                     │  │ │  │   │
│  │  │  │ │                                              │  │ │  │   │
│  │  │  │ │ Page 1: Row ID=1, user_id=123, ...           │  │ │  │   │
│  │  │  │ │ Page 2: Row ID=2, user_id=123, ...           │  │ │  │   │
│  │  │  │ │ Page 3: Row ID=3, user_id=456, ...           │  │ │  │   │
│  │  │  │ │ Page 4: NEW → Row ID=4, user_id=123 ← WRITE │  │ │  │   │
│  │  │  │ │          name="SPY 2x Conservative"          │  │ │  │   │
│  │  │  │ │          results_data={...3MB JSONB...}      │  │ │  │   │
│  │  │  │ └──────────────────────────────────────────────┘  │ │  │   │
│  │  │  │                                                    │ │  │   │
│  │  │  │ Indexes:                                           │ │  │   │
│  │  │  │ ┌──────────────────────────────────────────────┐  │ │  │   │
│  │  │  │ │ idx_user_created:                            │  │ │  │   │
│  │  │  │ │ (user_id=123, created_at) → Page 4 ← UPDATE │  │ │  │   │
│  │  │  │ │ (user_id=123, created_at) → Page 2          │  │ │  │   │
│  │  │  │ │ (user_id=123, created_at) → Page 1          │  │ │  │   │
│  │  │  │ └──────────────────────────────────────────────┘  │ │  │   │
│  │  │  │                                                    │ │  │   │
│  │  │  │ ✓ Data encrypted at rest (AES-256)                │ │  │   │
│  │  │  │ ✓ Automatic backup to S3 (daily snapshot)         │ │  │   │
│  │  │  │ ✓ Transaction log for point-in-time recovery      │ │  │   │
│  │  │  └────────────────────────────────────────────────────┘ │  │   │
│  │  │  │                                                       │  │   │
│  │  │  │ Write Confirmed                                      │  │   │
│  │  │  ↓                                                       │  │   │
│  │  │ Return: id=4, result_id='a4f3...'                       │  │   │
│  │  │                                                           │  │   │
│  │  └───────────────────────┬───────────────────────────────────┘  │   │
│  │                          │                                       │   │
│  │                          │ Response                              │   │
│  │                          ↓                                       │   │
│  │  ┌──────────────────────────────────────────────────────────┐  │   │
│  │  │ EC2: Django ORM receives confirmation                    │  │   │
│  │  │  ↓                                                        │  │   │
│  │  │ Django View: Create success response                     │  │   │
│  │  │ {                                                        │  │   │
│  │  │   status: "success",                                     │  │   │
│  │  │   result_id: "a4f3...",                                  │  │   │
│  │  │   message: "Backtest saved successfully"                 │  │   │
│  │  │ }                                                        │  │   │
│  │  └──────────────────────────────────────────────────────────┘  │   │
│  │       │                                                          │   │
│  │       ↓                                                          │   │
│  │  Application Load Balancer                                      │   │
│  │       │                                                          │   │
│  └───────┼──────────────────────────────────────────────────────────┘   │
│          │                                                               │
│          ↓ Through Internet (HTTPS)                                     │
│                                                                         │
│  User's Browser (New York)                                             │
│  Shows: "✓ Backtest saved successfully!"                               │
│                                                                         │
│  ═════════════════════════════════════════════════════════════════     │
│                                                                         │
│  USER LOADS BACKTEST FROM DIFFERENT DEVICE:                            │
│  ═════════════════════════════════════════════════════════════════     │
│                                                                         │
│  User's Different Laptop (California, 3 days later)                    │
│  Browser: https://margin-analytics.yourdomain.com                      │
│       │                                                                 │
│       │ User logs in (same account, user_id=123)                       │
│       │ Clicks "Load Saved Results"                                    │
│       ↓                                                                 │
│  ┌────────────────────────────────────────────┐                        │
│  │ HTTPS Request                              │                        │
│  │ GET /historical-backtest/list-saved-results/│                       │
│  │ Cookie: session_id (logged in as user 123)  │                       │
│  └────────────────────────────────────────────┘                        │
│       ↓                                                                 │
│  Route 53 → ALB → EC2 (same path as save)                              │
│       ↓                                                                 │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ EC2: Django View                                             │      │
│  │ list_saved_results():                                        │      │
│  │  - Get user_id from session: 123                             │      │
│  │  - Query database                                            │      │
│  │                                                               │      │
│  │ Django ORM:                                                  │      │
│  │ SavedBacktestResult.objects.filter(user_id=123)              │      │
│  │  ↓                                                           │      │
│  │ ┌─────────────────────────────────────────────────────────┐ │      │
│  │ │ SQL Query:                                              │ │      │
│  │ │                                                         │ │      │
│  │ │ SELECT id, result_id, name, ticker, mode,               │ │      │
│  │ │        start_date, end_date, leverage,                  │ │      │
│  │ │        created_at, updated_at                           │ │      │
│  │ │ FROM saved_backtest_result                              │ │      │
│  │ │ WHERE user_id = 123                                     │ │      │
│  │ │ ORDER BY created_at DESC;                               │ │      │
│  │ └─────────────────────────────────────────────────────────┘ │      │
│  └───────────────────────┬──────────────────────────────────────┘      │
│                          │                                             │
│                          ↓                                             │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ RDS PostgreSQL                                               │      │
│  │                                                               │      │
│  │ Query Optimizer:                                             │      │
│  │  - Use idx_user_created index (FAST!)                        │      │
│  │  - Sequential scan not needed                                │      │
│  │  ↓                                                           │      │
│  │ Index Scan:                                                  │      │
│  │  - Lookup user_id=123 in index                               │      │
│  │  - Found pointers to: Page 4, Page 2, Page 1                │      │
│  │  ↓                                                           │      │
│  │ Fetch Data Pages:                                            │      │
│  │  - Read Page 4: Row ID=4, "SPY 2x Conservative", ...         │      │
│  │  - Read Page 2: Row ID=2, "VTI 4x Aggressive", ...          │      │
│  │  - Read Page 1: Row ID=1, "SPY Test", ...                    │      │
│  │  ↓                                                           │      │
│  │ Return Result Set:                                           │      │
│  │ [                                                            │      │
│  │   {id: 4, result_id: "a4f3...", name: "SPY 2x Cons.", ...},  │      │
│  │   {id: 2, result_id: "def4...", name: "VTI 4x Agg.", ...},  │      │
│  │   {id: 1, result_id: "xyz7...", name: "SPY Test", ...}       │      │
│  │ ]                                                            │      │
│  └───────────────────────┬──────────────────────────────────────┘      │
│                          │                                             │
│                          ↓                                             │
│  EC2: Django formats response                                          │
│  Browser receives list of saved results                                │
│       ↓                                                                 │
│  User sees their saved results:                                        │
│  - SPY 2x Conservative (saved 3 days ago in NY)                        │
│  - VTI 4x Aggressive                                                   │
│  - SPY Test                                                            │
│       ↓                                                                 │
│  User clicks "LOAD" on "SPY 2x Conservative"                           │
│       ↓                                                                 │
│  ┌────────────────────────────────────────────┐                        │
│  │ GET /historical-backtest/load-saved-result/ │                       │
│  │     ?result_id=a4f3...                      │                       │
│  └────────────────────────────────────────────┘                        │
│       ↓                                                                 │
│  Route 53 → ALB → EC2                                                  │
│       ↓                                                                 │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ EC2: Django View                                             │      │
│  │ load_saved_result():                                         │      │
│  │  - user_id from session: 123                                 │      │
│  │  - result_id from params: "a4f3..."                          │      │
│  │                                                               │      │
│  │ Django ORM:                                                  │      │
│  │ SavedBacktestResult.objects.get(                             │      │
│  │   result_id="a4f3...",                                       │      │
│  │   user_id=123  ← Security: only user's own data             │      │
│  │ )                                                            │      │
│  │  ↓                                                           │      │
│  │ ┌─────────────────────────────────────────────────────────┐ │      │
│  │ │ SQL Query:                                              │ │      │
│  │ │                                                         │ │      │
│  │ │ SELECT *                                                │ │      │
│  │ │ FROM saved_backtest_result                              │ │      │
│  │ │ WHERE result_id = 'a4f3...'                             │ │      │
│  │ │   AND user_id = 123;                                    │ │      │
│  │ └─────────────────────────────────────────────────────────┘ │      │
│  └───────────────────────┬──────────────────────────────────────┘      │
│                          │                                             │
│                          ↓                                             │
│  ┌──────────────────────────────────────────────────────────────┐      │
│  │ RDS PostgreSQL                                               │      │
│  │                                                               │      │
│  │ Index Scan: result_id (unique) → Page 4                      │      │
│  │ Verify: user_id=123 ✓                                        │      │
│  │  ↓                                                           │      │
│  │ Fetch Complete Row including results_data:                   │      │
│  │ ┌─────────────────────────────────────────────────────────┐ │      │
│  │ │ results_data (JSONB, 3MB):                              │ │      │
│  │ │ {                                                       │ │      │
│  │ │   status: "success",                                    │ │      │
│  │ │   mode: "profit_threshold",                             │ │      │
│  │ │   ticker: "SPY",                                        │ │      │
│  │ │   metrics: {                                            │ │      │
│  │ │     "Total Return": 456.8,                              │ │      │
│  │ │     "CAGR": 12.5,                                       │ │      │
│  │ │     ...500 metrics...                                   │ │      │
│  │ │   },                                                    │ │      │
│  │ │   charts: {                                             │ │      │
│  │ │     portfolio_value: {                                  │ │      │
│  │ │       data: [...Plotly JSON...],                        │ │      │
│  │ │       layout: {...}                                     │ │      │
│  │ │     },                                                  │ │      │
│  │ │     ...15 charts...                                     │ │      │
│  │ │   },                                                    │ │      │
│  │ │   dataframe: {                                          │ │      │
│  │ │     columns: ["Date", "Portfolio_Value", ...],          │ │      │
│  │ │     data: [[...3650 rows of daily data...]]             │ │      │
│  │ │   }                                                     │ │      │
│  │ │ }                                                       │ │      │
│  │ └─────────────────────────────────────────────────────────┘ │      │
│  │                                                               │      │
│  │ Return: Complete 3MB JSON to Django                          │      │
│  └───────────────────────┬──────────────────────────────────────┘      │
│                          │                                             │
│                          ↓                                             │
│  EC2: Django sends complete results to browser                         │
│       ↓                                                                 │
│  Browser receives 3MB JSON                                             │
│       ↓                                                                 │
│  JavaScript: displayBacktestResults()                                  │
│  - Renders all Plotly charts                                           │
│  - Displays all metrics                                                │
│  - Populates all tables                                                │
│       ↓                                                                 │
│  User sees EXACT SAME results as saved 3 days ago in NY!               │
│  ✓ All charts interactive                                              │
│  ✓ All metrics displayed                                               │
│  ✓ All data tables complete                                            │
│  ✓ Cross-device access WORKS!                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Database Comparison: SQLite vs PostgreSQL

```
┌──────────────────────────────────────────────────────────────────────┐
│                    FEATURE COMPARISON                                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  SQLite (Development)           PostgreSQL RDS (Production)          │
│  ═══════════════════            ══════════════════════              │
│                                                                      │
│  Storage Location:              Storage Location:                   │
│  ├─ Local file on laptop        ├─ AWS EBS volume (SSD)             │
│  ├─ D:\...\db.sqlite3           ├─ Separate from app server         │
│  └─ Lost if disk fails          └─ Persistent, redundant            │
│                                                                      │
│  Concurrency:                   Concurrency:                        │
│  ├─ Single writer               ├─ Multiple concurrent writers      │
│  ├─ File-level locking          ├─ Row-level locking                │
│  └─ 1 request at a time         └─ Thousands of concurrent requests │
│                                                                      │
│  Backups:                       Backups:                            │
│  ├─ Manual only                 ├─ Automatic daily snapshots        │
│  ├─ Risk of data loss           ├─ Point-in-time recovery           │
│  └─ No disaster recovery        └─ Multi-AZ replication (optional)  │
│                                                                      │
│  Performance:                   Performance:                        │
│  ├─ Fast for small datasets     ├─ Optimized for large datasets     │
│  ├─ Slow with many users        ├─ Scales with user count           │
│  └─ Limited indexing            └─ Advanced indexing (GiST, GIN)    │
│                                                                      │
│  JSON Support:                  JSON Support:                       │
│  ├─ JSON stored as text         ├─ Native JSONB type                │
│  ├─ No JSON indexing            ├─ JSON indexing supported          │
│  └─ Slower JSON queries         └─ Fast JSON path queries           │
│                                                                      │
│  Security:                      Security:                           │
│  ├─ File permissions only       ├─ Network isolation (VPC)          │
│  ├─ No encryption at rest       ├─ Encryption at rest (AES-256)     │
│  └─ No SSL/TLS                  └─ SSL/TLS connections              │
│                                                                      │
│  Cost:                          Cost:                               │
│  ├─ Free                        ├─ ~$15-20/month                     │
│  └─ No infrastructure cost      └─ AWS RDS + data transfer           │
│                                                                      │
│  Best For:                      Best For:                           │
│  ├─ Development                 ├─ Production                        │
│  ├─ Testing                     ├─ Multiple users                    │
│  ├─ Single user                 ├─ High availability                 │
│  └─ Prototyping                 └─ Data integrity critical           │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 9. Storage & Scalability

### Storage Estimates

**Per Backtest Result:**
- Small (1 year): 200-500 KB
- Medium (5 years): 800 KB - 1.5 MB
- Large (15 years): 2-4 MB

**Per User (50 results max):**
- Average: 50 MB - 100 MB
- Maximum: 200 MB

**For 100 Users:**
- Average: 5 GB - 10 GB
- Maximum: 20 GB

### Database Recommendations

**Development (Current):**
- SQLite: `margin_app/db.sqlite3`
- Works well for < 10 users
- Simple, no setup required

**Production (AWS Deployment):**
- PostgreSQL RDS
- Persistent, reliable
- Automatic backups
- Scalable to thousands of users
- Cost: ~$15-30/month

### Migration Path

```python
# settings.py - Switch to PostgreSQL
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST'),  # RDS endpoint
        'PORT': '5432',
    }
}
```

---

## 9. Testing Strategy

### Unit Tests

```python
# Test save functionality
def test_save_backtest_result():
    # Setup mock user session
    # Call save endpoint
    # Verify database row created
    # Check all fields saved correctly

# Test load functionality
def test_load_saved_result():
    # Create test result in database
    # Call load endpoint
    # Verify correct data returned
    # Verify user isolation

# Test quota enforcement
def test_quota_limit():
    # Create 50 results for user
    # Attempt to save 51st
    # Verify error returned
```

### Integration Tests

1. **Save → Load → Verify**
   - Run full backtest
   - Save results
   - Clear UI
   - Load saved results
   - Verify all charts render
   - Verify all metrics match

2. **Multi-Device Simulation**
   - Save on "Device A"
   - Clear browser cache
   - Log in as same user
   - Load results
   - Verify accessible

3. **User Isolation Test**
   - User A saves result
   - Log out
   - Log in as User B
   - Verify User B cannot see User A's results

### Manual Testing Checklist

- [ ] Save backtest with all fields
- [ ] Save backtest with only name
- [ ] Load saved result - verify complete
- [ ] Delete saved result
- [ ] Try to save 51st result (quota test)
- [ ] Try to load non-existent result_id
- [ ] Try to access another user's result_id
- [ ] Edit result metadata
- [ ] Filter/search saved results
- [ ] Test on different laptop (same account)

---

## 📌 Quick Reference

### File Checklist
- `models.py` - Add SavedBacktestResult model
- `views.py` - Add 5 new API endpoints
- `urls.py` - Add 5 new URL routes
- `index.html` - Add save/load buttons + modals
- `historical_backtest_v2.js` - Add save/load functions
- `historical_backtest.css` - Add modal styles

### Key Commands
```bash
# Create migration
python manage.py makemigrations historical_backtest

# Apply migration
python manage.py migrate historical_backtest

# Check saved results in DB
python manage.py dbshell
> SELECT COUNT(*) FROM saved_backtest_result GROUP BY user_id;
```

### API Quick Test
```bash
# List saved results
curl http://localhost:8001/historical-backtest/list-saved-results/

# Load specific result
curl http://localhost:8001/historical-backtest/load-saved-result/?result_id=abc123
```

---

**Documentation Version:** 1.0  
**Last Updated:** October 27, 2024  
**Author:** Technical Documentation Team  
**Feature Status:** Ready for Implementation

