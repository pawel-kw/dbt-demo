# Target Model Specification

## Overview

This document defines the source-to-target mapping for transforming raw banking data into reporting-ready models. The candidate should implement dbt models following these specifications exactly.

---

## Source Data (Seeds)

The raw data is loaded as dbt seeds from CSV files. Use `{{ ref('seed_name') }}` to reference them in your models.

### 1. `customers` (seed: customers.csv)

| Column Name     | Data Type | Description                          |
|-----------------|-----------|--------------------------------------|
| customer_id     | STRING    | Unique customer identifier           |
| first_name      | STRING    | Customer's first name                |
| last_name       | STRING    | Customer's last name                 |
| date_of_birth   | DATE      | Customer's date of birth             |
| city            | STRING    | Customer's city of residence         |
| state           | STRING    | Customer's state of residence        |
| country         | STRING    | Customer's country of residence      |
| created_at      | TIMESTAMP | Record creation timestamp            |
| updated_at      | TIMESTAMP | Record last update timestamp         |

### 2. `accounts` (seed: accounts.csv)

| Column Name       | Data Type | Description                        |
|-------------------|-----------|------------------------------------|
| account_id        | STRING    | Unique account identifier          |
| customer_id       | STRING    | Foreign key to customers           |
| account_type      | STRING    | Type of account (CHECKING/SAVINGS) |
| open_date         | DATE      | Date account was opened            |
| status            | STRING    | Account status (ACTIVE/CLOSED)     |
| initial_balance   | DECIMAL   | Initial balance when opened        |
| currency          | STRING    | Account currency (e.g., USD)       |

### 3. `transactions` (seed: transactions.csv)

| Column Name       | Data Type | Description                              |
|-------------------|-----------|------------------------------------------|
| transaction_id    | STRING    | Unique transaction identifier            |
| account_id        | STRING    | Foreign key to accounts                  |
| amount            | DECIMAL   | Transaction amount                       |
| transaction_date  | TIMESTAMP | Date and time of transaction             |
| transaction_type  | STRING    | Type: DEBIT, CREDIT, or TRANSFER         |
| merchant_name     | STRING    | Name of merchant (if applicable)         |
| category          | STRING    | Transaction category                     |
| description       | STRING    | Transaction description                  |

---

## Target Models

### Staging Models

The staging layer should clean and standardize source data with consistent naming conventions.

#### `stg_customers`

| Target Column       | Source Column          | Transformation Rule                                           |
|---------------------|------------------------|---------------------------------------------------------------|
| customer_id         | customer_id            | Direct mapping, cast to VARCHAR                               |
| full_name           | first_name, last_name  | Concatenate: `first_name || ' ' || last_name`                 |
| first_name          | first_name             | Direct mapping, TRIM whitespace                               |
| last_name           | last_name              | Direct mapping, TRIM whitespace                               |
| date_of_birth       | date_of_birth          | Cast to DATE                                                  |
| city                | city                   | Direct mapping, UPPER case                                    |
| state               | state                  | Direct mapping, UPPER case                                    |
| country             | country                | Direct mapping, UPPER case                                    |
| created_at          | created_at             | Cast to TIMESTAMP                                             |
| updated_at          | updated_at             | Cast to TIMESTAMP                                             |

#### `stg_accounts`

| Target Column       | Source Column     | Transformation Rule                                            |
|---------------------|-------------------|----------------------------------------------------------------|
| account_id          | account_id        | Direct mapping, cast to VARCHAR                                |
| customer_id         | customer_id       | Direct mapping, cast to VARCHAR                                |
| account_type        | account_type      | Direct mapping, LOWER case                                     |
| open_date           | open_date         | Cast to DATE                                                   |
| is_active           | status            | Boolean: `status = 'ACTIVE'`                                   |
| initial_balance     | initial_balance   | Cast to DECIMAL(18,2)                                          |
| currency            | currency          | Direct mapping, UPPER case                                     |

#### `stg_transactions`

| Target Column       | Source Column     | Transformation Rule                                            |
|---------------------|-------------------|----------------------------------------------------------------|
| transaction_id      | transaction_id    | Direct mapping, cast to VARCHAR                                |
| account_id          | account_id        | Direct mapping, cast to VARCHAR                                |
| amount              | amount            | Cast to DECIMAL(18,2)                                          |
| transaction_date    | transaction_date  | Cast to TIMESTAMP                                              |
| transaction_type    | transaction_type  | Direct mapping, LOWER case                                     |
| merchant_name       | merchant_name     | Direct mapping, COALESCE with 'Unknown' if NULL                |
| category            | category          | Direct mapping, LOWER case                                     |
| description         | description       | Direct mapping, TRIM whitespace                                |

---

### Mart Model: `customer_monthly_summary`

This is the final reporting model that aggregates transaction data per customer per month.

