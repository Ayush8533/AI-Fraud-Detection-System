📊 Database Schema
fraud_detection_db/
├── users (4 demo accounts)
│   ├── user_id (PK)
│   ├── full_name
│   ├── email (UNIQUE)
│   ├── password_hash
│   ├── role (CUSTOMER/ADMIN/INVESTIGATOR)
│   ├── account_status (ACTIVE/BLOCKED/SUSPENDED)
│   ├── created_at
│   └── updated_at
│
├── accounts (2 bank accounts with ₹150k total)
│   ├── account_id (PK)
│   ├── user_id (FK)
│   ├── account_number (UNIQUE)
│   ├── account_type (SAVINGS/CURRENT)
│   ├── balance (DECIMAL)
│   ├── currency
│   ├── account_status
│   └── created_at
│
├── transactions (transaction history)
│   ├── transaction_id (PK)
│   ├── transaction_reference (UNIQUE)
│   ├── sender_account_id (FK)
│   ├── receiver_account_id (FK)
│   ├── amount (DECIMAL)
│   ├── transaction_type (TRANSFER/PAYMENT/WITHDRAWAL/DEPOSIT)
│   ├── transaction_time
│   ├── device_id (FK)
│   ├── ip_address
│   ├── location
│   ├── status (PENDING/COMPLETED/FAILED/BLOCKED)
│   └── created_at
│
├── fraud_analysis (ML predictions)
│   ├── analysis_id (PK)
│   ├── transaction_id (FK, UNIQUE)
│   ├── risk_score (0-100)
│   ├── prediction (NORMAL/SUSPICIOUS/FRAUD)
│   ├── model_name
│   ├── model_version
│   ├── analysis_reason (TEXT)
│   └── analyzed_at
│
├── fraud_alerts (investigation cases)
│   ├── alert_id (PK)
│   ├── transaction_id (FK)
│   ├── severity (LOW/MEDIUM/HIGH/CRITICAL)
│   ├── alert_reason (TEXT)
│   ├── alert_status (OPEN/UNDER_REVIEW/RESOLVED/FALSE_POSITIVE)
│   ├── assigned_to (FK to users)
│   ├── investigator_notes (TEXT)
│   ├── created_at
│   └── resolved_at
│
├── devices (device tracking)
│   ├── device_id (PK)
│   ├── user_id (FK)
│   ├── device_identifier
│   ├── device_type (Mobile/Desktop/Tablet)
│   ├── operating_system
│   ├── ip_address
│   ├── location
│   ├── first_seen
│   └── last_seen
│
└── login_history (audit trail)
    ├── login_id (PK)
    ├── user_id (FK)
    ├── login_time
    ├── ip_address
    ├── location
    ├── device_id (FK)
    └── login_status (SUCCESS/FAILED)
📈 Database Relationships
users ──────┬──→ accounts ──→ transactions ──┬──→ fraud_analysis
            │                            ↓   └──→ fraud_alerts
            │                         devices
            ├──→ devices ──→ transactions
            │
            └──→ login_history
            └──→ fraud_alerts (assigned_to)
🔑 Key Indexes
sql
-- Transaction Queries
CREATE INDEX idx_transactions_sender ON transactions(sender_account_id);
CREATE INDEX idx_transactions_receiver ON transactions(receiver_account_id);
CREATE INDEX idx_transactions_time ON transactions(transaction_time);
CREATE INDEX idx_transactions_status ON transactions(status);

-- Fraud Analysis Queries
CREATE INDEX idx_fraud_prediction ON fraud_analysis(prediction);

-- Alert Management Queries
CREATE INDEX idx_alert_status ON fraud_alerts(alert_status);
CREATE INDEX idx_alert_severity ON fraud_alerts(severity);
📋 Demo Data
Users Table (4 accounts)
user_id	full_name	email	role	status
1	System Admin	admin@fraud.local	ADMIN	ACTIVE
2	Fraud Investigator	investigator@fraud.local	INVESTIGATOR	ACTIVE
3	Demo Customer	user@fraud.local	CUSTOMER	ACTIVE
4	Demo Customer Two	user2@fraud.local	CUSTOMER	ACTIVE
Accounts Table (2 accounts)
account_id	user_id	account_number	account_type	balance	status
1	3	ACCT100001	SAVINGS	100000.00	ACTIVE
2	4	ACCT100002	SAVINGS	50000.00	ACTIVE
🔄 Data Flow
1. Customer initiates transaction
   └─→ INSERT into transactions table (status: PENDING)

2. Transaction data captured
   ├─ Amount
   ├─ Time
   ├─ Location
   ├─ Device info
   └─ IP address

3. ML Model analyzes transaction
   └─→ INSERT into fraud_analysis table
       ├─ Extract features
       ├─ Run Isolation Forest
       ├─ Calculate risk score (0-100)
       └─ Generate prediction

4. System evaluates risk score
   ├─ If risk_score > 66 → FRAUD
   │   └─→ INSERT into fraud_alerts (CRITICAL)
   │   └─→ UPDATE transactions.status = BLOCKED
   │
   ├─ If 34 < risk_score < 66 → SUSPICIOUS
   │   └─→ INSERT into fraud_alerts (HIGH/MEDIUM)
   │
   └─ If risk_score < 34 → NORMAL
       └─→ UPDATE transactions.status = COMPLETED

