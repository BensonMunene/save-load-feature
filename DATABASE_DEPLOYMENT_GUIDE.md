# 🚨 Database Deployment Guide: SQLite vs PostgreSQL on AWS

## CRITICAL: Understanding Database Deployment

---

## ❌ What's WRONG with SQLite in Production

### Current Development Setup

```
┌─────────────────────────────────────────────────────────────────┐
│  YOUR LAPTOP (Development)                                      │
│  ────────────────────────                                       │
│                                                                  │
│  margin_app/                                                     │
│  ├── manage.py                                                   │
│  ├── db.sqlite3  ← DATABASE FILE (NOT in git)                   │
│  ├── settings.py                                                 │
│  └── ...                                                         │
│                                                                  │
│  When you save results:                                         │
│  User data → db.sqlite3 (local file on your hard drive)         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### What Happens When You Push to GitHub?

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 1: Git Push to GitHub                                     │
│  ────────────────────────────                                   │
│                                                                  │
│  Your Laptop                    GitHub Repository               │
│  ├── manage.py      ────→      ├── manage.py       ✓           │
│  ├── db.sqlite3     ✗✗✗✗       ├── db.sqlite3      ✗ NOT PUSHED│
│  ├── settings.py    ────→      ├── settings.py     ✓           │
│  └── views.py       ────→      └── views.py        ✓           │
│                                                                  │
│  ⚠️ db.sqlite3 is in .gitignore → NOT pushed to GitHub         │
│  ⚠️ Your saved results stay ONLY on your laptop                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### What Happens on AWS Deployment?

```
┌─────────────────────────────────────────────────────────────────┐
│  STEP 2: AWS Pulls from GitHub                                  │
│  ────────────────────────────────                               │
│                                                                  │
│  AWS EC2 Instance                                               │
│  /var/www/margin_app/                                           │
│  ├── manage.py       ✓ Downloaded from GitHub                   │
│  ├── db.sqlite3      ✗ DOES NOT EXIST (not in GitHub)          │
│  ├── settings.py     ✓ Downloaded                               │
│  └── views.py        ✓ Downloaded                               │
│                                                                  │
│  When Django starts:                                            │
│  → Creates NEW EMPTY db.sqlite3                                 │
│  → ALL your development data is GONE                            │
│  → Database starts fresh with 0 records                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔍 CRITICAL DISTINCTION: CI/CD Updates vs Fresh Deployment

### Understanding When Data is Lost vs Preserved

**This is the most important concept to understand about SQLite on AWS:**

#### ✅ CI/CD Updates (GitHub Actions) - Data Usually Preserved

```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: Code updates via CI/CD pipeline                      │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  Developer pushes code → GitHub Actions triggers                │
│          ↓                                                       │
│  Pipeline connects to EXISTING EC2 instance via SSH             │
│          ↓                                                       │
│  Actions performed:                                             │
│  1. git pull origin main          ← Only code updated           │
│  2. pip install -r requirements.txt                             │
│  3. python manage.py migrate      ← Database updated, not wiped │
│  4. python manage.py collectstatic                              │
│  5. sudo systemctl restart gunicorn                             │
│          ↓                                                       │
│  Result: db.sqlite3 file stays on disk                          │
│         ✓ User data preserved                                   │
│         ✓ Saved backtest results still there                    │
│         ✓ User accounts intact                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**In this case:** SQLite data survives because:
- The EC2 instance **keeps running**
- Only **code files** are updated
- The `db.sqlite3` file is **not touched**
- Database file remains on the EC2 disk

---

#### ❌ Fresh Deployment - Data LOST

```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIOS WHERE DATA IS LOST:                                  │
│  ──────────────────────────────────────────────────────────     │
│                                                                  │
│  1. EC2 INSTANCE TERMINATED                                     │
│     • Auto-scaling scales down                                  │
│     • Cost optimization (terminate to save money)               │
│     • Instance becomes unhealthy                                │
│     • Manual termination                                        │
│     → Entire EC2 disk deleted                                   │
│     → db.sqlite3 GONE FOREVER                                   │
│                                                                  │
│  2. NEW EC2 INSTANCE CREATED                                    │
│     • Fresh Ubuntu installation                                 │
│     • Clone code from GitHub                                    │
│     • db.sqlite3 NOT in repository                              │
│     • Django creates NEW EMPTY database                         │
│     → All user data LOST                                        │
│                                                                  │
│  3. DOCKER CONTAINER DEPLOYMENT                                 │
│     • Container restarts = new filesystem                       │
│     • Unless volumes configured (often not)                     │
│     → db.sqlite3 lost on restart                                │
│                                                                  │
│  4. ELASTIC BEANSTALK / ECS / EKS                               │
│     • AWS can replace instances anytime                         │
│     • Instances are "cattle not pets"                           │
│     → Data not guaranteed to persist                            │
│                                                                  │
│  5. DISK FAILURE                                                │
│     • Hard drive crashes                                        │
│     • Filesystem corruption                                     │
│     → No recovery without backups                               │
│                                                                  │
│  6. DISASTER RECOVERY TEST                                      │
│     • Need to restore from backup                               │
│     → No backups exist for SQLite                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 📊 Detailed Comparison Table

