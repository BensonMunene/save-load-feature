# 🗄️ Complete Database Architecture Report - All Applications
## Umbrella Hub Project - Database Inventory & Design

**Report Date:** October 28, 2025  
**Total Applications:** 4  
**Total Databases:** 4 SQLite Databases  
**Production Status:** ⚠️ NOT production-ready (all using SQLite)

---

# 📊 DATABASE OVERVIEW - ALL APPLICATIONS

## Executive Summary

The Umbrella Hub project contains **4 separate Django applications**, each with its own **independent SQLite database**. Total database count: **4 databases** with **17 tables** across all applications.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    UMBRELLA HUB PROJECT DATABASES                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. UMBRELLA APP        → umbrella_db.sqlite3      (Primary database)   │
│  2. MARGIN APP          → db.sqlite3               (Margin analytics)   │
│  3. RETURN APP          → db.sqlite3               (ETF returns)        │
│  4. CREDIT ANALYZER APP → db.sqlite3               (Credit analysis)    │
│                                                                          │
│  Total Storage: ~500 MB - 1 GB (development)                            │
│  Production Risk: HIGH - All using SQLite (file-based)                  │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# 🏢 APPLICATION 1: UMBRELLA APP

## Database Information
- **Database Name:** `umbrella_db.sqlite3`
- **Location:** `umbrella_app/umbrella_db.sqlite3`
- **Purpose:** Central authentication & user management hub
- **Status:** ✅ Active with custom models

## Database Tables (9 Total)

### Custom Application Tables (5 Tables)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE 1: authentication_user                                           │
│  Purpose: User accounts & authentication                                │
├─────────────────────────────────────────────────────────────────────────┤
│  Fields:                                                                 │
│  • id (PK)                     - Primary key                             │
│  • username                    - Login username                          │
│  • email (UNIQUE)              - User email (used for login)            │
│  • password                    - Hashed password (PBKDF2)               │
│  • first_name                  - User's first name                      │
│  • last_name                   - User's last name                       │
│  • is_active                   - Account active status                  │
│  • is_staff                    - Admin access flag                      │
│  • is_superuser                - Superuser flag                         │
│  • is_approved                 - Admin approval status                  │
│  • approval_date               - When approved                          │
│  • approved_by (FK)            - Admin who approved                     │
│  • rejection_reason            - Why rejected (if applicable)           │
│  • last_login                  - Last login timestamp                   │
│  • last_login_ip               - IP address of last login               │
│  • login_count                 - Total login attempts                   │
│  • date_joined                 - Account creation date                  │
│  • created_at                  - Timestamp                              │
│  • updated_at                  - Last modified                          │
│                                                                          │
│  Size per record: ~500 bytes                                            │
│  Critical Data: YES - All user accounts                                 │
│  Production Risk: CRITICAL - User accounts lost on instance termination │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE 2: authentication_invitetoken                                    │
│  Purpose: User invitation system                                        │
├─────────────────────────────────────────────────────────────────────────┤
│  Fields:                                                                 │
│  • id (PK)                     - Primary key                             │
│  • email                       - Invited user's email                   │
│  • token (UNIQUE)              - 64-char invitation token               │
│  • invite_type                 - analyst, manager, admin, viewer        │
│  • custom_message              - Custom invite message                  │
│  • created_at                  - When created                           │
│  • expires_at                  - Expiration timestamp                   │
│  • used_at                     - When used (nullable)                   │
│  • created_by (FK)             - Admin who created invite               │
│  • email_sent_at               - When email was sent                    │
│  • email_bounced               - Email delivery status                  │
│  • bounce_reason               - Bounce error message                   │
│                                                                          │
│  Size per record: ~300 bytes                                            │
│  Critical Data: MEDIUM - Invitation tracking                            │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE 3: authentication_usersession                                    │
│  Purpose: Active user session tracking                                  │
├─────────────────────────────────────────────────────────────────────────┤
│  Fields:                                                                 │
│  • id (PK)                     - Primary key                             │
│  • user (FK)                   - User ID                                │
│  • session_key (UNIQUE)        - Django session key                     │
│  • ip_address                  - Client IP address                      │
│  • user_agent                  - Browser/device info                    │
│  • created_at                  - Session start time                     │
│  • last_activity               - Last activity timestamp                │
│  • is_active                   - Session active flag                    │
│                                                                          │
│  Size per record: ~400 bytes                                            │
│  Critical Data: LOW - Session info (temporary)                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE 4: authentication_loginattempt                                   │
│  Purpose: Security audit log for login attempts                         │
├─────────────────────────────────────────────────────────────────────────┤
│  Fields:                                                                 │
│  • id (PK)                     - Primary key                             │
│  • email                       - Login email attempted                  │
│  • ip_address                  - Source IP address                      │
│  • user_agent                  - Browser info                           │
│  • success                     - True/False                             │
│  • failure_reason              - Why login failed                       │
│  • attempted_at                - Timestamp                              │
│  • user (FK)                   - User if successful (nullable)          │
│                                                                          │
│  Size per record: ~350 bytes                                            │
│  Critical Data: HIGH - Security audit trail                             │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE 5: authentication_invitetemplate                                 │
│  Purpose: Pre-built email invitation templates                          │
├─────────────────────────────────────────────────────────────────────────┤
│  Fields:                                                                 │
│  • id (PK)                     - Primary key                             │
│  • name                        - Template name                          │
│  • invite_type                 - analyst, manager, admin, viewer        │
│  • subject                     - Email subject line                     │
│  • message_template            - Email body template                    │
│  • is_active                   - Active flag                            │
│  • created_at                  - Creation timestamp                     │
│  • created_by (FK)             - Admin who created                      │
│                                                                          │
│  Size per record: ~1 KB                                                 │
│  Critical Data: LOW - Email templates (can be recreated)                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Django System Tables (4 Tables)
- `django_session` - Session storage
- `django_admin_log` - Admin action log
- `auth_permission` - Django permissions
- `auth_group` - User groups