5. Investigator reviews alert
   └─→ UPDATE fraud_alerts
       ├─ Status: UNDER_REVIEW
       ├─ Assigned to: Investigator
       └─ Notes: Investigation details

6. Investigation complete
   └─→ UPDATE fraud_alerts
       ├─ Status: RESOLVED
       ├─ Resolved_at: Timestamp
       └─ Final notes

7. Transaction finalized
   └─→ UPDATE transactions
       └─ Status: COMPLETED or FAILED
🎯 Database Queries (Common)
Get User Transactions
sql
SELECT t.*, fa.prediction, fa.risk_score
FROM transactions t
LEFT JOIN fraud_analysis fa ON t.transaction_id = fa.transaction_id
WHERE t.sender_account_id IN (
    SELECT account_id FROM accounts WHERE user_id = 3
)
ORDER BY t.transaction_time DESC;
Find Fraudulent Transactions
sql
SELECT t.*, fa.risk_score, fa.prediction
FROM transactions t
JOIN fraud_analysis fa ON t.transaction_id = fa.transaction_id
WHERE fa.prediction = 'FRAUD'
ORDER BY fa.risk_score DESC;
Get Open Fraud Alerts
sql
SELECT a.*, t.amount, acc.account_number
FROM fraud_alerts a
JOIN transactions t ON a.transaction_id = t.transaction_id
JOIN accounts acc ON t.sender_account_id = acc.account_id
WHERE a.alert_status = 'OPEN'
ORDER BY a.severity DESC, a.created_at DESC;
Investigator Workload
sql
SELECT assigned_to, COUNT(*) as open_cases
FROM fraud_alerts
WHERE alert_status = 'OPEN'
GROUP BY assigned_to
ORDER BY open_cases DESC;
Transaction Statistics
sql
SELECT 
    DATE(transaction_time) as date,
    COUNT(*) as total_transactions,
    SUM(amount) as total_amount,
    COUNT(CASE WHEN prediction = 'FRAUD' THEN 1 END) as fraud_count
FROM transactions t
LEFT JOIN fraud_analysis fa ON t.transaction_id = fa.transaction_id
WHERE transaction_time >= DATE_SUB(NOW(), INTERVAL 30 DAY)
GROUP BY DATE(transaction_time)
ORDER BY date DESC;
🔐 Constraints & Rules
sql
-- Foreign Key Constraints
ALTER TABLE accounts 
ADD CONSTRAINT fk_accounts_users 
FOREIGN KEY (user_id) REFERENCES users(user_id);

ALTER TABLE transactions 
ADD CONSTRAINT fk_transactions_sender 
FOREIGN KEY (sender_account_id) REFERENCES accounts(account_id);

ALTER TABLE transactions 
ADD CONSTRAINT fk_transactions_receiver 
FOREIGN KEY (receiver_account_id) REFERENCES accounts(account_id);

ALTER TABLE fraud_analysis 
ADD CONSTRAINT fk_analysis_transactions 
FOREIGN KEY (transaction_id) REFERENCES transactions(transaction_id) UNIQUE;

ALTER TABLE fraud_alerts 
ADD CONSTRAINT fk_alerts_transactions 
FOREIGN KEY (transaction_id) REFERENCES transactions(transaction_id);

-- Check Constraints
ALTER TABLE transactions 
ADD CONSTRAINT check_positive_amount 
CHECK (amount > 0);

ALTER TABLE fraud_analysis 
ADD CONSTRAINT check_risk_score 
CHECK (risk_score >= 0 AND risk_score <= 100);
📊 Database Optimization
Performance Tuning
sql
-- Analyze table for query optimization
ANALYZE TABLE transactions;
ANALYZE TABLE fraud_analysis;
ANALYZE TABLE fraud_alerts;

-- Check index statistics
SELECT * FROM information_schema.STATISTICS 
WHERE TABLE_NAME = 'transactions';

-- Monitor query performance
EXPLAIN SELECT * FROM transactions 
WHERE sender_account_id = 1 
ORDER BY transaction_time DESC LIMIT 10;
Backup & Recovery
bash
# Backup entire database
mysqldump -u root -p fraud_detection_db > backup.sql

# Restore database
mysql -u root -p fraud_detection_db < backup.sql
💾 Storage Requirements
Estimated Storage:
├── Users: ~2 KB
├── Accounts: ~1 KB
├── Transactions: ~500 MB (1M records)
├── Fraud Analysis: ~250 MB (1M records)
├── Fraud Alerts: ~100 MB (200k records)
├── Devices: ~50 MB
└── Login History: ~200 MB

Total Estimated: ~1.1 GB (at scale)
🎯 Data Retention Policy
Users: Keep indefinitely
Accounts: Keep indefinitely
Transactions: Keep for 7 years (compliance)
Fraud Analysis: Keep for 3 years
Fraud Alerts: Keep for 2 years
Devices: Keep last 2 years
Login History: Keep last 1 year