```
┌──────────────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT SCENARIO              SQLite on EC2      PostgreSQL RDS      │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  CI/CD Code Updates:                                                     │
│  (git pull, restart app)          ✓ Data preserved   ✓ Data preserved    │
│                                                                           │
│  Database Schema Changes:                                                │
│  (python manage.py migrate)       ✓ Data preserved   ✓ Data preserved    │
│                                                                           │
│  EC2 Instance Terminated:                                                │
│  (manual, auto-scaling, spot)     ❌ Data LOST       ✓ Data SAFE         │
│                                                                           │
│  Application Crash:                                                      │
│  (restart required)               ✓ Data preserved   ✓ Data preserved    │
│                                                                           │
│  Disk Failure:                                                           │
│  (hardware issue)                 ❌ Data LOST       ✓ Recovered         │
│                                                                           │
│  Accidental File Deletion:                                               │
│  (rm db.sqlite3)                  ❌ Data LOST       ✓ Data SAFE         │
│                                                                           │
│  Ransomware/Corruption:                                                  │
│  (file encrypted/corrupted)       ❌ Data LOST       ✓ Restore backup    │
│                                                                           │
│  Auto-Scaling (Multiple EC2s):                                           │
│  (load balancer + 2+ instances)   ❌ Inconsistent    ✓ Shared DB         │
│                                                                           │
│  Blue/Green Deployment:                                                  │
│  (new instance, switch traffic)   ❌ Data LOST       ✓ Same DB           │
│                                                                           │
│  Backup & Recovery:                                                      │
│  (restore to any point in time)   ❌ Manual only     ✓ Automatic         │
│                                                                           │
│  High Availability:                                                      │
│  (failover to standby)            ❌ None            ✓ Multi-AZ           │
│                                                                           │
│  Disaster Recovery:                                                      │
│  (region failure)                 ❌ None            ✓ Cross-region       │
│                                                                           │
│  Performance Monitoring:                                                 │
│  (query insights, slow logs)      ❌ Manual          ✓ CloudWatch         │
│                                                                           │
│  Encryption at Rest:                                                     │
│  (data security)                  ❌ Manual/DIY      ✓ Built-in           │
│                                                                           │
│  Concurrent Users:                                                       │
│  (50+ users writing data)         ❌ Locks/errors    ✓ 1000+ connections │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

---

### ⚠️ The Hidden Risks Even WITH CI/CD

**Even if your CI/CD preserves the database, you still face these risks:**

#### Risk 1: No Backup Strategy
```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: 6 months of user data accumulated                    │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  • 100 users have saved 2,000 backtest results                  │
│  • Database file = 8 GB of valuable data                        │
│  • No backups configured                                        │
│                                                                  │
│  Day 180: Developer accidentally runs command:                  │
│           rm db.sqlite3                                          │
│           (thinking it's a test file)                            │
│                                                                  │
│  Result: 6 months of data GONE                                  │
│         No way to recover                                       │
│         Users lose all saved work                               │
│         Trust destroyed                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Risk 2: AWS Spot Instance Termination
```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: Using EC2 Spot Instances (60-90% cheaper)            │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  Monday 2:00 AM:                                                │
│  AWS: "Spot price exceeded your bid"                            │
│  AWS: "Terminating instance in 2 minutes"                       │
│         ↓                                                        │
│  Instance terminated                                            │
│         ↓                                                        │
│  ALL data on instance = DELETED                                 │
│         ↓                                                        │
│  Launch replacement instance                                    │
│         ↓                                                        │
│  NEW EMPTY database                                             │
│         ↓                                                        │
│  Users wake up → All saved results GONE                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Risk 3: Auto-Scaling Inconsistency
```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: AWS Auto-Scaling for high traffic                    │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  Load increases → AWS launches EC2 Instance #2                  │
│                                                                  │
│  EC2 Instance #1:                    EC2 Instance #2:           │
│  ├── db.sqlite3 (8 GB)              ├── db.sqlite3 (empty)      │
│  ├── 2,000 saved results            ├── 0 saved results         │
│                                                                  │
│  Load Balancer distributes traffic:                             │
│  ├── User A → Instance #1 → Sees saved results ✓                │
│  ├── User A → Instance #2 → Sees nothing! ❌                    │
│  ├── User B → Instance #2 → Saves result                        │
│  └── User B → Instance #1 → Result not there! ❌                │
│                                                                  │
│  Result: Inconsistent, confusing experience                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Risk 4: Disk Full / Corruption
```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: Database grows over time                             │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  Month 1: db.sqlite3 = 500 MB                                   │
│  Month 2: db.sqlite3 = 2 GB                                     │
│  Month 3: db.sqlite3 = 8 GB                                     │
│  Month 4: db.sqlite3 = 15 GB                                    │
│  Month 5: EC2 disk full → Write fails → File corrupted          │
│           ↓                                                      │
│  Database corruption detected                                   │
│           ↓                                                      │
│  No backups to restore from                                     │
│           ↓                                                      │
│  Data recovery tools?                                           │
│  • Maybe recover 60% of data                                    │
│  • 40% of data lost                                             │
│  • Weeks of recovery effort                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Risk 5: Regulatory Compliance
```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: Financial data compliance requirements               │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  Regulations often require:                                     │
│  • ❌ Encryption at rest (SQLite not encrypted)                 │
│  • ❌ Audit logs (who accessed what)                            │
│  • ❌ Point-in-time recovery                                    │
│  • ❌ Disaster recovery plan                                    │
│  • ❌ High availability (99.9% uptime)                          │
│                                                                  │
│  With PostgreSQL RDS:                                           │
│  • ✓ AES-256 encryption at rest                                 │
│  • ✓ CloudWatch audit logs                                      │
│  • ✓ Point-in-time recovery (1 second granularity)              │
│  • ✓ Multi-AZ failover                                          │
│  • ✓ 99.95% SLA                                                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### 🎯 When SQLite Works (Temporarily)

```
ACCEPTABLE use cases for SQLite on EC2:
────────────────────────────────────────

✓ Development environment
✓ Demo/prototype for investors
✓ Internal testing (non-production)
✓ Single-user application
✓ Data is not critical (can be lost)
✓ Short-term deployment (< 1 week)
✓ Learning/education projects

NOT ACCEPTABLE for:
───────────────────

❌ Production with real users
❌ Any paid service
❌ Critical user data
❌ Multi-user applications
❌ Long-term deployments
❌ Auto-scaling scenarios
❌ Anything requiring backups
```

---

### 💡 Real-World Disaster Stories

#### Story 1: The 2 AM Incident
```
Startup using SQLite on EC2 for 8 months
→ AWS terminated spot instance
→ 5,000 user accounts lost
→ 20,000 saved analyses lost
→ No backups
→ Company reputation destroyed
→ Mass user exodus
Cost of not using RDS: Lost business
```

#### Story 2: The Scaling Nightmare
```
App went viral on Product Hunt
→ Auto-scaling launched 5 EC2 instances
→ Each had different database state
→ Users saw random data depending on which instance they hit
→ Support tickets flooded in
→ Emergency migration to RDS at 3 AM
→ 12 hours of downtime
Cost of not using RDS: $50k in lost revenue
```

#### Story 3: The Corruption
```
Power outage at AWS datacenter
→ SQLite file corrupted during write
→ Database unrecoverable
→ 1 year of financial analysis data lost
→ Legal issues (no audit trail)
→ Clients sued for data loss
Cost of not using RDS: $500k settlement + legal fees
```

---

## 🚨 CRITICAL PROBLEMS with SQLite on AWS EC2

### Problem 1: Data Loss on Redeploy

```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: You deploy, users save data, then you redeploy       │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  Day 1: Deploy to AWS                                           │
│  ├── EC2 creates db.sqlite3                                     │
│  ├── User A saves backtest result #1                            │
│  ├── User B saves backtest result #2                            │
│  └── User C saves backtest result #3                            │
│      → db.sqlite3 now contains 3 saved results                  │
│                                                                  │
│  Day 2: You fix a bug and redeploy                              │
│  ├── EC2 pulls new code from GitHub                             │
│  ├── EC2 restarts application                                   │
│  ├── Old db.sqlite3 gets OVERWRITTEN or DELETED                 │
│  └── NEW EMPTY db.sqlite3 created                               │
│      → ALL 3 saved results are LOST FOREVER ❌                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Problem 2: Server Crashes = Data Loss

```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: EC2 instance crashes or gets terminated              │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  EC2 Instance with db.sqlite3 file                              │
│  ├── 100 users have saved 1,000 backtest results                │
│  ├── Database file = 4 GB                                       │
│  └── All data stored in single file on EC2 disk                 │
│                                                                  │
│  ⚡ EC2 instance terminates (auto-scaling, crash, etc.)         │
│      ↓                                                           │
│  💥 ENTIRE DATABASE LOST                                        │
│      ↓                                                           │
│  😱 1,000 saved results = GONE                                  │
│      ↓                                                           │
│  😱 NO BACKUPS, NO RECOVERY                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Problem 3: Multiple Users = Corruption

```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIO: 2 users save results at exact same time              │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  Time: 14:30:00.000                                             │
│  ├── User A saves result → Locks db.sqlite3 file                │
│  └── User B saves result → WAITS for lock... TIMEOUT ERROR ❌   │
│                                                                  │
│  SQLite allows only ONE writer at a time                        │
│  → User B gets "database is locked" error                       │
│  → User B's data is NOT saved                                   │
│  → Poor user experience                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Problem 4: No Backups

```
┌─────────────────────────────────────────────────────────────────┐
│  SQLite on EC2 = No Automatic Backups                           │
│  ────────────────────────────────────────────────────────       │
│                                                                  │
│  ❌ No automatic daily backups                                  │
│  ❌ No point-in-time recovery                                   │
│  ❌ No disaster recovery                                        │
│  ❌ No replication                                              │
│  ❌ No high availability                                        │
│                                                                  │
│  If data is lost → IT'S GONE FOREVER                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ✅ CORRECT SOLUTION: PostgreSQL RDS on AWS

### Production Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           AWS CLOUD                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  EC2 Instance (Application Server)                             │    │
│  │  ──────────────────────────────────────────────────────────    │    │
│  │  /var/www/margin_app/                                          │    │
│  │  ├── manage.py                                                  │    │
│  │  ├── settings.py   ← NO db.sqlite3 file!                       │    │
│  │  ├── views.py                                                   │    │
│  │  └── ...                                                        │    │
│  │                                                                  │    │
│  │  Django connects to → PostgreSQL (network connection)          │    │
│  └────────────────────────┬───────────────────────────────────────┘    │
│                           │                                              │
│                           │ Network connection                           │
│                           │ (encrypted)                                  │
│                           ↓                                              │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  RDS PostgreSQL (Separate Database Server)                     │    │
│  │  ──────────────────────────────────────────────────────────    │    │
│  │  Database: margin_production_db                                │    │
│  │  Endpoint: margin-db.xxx.us-east-1.rds.amazonaws.com          │    │
│  │                                                                  │    │
│  │  Tables:                                                        │    │
│  │  ├── margin_calculator_margincalculation                        │    │
│  │  ├── historical_backtest_backtestcache                          │    │
│  │  ├── historical_backtest_backtestsession                        │    │
│  │  └── historical_backtest_savedbacktestresult                    │    │
│  │                                                                  │    │
│  │  ✓ Automatic daily backups                                      │    │
│  │  ✓ Point-in-time recovery (restore to any second)              │    │
│  │  ✓ Multi-AZ replication (high availability)                     │    │
│  │  ✓ Automatic failover                                           │    │
│  │  ✓ Encryption at rest                                           │    │
│  │  ✓ 1000+ concurrent connections                                 │    │
│  │  ✓ Persistent storage (survives EC2 restarts)                   │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### How User Data Flows in Production

```
┌─────────────────────────────────────────────────────────────────┐
│  USER SAVES BACKTEST RESULT                                     │
└───────────────────┬─────────────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────────────┐
│  User's Browser                                                  │
│  POST /historical-backtest/save-result/                         │
│  Body: { name: "SPY 2x", results: {...3 MB...} }                │
└───────────────────┬─────────────────────────────────────────────┘
                    │ HTTPS over Internet
                    ↓
┌─────────────────────────────────────────────────────────────────┐
│  AWS ALB (Load Balancer)                                        │
│  → Routes to healthy EC2 instance                               │
└───────────────────┬─────────────────────────────────────────────┘
                    │
                    ↓
┌─────────────────────────────────────────────────────────────────┐
│  EC2 Instance (Django App)                                      │
│  views.save_backtest_result():                                  │
│    1. Validate user_id from session                             │
│    2. Check quota (< 50 results)                                │
│    3. Prepare INSERT query                                      │
└───────────────────┬─────────────────────────────────────────────┘
                    │ PostgreSQL network connection
                    │ (encrypted, inside VPC)
                    ↓
┌─────────────────────────────────────────────────────────────────┐
│  RDS PostgreSQL Database                                        │
│  INSERT INTO saved_backtest_result (...)                        │
│  VALUES (user_id=123, name="SPY 2x", results_data={...})        │
│    ↓                                                             │
│  Data written to disk (persistent storage)                      │
│    ↓                                                             │
│  Automatic backup created (within 5 minutes)                    │
│    ↓                                                             │
│  Replicated to standby instance (Multi-AZ)                      │
└───────────────────┬─────────────────────────────────────────────┘
                    │ Success response
                    ↓
┌─────────────────────────────────────────────────────────────────┐
│  User's Browser                                                  │
│  Alert: "✓ Backtest saved successfully!"                        │
│                                                                  │
│  Data is now SAFE:                                              │
│  ✓ Stored in RDS (not on EC2)                                   │
│  ✓ Backed up automatically                                      │
│  ✓ Survives EC2 restarts/crashes                                │
│  ✓ Accessible from any EC2 instance                             │
│  ✓ Can be restored if needed                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 Step-by-Step: Migrating to PostgreSQL on AWS

### Step 1: Create RDS PostgreSQL Instance

**Via AWS Console:**

```
1. Go to AWS RDS Console
2. Click "Create database"
3. Select:
   ✓ PostgreSQL
   ✓ Version: 15.x (latest stable)
   ✓ Templates: Production
   ✓ DB instance class: db.t3.micro (start small)
   ✓ Storage: 20 GB (auto-scaling enabled)
   ✓ Multi-AZ: No (for cost savings, Yes for production)
   ✓ VPC: Same as EC2
   ✓ Public access: No (security)
   ✓ Database name: margin_production_db
   ✓ Master username: postgres
   ✓ Master password: [create strong password]
   ✓ Automated backups: Enabled (7 days retention)
   ✓ Encryption: Enabled

4. Click "Create database"
5. Wait 5-10 minutes for creation
6. Note the endpoint: margin-db.xxx.us-east-1.rds.amazonaws.com
```

**Cost:** ~$15-20/month for db.t3.micro

### Step 2: Update Django Settings

**File:** `margin_app/margin_analytics/settings.py`

```python
# OLD (Development)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# NEW (Production)
import os

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME', 'margin_production_db'),
        'USER': os.environ.get('DB_USER', 'postgres'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),  # From environment variable
        'HOST': os.environ.get('DB_HOST', 'margin-db.xxx.us-east-1.rds.amazonaws.com'),
        'PORT': os.environ.get('DB_PORT', '5432'),
        'OPTIONS': {
            'sslmode': 'require',  # Encrypted connection
        },
        'CONN_MAX_AGE': 600,  # Connection pooling (10 minutes)
    }
}
```

### Step 3: Install PostgreSQL Adapter

**File:** `margin_app/requirements.txt`

```
# Add this line:
psycopg2-binary==2.9.9
```

### Step 4: Set Environment Variables on EC2

**On AWS EC2 instance:**

```bash
# Create .env file (NOT in git!)
sudo nano /var/www/margin_app/.env