### Total Tables: 9 (5 custom + 4 Django system)

### Estimated Size: 
- Small deployment (10 users): **5-10 MB**
- Medium deployment (100 users): **50-100 MB**

### Critical Data:
- 🔴 **User accounts** (authentication_user) - CRITICAL
- 🔴 **Login audit logs** (authentication_loginattempt) - HIGH
- 🟡 **Active sessions** (authentication_usersession) - MEDIUM
- 🟢 **Invite tokens** - LOW
- 🟢 **Invite templates** - LOW

---

# 💰 APPLICATION 2: MARGIN APP

## Database Information
- **Database Name:** `db.sqlite3`
- **Location:** `margin_app/db.sqlite3`
- **Purpose:** Margin trading analytics & backtesting
- **Status:** ✅ Active with custom models

## Database Tables (8 Total)

### Custom Application Tables (4 Tables)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE 1: margin_calculator_margincalculation                           │
│  Purpose: Margin calculation history                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  Fields:                                                                 │
│  • id (PK)                     - Primary key                             │
│  • user_id (FK)                - User from umbrella_app (nullable)      │
│  • ticker                      - Stock symbol (e.g., SPY)               │
│  • investment_amount           - Dollar amount                          │
│  • leverage                    - 1.0x - 7.0x                            │
│  • current_price               - Stock price                            │
│  • account_type                - reg_t or portfolio                     │
│  • position_type               - Long or Short                          │
│  • interest_rate               - Annual % (1-10%)                       │
│  • holding_period              - Months (1-60)                          │
│  • results (JSON)              - All calculation results                │
│  • created_at, updated_at      - Timestamps                             │
│                                                                          │
│  Size per record: ~1-2 KB                                               │
│  Critical Data: LOW - Calculation history (can be recalculated)         │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE 2: historical_backtest_backtestcache                             │
│  Purpose: Temporary performance cache                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  Fields:                                                                 │
│  • id (PK)                     - Primary key                             │
│  • cache_key (UNIQUE)          - MD5 hash of parameters                 │
│  • etf                         - SPY or VTI                             │
│  • mode                        - Backtest mode                          │
│  • parameters (JSON)           - Input params                           │
│  • results (JSON)              - Complete results (2-4 MB)              │
│  • created_at                  - Cache timestamp                        │
│  • expires_at                  - Expiry (24 hours)                      │
│                                                                          │
│  TTL: 24 hours (auto-deleted)                                           │
│  Size per record: ~2-4 MB                                               │
│  Critical Data: NONE - Temporary cache only                             │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE 3: historical_backtest_backtestsession                           │
│  Purpose: Backtest execution tracking                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  Fields:                                                                 │
│  • id (PK)                     - Primary key                             │
│  • session_id (UNIQUE)         - UUID                                   │
│  • mode                        - liquidation_reentry, etc.              │
│  • etf                         - SPY or VTI                             │
│  • start_date, end_date        - Backtest period                        │
│  • parameters (JSON)           - All parameters                         │
│  • status                      - pending, running, completed, error     │
│  • created_at                  - Start time                             │
│  • completed_at                - End time                               │
│  • execution_time_ms           - Processing duration                    │
│                                                                          │
│  Size per record: ~500 bytes                                            │
│  Critical Data: LOW - Analytics only                                    │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│  TABLE 4: historical_backtest_savedbacktestresult                       │
│  Purpose: User's permanently saved backtest results                     │
├─────────────────────────────────────────────────────────────────────────┤
│  Fields:                                                                 │
│  • id (PK)                     - Primary key                             │
│  • result_id (UNIQUE)          - UUID                                   │
│  • user_id (INDEXED)           - User from umbrella_app                 │
│  • user_email                  - User email                             │
│  • name                        - User-friendly name                     │
│  • description                 - Optional notes                         │
│  • mode                        - Backtest mode                          │
│  • ticker                      - ETF symbol                             │
│  • start_date, end_date        - Backtest period                        │
│  • leverage                    - Leverage used                          │
│  • account_type                - reg_t or portfolio                     │
│  • initial_investment          - Starting equity                        │
│  • profit_threshold            - Rebalance % (nullable)                 │
│  • results_data (JSON)         - COMPLETE results (2-4 MB)              │
│  • created_at, updated_at      - Timestamps                             │
│                                                                          │
│  Quota: 50 results per user                                             │
│  Size per record: ~2-4 MB                                               │
│  Critical Data: CRITICAL - User's saved analysis work                   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Django System Tables (4 Tables)
- `django_session` - Session storage
- `django_admin_log` - Admin actions
- `django_migrations` - Migration history
- `auth_permission` - Permissions

