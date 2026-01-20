# dbt Data Pipeline - Take-Home Challenge

## Position: Data Engineer

### Challenge Duration: 2-4 hours

---

## Overview

Welcome to the Data Engineering take-home challenge! This exercise evaluates your ability to build data transformation pipelines using **[dbt (data build tool)](https://docs.getdbt.com)** based on source-target mapping specifications.

You will transform raw banking transaction data into a reporting-ready data product that provides customer-level monthly transaction summaries.

---

## Prerequisites

Before starting, ensure you have the following installed:

- **Python 3.9+** 
- **uv** (fast Python package manager) - [Install uv](https://docs.astral.sh/uv/getting-started/installation/)

That's it! This project uses **DuckDB** as the database, which requires no separate installation.

### Installing uv

```bash
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Or with pip
pip install uv
```

---

## Quick Start

### 1. Clone/Download/Fork this repository

You can choose to just clone or download this repository content to your local machine directly or fork it to your own account to later share the 
solution. In any case, once you have the code locally:

```bash
cd dbt-banking-challenge
```

### 2. Install dependencies (uv creates virtualenv automatically)

```bash
uv sync
```

### 3. Verify dbt installation

```bash
uv run dbt --version
```

### 4. Test database connection

```bash
cd bank_dbt
uv run dbt debug
```

You should see "All checks passed!" if everything is configured correctly.

---

## Project Structure

```
dbt-banking-challenge/
├── README.md                    # This file
├── pyproject.toml               # Python dependencies (uv)
├── target_model_spec.md         # ⭐ SOURCE-TARGET MAPPING SPEC
├── data/                        # Raw data files
│   ├── customers.csv            # Customer master data
│   ├── accounts.csv             # Account information
│   └── transactions.csv         # Transaction records
├── bank_dbt/                    # dbt project folder
│   ├── dbt_project.yml          # dbt project configuration
│   ├── profiles.yml             # Database connection (DuckDB)
│   ├── seeds/                   # ⭐ Copy CSVs here
│   │   └── .gitkeep
│   ├── models/
│   │   ├── staging/             # ⭐ Create staging models here
│   │   │   └── .gitkeep
│   │   └── marts/               # ⭐ Create mart models here
│   │       └── .gitkeep
│   └── snapshots/               # ⭐ Bonus: SCD snapshots here
│       └── .gitkeep
└── solution/                    # Reference solution (DO NOT PEEK!)
    └── ...
```

---

## Your Tasks

Complete the following tasks based on the specifications in **`target_model_spec.md`**:

### Task 1: Set Up Seeds (10 min)
- Copy the CSV files from `data/` to `bank_dbt/seeds/`
- Create `seeds/schema.yml` to document the seed tables
- Run `uv run dbt seed` to load them into DuckDB

### Task 2: Build Staging Models (45 min)
- Create `stg_customers.sql` following the mapping spec (use `{{ ref('customers') }}`)
- Create `stg_accounts.sql` following the mapping spec (use `{{ ref('accounts') }}`)
- Create `stg_transactions.sql` following the mapping spec (use `{{ ref('transactions') }}`)
- Apply transformation rules as specified

### Task 3: Build Mart Model (45 min)
- Create `customer_monthly_summary.sql` in `models/marts/`
- Implement all columns as specified
- Follow join logic and business rules

### Task 4: Add dbt Tests (30 min)
- Add `schema.yml` files with tests:
  - `not_null` on all ID columns
  - `unique` on all ID columns
  - `accepted_values` for `transaction_type`
- Run `uv run dbt test` and ensure all pass

### Task 5: Documentation (20 min)
- Add descriptions to all models in YAML files
- Generate docs: `uv run dbt docs generate`
- View docs: `uv run dbt docs serve`
- Take a screenshot of the DAG

### Bonus Task: Snapshot (30 min)
- Create a dbt snapshot for tracking customer city changes
- Use `timestamp` strategy with `updated_at` column
- Document the snapshot

---

## Useful Commands

```bash
# Navigate to dbt project
cd bank_dbt

# Load seed data
uv run dbt seed

# Run all models
uv run dbt run

# Run tests
uv run dbt test

# Build everything (seed + run + test)
uv run dbt build

# Generate and serve documentation
uv run dbt docs generate
uv run dbt docs serve

# Run snapshots
uv run dbt snapshot

# Run specific model
uv run dbt run --select stg_customers

# Run model and all downstream dependencies
uv run dbt run --select stg_customers+
```

---

## DuckDB Tips

This project uses DuckDB, a lightweight analytical database. Some syntax notes:

- Date formatting: `STRFTIME('%Y-%m', date_column)`
- String concatenation: `first_name || ' ' || last_name`
- Boolean from condition: `CASE WHEN status = 'ACTIVE' THEN true ELSE false END`
- The database file is created automatically at `bank_dbt/dev.duckdb`

---

## Evaluation Criteria

Your submission will be evaluated on:

| Criteria | Weight | Description |
|----------|--------|-------------|
| **Correctness** | 30% | Models produce expected output per mapping spec |
| **Code Quality** | 25% | Clean SQL, proper CTEs, readable formatting |
| **Testing** | 20% | Comprehensive tests, all tests pass |
| **Documentation** | 15% | Clear model descriptions, DAG explanation |
| **Bonus** | 10% | Snapshot implementation, extra tests |

---

## Submission

Please submit:

1. Your completed `bank_dbt/` project folder (you can share it as a link if you fork the repository)
2. A screenshot of your DAG from dbt docs
3. A brief write-up (1-2 paragraphs) explaining:
   - Any assumptions you made
   - Challenges encountered and how you solved them
   - Ideas for improving the pipeline

---

## Questions?

Don't hesitate to contact us if you have questions about the requirements.

**Note**: This is a take-home challenge. Please complete it independently without help from other people.

---

Good luck!