# Add these lines:
DB_NAME=margin_production_db
DB_USER=postgres
DB_PASSWORD=your-strong-password-here
DB_HOST=margin-db.xxx.us-east-1.rds.amazonaws.com
DB_PORT=5432
SECRET_KEY=your-django-secret-key
DEBUG=False
```

**Load environment variables:**

```bash
# In your deployment script or systemd service:
export $(cat /var/www/margin_app/.env | xargs)
```

### Step 5: Run Migrations

**On EC2 instance:**

```bash
cd /var/www/margin_app
source venv_margin/bin/activate

# Test connection
python manage.py dbshell
# Should connect to PostgreSQL, not SQLite

# Run migrations (creates all tables)
python manage.py migrate

# Expected output:
# Operations to perform:
#   Apply all migrations: admin, auth, contenttypes, sessions, 
#                         margin_calculator, historical_backtest
# Running migrations:
#   Applying contenttypes.0001_initial... OK
#   Applying auth.0001_initial... OK
#   ...
#   Applying historical_backtest.0004_remove_tags... OK
```

### Step 6: (Optional) Migrate Existing Development Data

**If you have test data to migrate:**

```bash
# On your laptop (development)
cd margin_app
python manage.py dumpdata > production_data.json

# Upload to EC2
scp production_data.json ec2-user@your-ec2-ip:/var/www/margin_app/