### Total Tables: 8 (4 custom + 4 Django system)
### Estimated Size: 100 MB - 5 GB (depends on saved results)

---

# 📈 APPLICATION 3: RETURN APP (ETF Analyzer)

## Database Information
- **Database Name:** `db.sqlite3`
- **Location:** `return_app/db.sqlite3`
- **Purpose:** ETF return analysis & visualization
- **Status:** ⚠️ Minimal - Only Django system tables

## Database Tables (4 Total)

### Custom Application Tables (0 Tables)
**NONE** - The `etf_analyzer/models.py` file is empty!

```python
# etf_analyzer/models.py
from django.db import models
# Create your models here.
```

**This means:** Return app has NO custom data storage!

### Django System Tables Only (4 Tables)
- `django_session` - Session storage
- `django_admin_log` - Admin log
- `django_migrations` - Migration tracking
- `auth_user` - Default Django users
- `auth_permission` - Permissions
- `auth_group` - User groups
- `django_content_type` - Content types

### Total Tables: ~6-7 (Django defaults only)
### Estimated Size: 1-5 MB (minimal)

### How Does Return App Work Without Database?

```
┌─────────────────────────────────────────────────────────────────────────┐
│  RETURN APP ARCHITECTURE                                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  User uploads CSV file with returns data                                │
│         ↓                                                                │
│  Backend processes in-memory (pandas DataFrame)                         │
│         ↓                                                                │
│  Generates charts & visualizations                                      │
│         ↓                                                                │
│  Sends results to browser                                               │
│         ↓                                                                │
│  NO DATA SAVED TO DATABASE                                              │
│                                                                          │
│  ⚠️ All processing is stateless/in-memory                               │
│  ⚠️ No history tracking                                                 │
│  ⚠️ No saved analysis feature                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Critical Data: **NONE** - All computations are stateless

---

# 💳 APPLICATION 4: CREDIT ANALYZER APP

## Database Information
- **Database Name:** `db.sqlite3`
- **Location:** `Credit_Analyzer_App/db.sqlite3`
- **Purpose:** Credit analysis tools
- **Status:** ⚠️ Minimal - Only Django system tables

## Database Tables (4 Total)

### Custom Application Tables (0 Tables)
**NONE** - The `credit_analyzer/models.py` file is empty!

```python
# credit_analyzer/models.py
from django.db import models
# Create your models here.
```

**This means:** Credit Analyzer app has NO custom data storage!

### Django System Tables Only (4 Tables)
- `django_session` - Session storage
- `django_admin_log` - Admin log
- `django_migrations` - Migration tracking
- `auth_user` - Default Django users
- `auth_permission` - Permissions
- `auth_group` - User groups
- `django_content_type` - Content types

### Total Tables: ~6-7 (Django defaults only)
### Estimated Size: 1-5 MB (minimal)

### How Does Credit Analyzer Work Without Database?

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CREDIT ANALYZER ARCHITECTURE                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  User inputs credit analysis parameters                                 │
│         ↓                                                                │
│  Backend processes in-memory                                            │
│         ↓                                                                │
│  Generates credit reports & analysis                                    │
│         ↓                                                                │
│  Sends results to browser                                               │
│         ↓                                                                │
│  NO DATA SAVED TO DATABASE                                              │
│                                                                          │
│  ⚠️ All processing is stateless/in-memory                               │
│  ⚠️ No history tracking                                                 │
│  ⚠️ No saved analysis feature                                           │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Critical Data: **NONE** - All computations are stateless

---

# 📊 COMPLETE DATABASE SUMMARY - ALL APPS

## Database Inventory Table

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  App Name         Database File         Custom Tables   System Tables  Total │
├──────────────────────────────────────────────────────────────────────────────┤
│  Umbrella App     umbrella_db.sqlite3   5               4              9     │
│  Margin App       db.sqlite3            4               4              8     │
│  Return App       db.sqlite3            0               6-7            6-7   │
│  Credit App       db.sqlite3            0               6-7            6-7   │
├──────────────────────────────────────────────────────────────────────────────┤
│  TOTALS:          4 databases           9 custom        20+ system     29+   │
└──────────────────────────────────────────────────────────────────────────────┘
```

