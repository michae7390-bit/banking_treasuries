# Core Banking Schema Design

## Entity-Relationship Diagram

```
CUSTOMERS
  │
  ├── ACCOUNTS
  │    ├── TRANSACTIONS
  │    ├── BALANCES
  │    └── SETTLEMENTS
  │
  ├── RISK_PROFILES
  │    ├── AML_KYC_STATUS
  │    └── COMPLIANCE_CHECKS
  │
  └── FRAUD_ALERTS
       ├── ALERT_RULES
       └── ALERT_CASES

COUNTERPARTIES
  │
  ├── RELATIONSHIPS
  ├── EXPOSURES
  └── CREDIT_LIMITS

MARKET_DATA
  ├── EXCHANGE_RATES
  ├── PRICE_TICKERS
  └── VOLATILITY_INDEX
```

## Table Specifications

### customers
```sql
CREATE TABLE customers (
  customer_id BIGINT PRIMARY KEY,
  customer_name VARCHAR(256) NOT NULL,
  date_of_birth DATE,
  nationality VARCHAR(3),  -- ISO-3166 code
  kyc_status VARCHAR(20),  -- VERIFIED, PENDING, REJECTED
  kyc_completion_date TIMESTAMP,
  pep_status BOOLEAN DEFAULT FALSE,  -- Politically Exposed Person
  risk_classification VARCHAR(20),  -- LOW, MEDIUM, HIGH, CRITICAL
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT check_kyc_status IN ('VERIFIED', 'PENDING', 'REJECTED'),
  CONSTRAINT check_risk_classification IN ('LOW', 'MEDIUM', 'HIGH', 'CRITICAL')
);
```

### accounts
```sql
CREATE TABLE accounts (
  account_id BIGINT PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customers,
  account_type VARCHAR(20),  -- CHECKING, SAVINGS, INVESTMENT
  currency_code VARCHAR(3),  -- ISO-4217 code
  balance NUMERIC(19,4) DEFAULT 0,
  available_balance NUMERIC(19,4) DEFAULT 0,
  credit_limit NUMERIC(19,4),
  status VARCHAR(20),  -- ACTIVE, SUSPENDED, CLOSED
  opened_date DATE NOT NULL,
  closed_date DATE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT fk_accounts_customer FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE INDEX idx_accounts_customer ON accounts(customer_id);
CREATE INDEX idx_accounts_status ON accounts(status);
```

### transactions
```sql
CREATE TABLE transactions (
  transaction_id BIGINT PRIMARY KEY,
  account_id BIGINT NOT NULL REFERENCES accounts,
  transaction_date TIMESTAMP NOT NULL,
  transaction_type VARCHAR(20),  -- DEBIT, CREDIT, TRANSFER
  merchant_id BIGINT,
  merchant_name VARCHAR(256),
  merchant_category VARCHAR(50),  -- MCC code
  merchant_country VARCHAR(3),
  amount NUMERIC(19,4) NOT NULL,
  currency_code VARCHAR(3),
  status VARCHAR(20),  -- PENDING, CONFIRMED, SETTLED, FAILED
  fraud_score NUMERIC(5,2),  -- 0-100
  fraud_flag BOOLEAN DEFAULT FALSE,
  risk_score NUMERIC(5,2),
  description TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  CONSTRAINT check_transaction_status IN ('PENDING', 'CONFIRMED', 'SETTLED', 'FAILED'),
  CONSTRAINT check_amount_positive CHECK (amount > 0)
);

-- Partitioned by transaction_date for performance
CREATE INDEX idx_transactions_account_date ON transactions(account_id, transaction_date DESC);
CREATE INDEX idx_transactions_fraud_flag ON transactions(fraud_flag) WHERE fraud_flag = TRUE;
CREATE INDEX idx_transactions_merchant ON transactions(merchant_category, transaction_date);
```

### fraud_alerts
```sql
CREATE TABLE fraud_alerts (
  alert_id BIGSERIAL PRIMARY KEY,
  transaction_id BIGINT NOT NULL REFERENCES transactions,
  alert_type VARCHAR(50),  -- VELOCITY, ANOMALY, GEOGRAPHIC, BEHAVIORAL
  alert_severity VARCHAR(20),  -- LOW, MEDIUM, HIGH, CRITICAL
  alert_message TEXT,
  triggered_rule_id INTEGER,
  investigation_status VARCHAR(20),  -- NEW, INVESTIGATING, RESOLVED, FALSE_POSITIVE
  investigated_by VARCHAR(256),
  investigation_notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_fraud_alerts_status ON fraud_alerts(investigation_status);
CREATE INDEX idx_fraud_alerts_created ON fraud_alerts(created_at DESC);
```

### risk_profiles
```sql
CREATE TABLE risk_profiles (
  profile_id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL REFERENCES customers,
  profile_date DATE NOT NULL,
  credit_score INTEGER,  -- 300-850
  default_probability NUMERIC(5,4),  -- 0-1
  liquidity_risk NUMERIC(5,2),  -- 0-100
  concentration_risk NUMERIC(5,2),
  counterparty_risk NUMERIC(5,2),
  overall_risk_score NUMERIC(5,2),
  review_status VARCHAR(20),  -- APPROVED, REVIEW_NEEDED, REJECTED
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(customer_id, profile_date)
);
```

### audit_log
```sql
CREATE TABLE audit_log (
  log_id BIGSERIAL PRIMARY KEY,
  table_name VARCHAR(256) NOT NULL,
  operation VARCHAR(10),  -- INSERT, UPDATE, DELETE
  record_id BIGINT,
  old_values JSONB,
  new_values JSONB,
  modified_by VARCHAR(256),
  modified_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  change_hash VARCHAR(256),  -- SHA256 hash for integrity
  CONSTRAINT check_operation IN ('INSERT', 'UPDATE', 'DELETE')
);

CREATE INDEX idx_audit_log_table_date ON audit_log(table_name, modified_at DESC);
```

## Constraints & Business Rules

1. **Referential Integrity**: All foreign keys have cascade rules
2. **Data Validation**: Check constraints on amounts (> 0) and statuses
3. **Temporal Consistency**: opened_date must be before closed_date
4. **Immutability**: Audit logs cannot be deleted
5. **Account Status**: Cannot transact on CLOSED or SUSPENDED accounts