# On EC2 (production)
cd /var/www/margin_app
python manage.py loaddata production_data.json
```

**⚠️ WARNING:** Only do this for initial setup, NOT for ongoing production!

### Step 7: Verify Production Database

**Test queries on EC2:**

```bash
python manage.py shell

>>> from historical_backtest.saved_results.models import SavedBacktestResult
>>> SavedBacktestResult.objects.count()
0  # Start with 0 results

>>> # Save a test result through the web interface
>>> # Then check again:
>>> SavedBacktestResult.objects.count()
1  # Should show the saved result

>>> SavedBacktestResult.objects.first()
<SavedBacktestResult: SPY 2x Test - SPY (profit_threshold)>
```

---

## 🔄 Deployment Workflow (With PostgreSQL)

### Every Time You Deploy Code Changes

```
┌─────────────────────────────────────────────────────────────────┐
│  YOUR LAPTOP                                                     │
│  ────────────                                                    │
│  1. Make code changes (views.py, templates, etc.)                │
│  2. Test locally with SQLite                                     │
│  3. Commit to git: git add . && git commit -m "..."             │
│  4. Push to GitHub: git push origin main                        │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────┐
│  GITHUB                                                          │
│  ────────                                                        │
│  Code stored in repository (NO database file)                   │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ↓
┌─────────────────────────────────────────────────────────────────┐
│  AWS EC2                                                         │
│  ────────                                                        │
│  1. SSH into EC2: ssh ec2-user@your-ec2-ip                      │
│  2. Pull latest code: git pull origin main                      │
│  3. Collect static files: python manage.py collectstatic        │
│  4. Restart application: sudo systemctl restart gunicorn        │
│                                                                  │
│  ⚠️ DATABASE IS NOT AFFECTED!                                   │
│  ✓ Data remains in RDS PostgreSQL                               │
│  ✓ No data loss on redeploy                                     │
│  ✓ Users' saved results are safe                                │
└─────────────────────────────────────────────────────────────────┘
```

### Database Changes (Migrations)

```
┌─────────────────────────────────────────────────────────────────┐
│  IF you change models.py (add/remove fields):                   │
│  ──────────────────────────────────────────────────────────     │
│                                                                  │
│  1. YOUR LAPTOP:                                                │
│     python manage.py makemigrations                             │
│     git add migrations/0005_*.py                                │
│     git commit -m "Add new field"                               │
│     git push                                                    │
│                                                                  │
│  2. AWS EC2:                                                    │
│     git pull                                                    │
│     python manage.py migrate  ← Applies to RDS PostgreSQL       │
│     sudo systemctl restart gunicorn                             │
│                                                                  │
│  ✓ Migration applied to production database                     │
│  ✓ Existing data preserved                                      │
│  ✓ New field added to all tables                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💰 Cost Comparison