## Critical Data Distribution

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CRITICAL DATA BY APPLICATION                                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  🔴 UMBRELLA APP - CRITICAL                                             │
│  ────────────────────────────────────                                   │
│  • User accounts & passwords                                            │
│  • Login audit logs                                                     │
│  • Invitation tokens                                                    │
│  → MUST migrate to PostgreSQL for production                            │
│                                                                          │
│  🔴 MARGIN APP - CRITICAL                                               │
│  ────────────────────────────────                                       │
│  • User's saved backtest results (2-4 MB each)                          │
│  • Up to 50 results per user                                            │
│  → MUST migrate to PostgreSQL for production                            │
│                                                                          │
│  🟡 MARGIN APP - MEDIUM                                                 │
│  ────────────────────────────────                                       │
│  • Calculation history (can be recalculated)                            │
│  • Session tracking (analytics only)                                    │
│                                                                          │
│  🟢 MARGIN APP - LOW (Temporary)                                        │
│  ────────────────────────────────────                                   │
│  • Backtest cache (24hr TTL, auto-expires)                              │
│                                                                          │
│  🟢 RETURN APP - NONE                                                   │
│  ────────────────────────────────────                                   │
│  • No custom data storage                                               │
│  • All processing is stateless                                          │
│                                                                          │
│  🟢 CREDIT APP - NONE                                                   │
│  ────────────────────────────────────                                   │
│  • No custom data storage                                               │
│  • All processing is stateless                                          │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

# 🗺️ INTER-APP DATABASE RELATIONSHIPS

## How Applications Connect

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DATABASE RELATIONSHIP MAP                             │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────┐
│  UMBRELLA APP                        │
│  umbrella_db.sqlite3                 │
│  ┌────────────────────────────────┐  │
│  │  authentication_user           │  │
│  │  ─────────────────────         │  │
│  │  • id: 1 (John Doe)            │  │
│  │  • id: 2 (Jane Smith)          │  │
│  │  • id: 3 (Bob Johnson)         │  │
│  └────────────────────────────────┘  │
└───────────────┬──────────────────────┘
                │
                │ Session Data (user_id passed via proxy)
                │ NO direct database connection!
                ↓