| Target Column           | Source Tables                        | Transformation Rule                                                                                       |
|-------------------------|--------------------------------------|-----------------------------------------------------------------------------------------------------------|
| summary_id              | Generated                            | Surrogate key: `customer_id || '-' || year_month`                                                         |
| customer_id             | stg_customers                        | Direct mapping                                                                                            |
| customer_name           | stg_customers                        | `full_name` from stg_customers                                                                            |
| year_month              | stg_transactions                     | Format: `YYYY-MM` extracted from transaction_date                                                         |
| total_transactions      | stg_transactions                     | COUNT of all transactions for customer in month                                                           |
| total_debit_amount      | stg_transactions                     | SUM of amount WHERE transaction_type = 'debit'                                                            |
| total_credit_amount     | stg_transactions                     | SUM of amount WHERE transaction_type = 'credit'                                                           |
| total_transfer_amount   | stg_transactions                     | SUM of amount WHERE transaction_type = 'transfer'                                                         |
| net_flow                | Calculated                           | `total_credit_amount - total_debit_amount`                                                                |
| avg_transaction_amount  | stg_transactions                     | AVG of amount for all transactions                                                                        |
| largest_transaction     | stg_transactions                     | MAX of amount for all transactions                                                                        |
| debit_count             | stg_transactions                     | COUNT of transactions WHERE transaction_type = 'debit'                                                    |
| credit_count            | stg_transactions                     | COUNT of transactions WHERE transaction_type = 'credit'                                                   |
| transfer_count          | stg_transactions                     | COUNT of transactions WHERE transaction_type = 'transfer'                                                 |
| customer_city           | stg_customers                        | `city` from stg_customers                                                                                 |
| customer_state          | stg_customers                        | `state` from stg_customers                                                                                |

#### Business Rules for `customer_monthly_summary`

1. **Join Logic**: 
   - Join `stg_transactions` to `stg_accounts` on `account_id`
   - Join `stg_accounts` to `stg_customers` on `customer_id`
   - Only include transactions from ACTIVE accounts (`is_active = true`)

2. **Aggregation Level**: 
   - Group by `customer_id` and `year_month`
   - Each row represents one customer's activity for one calendar month

3. **Null Handling**:
   - Use COALESCE to replace NULL amounts with 0
   - Customers with no transactions in a month should NOT appear in the output

4. **Date Extraction**:
   - Extract year and month from `transaction_date` using: `STRFTIME('%Y-%m', transaction_date)` for DuckDB

---

## Data Quality Tests

### Required Tests

| Model                | Column              | Test Type         | Details                                    |
|----------------------|---------------------|-------------------|--------------------------------------------|
| stg_customers        | customer_id         | unique            | No duplicate customer IDs                  |
| stg_customers        | customer_id         | not_null          | All customers must have an ID              |
| stg_accounts         | account_id          | unique            | No duplicate account IDs                   |
| stg_accounts         | account_id          | not_null          | All accounts must have an ID               |
| stg_accounts         | customer_id         | not_null          | All accounts must belong to a customer     |
| stg_transactions     | transaction_id      | unique            | No duplicate transaction IDs               |
| stg_transactions     | transaction_id      | not_null          | All transactions must have an ID           |
| stg_transactions     | transaction_type    | accepted_values   | Must be: 'debit', 'credit', 'transfer'     |
| customer_monthly_summary | summary_id      | unique            | One row per customer per month             |
| customer_monthly_summary | summary_id      | not_null          | All summaries must have an ID              |

---

## DAG (Directed Acyclic Graph)

```
seeds/customers.csv ──► stg_customers ──────────┐
                                                │
seeds/accounts.csv ──► stg_accounts ────────────┼──► customer_monthly_summary
                                                │
seeds/transactions.csv ──► stg_transactions ────┘
```

### Dependency Explanation

1. **Seeds** are the raw CSV files loaded into the database
2. **Staging models** depend on seeds and apply cleaning/transformation rules
3. **Mart model** depends on all three staging models and joins them together

---

## Bonus: Slowly Changing Dimension (SCD Type 2)

For candidates attempting the bonus challenge:

### `snapshot_customers`

Track changes to customer city over time using dbt snapshots.

| Column            | Description                                      |
|-------------------|--------------------------------------------------|
| customer_id       | Customer identifier                              |
| full_name         | Customer's full name                             |
| city              | City value at snapshot time                      |
| state             | State value at snapshot time                     |
| dbt_valid_from    | Timestamp when this record became valid          |
| dbt_valid_to      | Timestamp when this record became invalid (NULL if current) |
| dbt_updated_at    | Timestamp of the source record update            |

**Snapshot Strategy**: Use `timestamp` strategy with `updated_at` as the check column.

---

## Expected Output

After implementing all models, running `dbt build` should:

1. Load 3 seed files
2. Create 3 staging models
3. Create 1 mart model
4. Pass all data quality tests
5. Generate documentation accessible via `dbt docs generate && dbt docs serve`