```
┌──────────────────────────────────────────────────────────────────┐
│  OPTION 1: SQLite on EC2                                         │
│  ────────────────────────                                        │
│  • EC2 t3.medium: $30/month                                      │
│  • Database: $0 (file on EC2)                                    │
│  • Backups: $0 (none)                                            │
│  • Total: $30/month                                              │
│                                                                   │
│  ❌ Data loss on redeploy                                        │
│  ❌ Data loss on server crash                                    │
│  ❌ No backups                                                   │
│  ❌ Poor concurrent performance                                  │
│  ❌ NOT PRODUCTION READY                                         │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  OPTION 2: PostgreSQL RDS (RECOMMENDED)                          │
│  ───────────────────────────────────────                         │
│  • EC2 t3.medium: $30/month                                      │
│  • RDS db.t3.micro: $18/month                                    │
│  • Backups: Included (7 days)                                    │
│  • Total: $48/month                                              │
│                                                                   │
│  ✓ Data survives redeployments                                   │
│  ✓ Data survives server crashes                                  │
│  ✓ Automatic daily backups                                       │
│  ✓ Excellent concurrent performance                              │
│  ✓ PRODUCTION READY                                              │
│                                                                   │
│  Extra $18/month = Peace of mind!                                │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🎯 FINAL ANSWER TO YOUR QUESTION

### Is SQLite Production Ready?

**NO. Absolutely not for a web application with user data.**

### What Happens When You Deploy to AWS with SQLite?

```
1. GitHub does NOT contain db.sqlite3 (in .gitignore)
2. AWS creates a NEW EMPTY database when Django starts
3. Every redeploy WIPES THE DATABASE
4. User data is LOST
5. This is UNACCEPTABLE for production
```

### What SHOULD Happen?

```
1. Create PostgreSQL RDS instance on AWS
2. Update settings.py to use PostgreSQL
3. User data → Saved in RDS (persistent, backed up)
4. EC2 crashes/restarts → Data is SAFE in RDS
5. Redeploy code → Data is SAFE in RDS
6. Production ready ✓
```

### Migration Checklist

```
[ ] Create RDS PostgreSQL instance on AWS
[ ] Update settings.py with PostgreSQL config
[ ] Add psycopg2-binary to requirements.txt
[ ] Set environment variables on EC2
[ ] Run migrations on production database
[ ] Test save/load functionality
[ ] Verify backups are enabled
[ ] Deploy with confidence!
```

---

## 📚 Additional Resources

- **AWS RDS Documentation:** https://docs.aws.amazon.com/rds/
- **Django PostgreSQL Setup:** https://docs.djangoproject.com/en/4.2/ref/databases/#postgresql-notes
- **Database Migration Guide:** See `MARGIN_APP_DATABASE_ARCHITECTURE.md`

---

**Bottom Line:** Use PostgreSQL RDS on AWS for production. SQLite is for development ONLY.