┌──────────────────────────────────────┐
│  MARGIN APP                          │
│  db.sqlite3                          │
│  ┌────────────────────────────────┐  │
│  │  savedbacktestresult           │  │
│  │  ─────────────────────         │  │
│  │  • user_id: 1 (stored as int)  │  │
│  │  • user_id: 1                  │  │
│  │  • user_id: 2                  │  │
│  │  • user_id: 3                  │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
        ↑
        │ Note: user_id is just an INTEGER field
        │ NOT a foreign key constraint
        │ Linked via session, not database FK
        └────────────────────────────────────────┘

┌──────────────────────────────────────┐
│  RETURN APP                          │
│  db.sqlite3                          │
│  ┌────────────────────────────────┐  │
│  │  NO CUSTOM TABLES              │  │
│  │  (only Django defaults)        │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│  CREDIT APP                          │
│  db.sqlite3                          │
│  ┌────────────────────────────────┐  │
│  │  NO CUSTOM TABLES              │  │
│  │  (only Django defaults)        │  │
│  └────────────────────────────────┘  │
└──────────────────────────────────────┘
```

### Key Points:
- **NO foreign key constraints between databases**
- User linking happens via **session data**, not database FKs
- Each app has its **own independent database**
- Databases are **not directly connected**

---

# 🚨 PRODUCTION DEPLOYMENT RISKS - ALL APPS

## Risk Matrix

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Application    Critical Data?   SQLite Risk Level   PostgreSQL Required?   │
├──────────────────────────────────────────────────────────────────────────────┤
│  Umbrella App   🔴 YES           🔴 CRITICAL          ✅ YES - MANDATORY     │
│                 (user accounts)                                              │
│                                                                              │
│  Margin App     🔴 YES           🔴 CRITICAL          ✅ YES - MANDATORY     │
│                 (saved results)                                              │
│                                                                              │
│  Return App     🟢 NO            🟢 LOW               ❌ NO - Optional       │
│                 (stateless)                                                  │
│                                                                              │
│  Credit App     🟢 NO            🟢 LOW               ❌ NO - Optional       │
│                 (stateless)                                                  │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Priority Migration Plan

```
PRIORITY 1 (CRITICAL): Umbrella App
────────────────────────────────────
Why: Contains ALL user accounts
Risk: Lose all users = app is unusable
Action: Migrate to PostgreSQL RDS FIRST
Cost: ~$18/month
Timeline: Before any production launch

PRIORITY 2 (CRITICAL): Margin App
────────────────────────────────────
Why: Contains users' saved work
Risk: Users lose all saved backtest results
Action: Migrate to PostgreSQL RDS
Cost: ~$18/month (can share with Umbrella)
Timeline: Before production launch

PRIORITY 3 (OPTIONAL): Return App
────────────────────────────────────
Why: No custom data storage
Risk: None (stateless app)
Action: Can keep SQLite or share PostgreSQL
Cost: $0 (SQLite) or shared
Timeline: Not urgent

PRIORITY 4 (OPTIONAL): Credit App
────────────────────────────────────
Why: No custom data storage
Risk: None (stateless app)
Action: Can keep SQLite or share PostgreSQL
Cost: $0 (SQLite) or shared
Timeline: Not urgent
```

---

# 💾 SHARED POSTGRESQL STRATEGY (RECOMMENDED)

## Single PostgreSQL for All Apps

Instead of 4 separate databases, use **one PostgreSQL RDS** with **separate schemas**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  AWS RDS PostgreSQL Instance                                            │
│  Endpoint: apps-db.xxx.us-east-1.rds.amazonaws.com                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Database: production_db                                                │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Schema: umbrella (or public)                                     │ │
│  │  ────────────────────────────                                     │ │
│  │  • authentication_user                                            │ │
│  │  • authentication_invitetoken                                     │ │
│  │  • authentication_usersession                                     │ │
│  │  • authentication_loginattempt                                    │ │
│  │  • authentication_invitetemplate                                  │ │
│  │  • django_session (umbrella app sessions)                         │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Schema: margin (or use table prefixes)                           │ │
│  │  ────────────────────────────                                     │ │
│  │  • margin_calculator_margincalculation                            │ │
│  │  • historical_backtest_backtestcache                              │ │
│  │  • historical_backtest_backtestsession                            │ │
│  │  • historical_backtest_savedbacktestresult                        │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Schema: returns                                                  │ │
│  │  ────────────────────────────                                     │ │
│  │  • django_session (return app sessions)                           │ │
│  │  • (future: custom return app tables if needed)                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  Schema: credit                                                   │ │
│  │  ────────────────────────────                                     │ │
│  │  • django_session (credit app sessions)                           │ │
│  │  • (future: custom credit app tables if needed)                   │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘

Cost: ~$18-25/month (ONE RDS instance for all apps!)
vs
$18 × 4 = $72/month (separate RDS per app)

SAVINGS: ~$47-54/month!
```

### Configuration for Shared PostgreSQL

```python
# umbrella_app/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'production_db',
        'USER': 'umbrella_user',
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': 'apps-db.xxx.rds.amazonaws.com',
        'PORT': '5432',
        'OPTIONS': {
            'options': '-c search_path=umbrella,public'
        }
    }
}

# margin_app/settings.py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'production_db',
        'USER': 'margin_user',
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': 'apps-db.xxx.rds.amazonaws.com',
        'PORT': '5432',
        'OPTIONS': {
            'options': '-c search_path=margin,public'
        }
    }
}

# Similar for return_app and credit_app...
```

---

# 📈 DATA STORAGE BREAKDOWN - ALL APPS

## Storage by Data Type

```
┌──────────────────────────────────────────────────────────────────────┐
│  DATA TYPE                        SIZE           CRITICAL?    APP     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  USER ACCOUNTS                    ~500 bytes     🔴 CRITICAL  Umbrella│
│  • Usernames, emails, passwords (hashed)                             │
│  • Profile information                                               │
│  • Approval status                                                   │
│                                                                       │
│  LOGIN AUDIT LOGS                 ~350 bytes     🔴 HIGH      Umbrella│
│  • Login attempts (success/fail)                                     │
│  • IP addresses, timestamps                                          │
│  • Security tracking                                                 │
│                                                                       │
│  ACTIVE SESSIONS                  ~400 bytes     🟡 MEDIUM    Umbrella│
│  • Current logged-in users                                           │
│  • Session keys                                                      │
│                                                                       │
│  INVITATION TOKENS                ~300 bytes     🟡 MEDIUM    Umbrella│
│  • Pending user invitations                                          │
│  • Email tracking                                                    │
│                                                                       │
│  SAVED BACKTEST RESULTS           2-4 MB         🔴 CRITICAL  Margin  │
│  • Complete analysis outputs                                         │
│  • User's saved work                                                 │
│  • Charts, metrics, data                                             │
│                                                                       │
│  BACKTEST CACHE                   2-4 MB         🟢 LOW       Margin  │
│  • Temporary (24hr TTL)                                              │
│  • Performance optimization                                          │
│  • Auto-expires                                                      │
│                                                                       │
│  MARGIN CALCULATIONS              ~1-2 KB        🟢 LOW       Margin  │
│  • Calculation history                                               │
│  • Can be recalculated                                               │
│                                                                       │
│  SESSION TRACKING                 ~500 bytes     🟢 LOW       Margin  │
│  • Analytics only                                                    │
│  • Performance monitoring                                            │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

## Total Storage Estimates

```
┌──────────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT SIZE ESTIMATES                                           │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Development (10 users, testing):                                    │
│  ────────────────────────────────────                                │
│  • Umbrella App:     10 MB                                           │
│  • Margin App:       500 MB (with saved results)                     │
│  • Return App:       2 MB                                            │
│  • Credit App:       2 MB                                            │
│  • TOTAL:            ~514 MB                                         │
│                                                                       │
│  Small Production (50 users):                                        │
│  ────────────────────────────────────                                │
│  • Umbrella App:     50 MB                                           │
│  • Margin App:       5-10 GB (50 users × 50 results each)            │
│  • Return App:       5 MB                                            │
│  • Credit App:       5 MB                                            │
│  • TOTAL:            ~5-10 GB                                        │
│                                                                       │
│  Large Production (500 users):                                       │
│  ────────────────────────────────────                                │
│  • Umbrella App:     500 MB                                          │
│  • Margin App:       50-100 GB (500 users × 50 results each)         │
│  • Return App:       50 MB                                           │
│  • Credit App:       50 MB                                           │
│  • TOTAL:            ~50-100 GB                                      │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

---

# ⚙️ PRODUCTION MIGRATION STRATEGY

## Recommended Approach: Shared PostgreSQL RDS

### Step 1: Create Single RDS Instance

```
AWS RDS Configuration:
─────────────────────

Instance Class: db.t3.small
  • 2 vCPUs
  • 2 GB RAM
  • 50 GB Storage (auto-scaling enabled)
  
Database Engine: PostgreSQL 15
Storage: GP3 SSD (fast, scalable)
Multi-AZ: Yes (high availability)
Backup: 7-day retention
Encryption: Enabled (AES-256)

Monthly Cost: ~$35-40
```

### Step 2: Configure Each App

**Umbrella App:**
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'production_db',
        'USER': 'umbrella_user',
        'PASSWORD': os.environ['UMBRELLA_DB_PASSWORD'],
        'HOST': os.environ['RDS_ENDPOINT'],
        'OPTIONS': {'options': '-c search_path=umbrella'}
    }
}
```

**Margin App:**
```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'production_db',
        'USER': 'margin_user',
        'PASSWORD': os.environ['MARGIN_DB_PASSWORD'],
        'HOST': os.environ['RDS_ENDPOINT'],
        'OPTIONS': {'options': '-c search_path=margin'}
    }
}
```

**Return App & Credit App:** Similar configuration

### Step 3: Run Migrations

```bash
# For each app on EC2:

# Umbrella App
cd /var/www/umbrella_app
python manage.py migrate

# Margin App
cd /var/www/margin_app
python manage.py migrate

# Return App
cd /var/www/return_app
python manage.py migrate

# Credit App
cd /var/www/credit_analyzer
python manage.py migrate
```

---

# 📋 FINAL SUMMARY

## Database Count: **4 Databases**

1. ✅ **umbrella_db.sqlite3** - 9 tables (5 custom, 4 Django)
2. ✅ **margin_app/db.sqlite3** - 8 tables (4 custom, 4 Django)
3. ✅ **return_app/db.sqlite3** - ~6 tables (0 custom, 6 Django)
4. ✅ **credit_app/db.sqlite3** - ~6 tables (0 custom, 6 Django)

## What Data is Stored

### UMBRELLA APP (CRITICAL)
- ✅ User accounts (username, email, hashed password)
- ✅ Login audit logs (security tracking)
- ✅ Active sessions (who's logged in)
- ✅ Invitation tokens (user onboarding)
- ✅ Email templates

### MARGIN APP (CRITICAL)
- ✅ Saved backtest results (2-4 MB each, user's work)
- ✅ Margin calculation history
- ✅ Backtest session tracking
- ✅ Performance cache (temporary)

### RETURN APP (MINIMAL)
- ❌ No custom application data
- ✅ Only Django system tables (sessions, admin log)

### CREDIT APP (MINIMAL)
- ❌ No custom application data
- ✅ Only Django system tables (sessions, admin log)

## Production Readiness

```
┌──────────────────────────────────────────────────────────────┐
│  Current State:         ❌ NOT PRODUCTION READY              │
│  Reason:                All apps using SQLite                │
│  Risk Level:            🔴 CRITICAL for data loss            │
│                                                               │
│  Required Actions:                                           │
│  1. Migrate Umbrella App → PostgreSQL RDS                    │
│  2. Migrate Margin App → PostgreSQL RDS (can share)          │
│  3. Return/Credit Apps → Can keep SQLite (no critical data)  │
│                                                               │
│  Timeline:              BEFORE production launch             │
│  Estimated Cost:        ~$35-40/month (shared RDS)           │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## Quick Decision Matrix

**Question:** Should I migrate to PostgreSQL?

```
IF deploying to production with real users:
  → YES, migrate Umbrella + Margin apps to PostgreSQL RDS
  
IF just testing/demo for < 1 week:
  → SQLite is acceptable (backup manually)
  
IF running locally for development:
  → SQLite is perfect (keep as-is)
  
IF you care about your users' data:
  → PostgreSQL is mandatory
  
IF budget is extremely limited:
  → At minimum, backup SQLite files daily to S3
```

---

**End of Report**

**Next Steps:**
1. Review this report
2. Decide on production timeline
3. Follow `DATABASE_DEPLOYMENT_GUIDE.md` for PostgreSQL migration
4. Test thoroughly before launch

