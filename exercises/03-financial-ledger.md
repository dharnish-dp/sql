# Exercise 03 — Financial Ledger

## Overview
Build a double-entry accounting ledger. Every financial transaction must
balance (debits = credits). Practice transactions, constraints, running balances,
and window functions.

## Skills Practiced
Transactions, ACID, window functions (running totals), constraints, triggers

---

## Step 1: Build the Schema

```sql
CREATE TABLE accounts (
    id       SERIAL PRIMARY KEY,
    name     TEXT NOT NULL UNIQUE,
    type     TEXT NOT NULL CHECK (type IN ('asset', 'liability', 'equity', 'revenue', 'expense')),
    balance  NUMERIC(15, 4) NOT NULL DEFAULT 0,
    currency TEXT NOT NULL DEFAULT 'USD'
);

CREATE TABLE ledger_entries (
    id           BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    transaction_id UUID NOT NULL,       -- groups entries that form one transaction
    account_id   INT NOT NULL REFERENCES accounts(id),
    amount       NUMERIC(15, 4) NOT NULL,  -- positive = debit, negative = credit
    description  TEXT,
    entry_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Index for fast balance calculations
CREATE INDEX idx_ledger_account_date ON ledger_entries(account_id, entry_date);

-- Sample accounts
INSERT INTO accounts (name, type) VALUES
    ('Cash',            'asset'),
    ('Accounts Receivable', 'asset'),
    ('Inventory',       'asset'),
    ('Accounts Payable','liability'),
    ('Revenue',         'revenue'),
    ('Cost of Goods',   'expense'),
    ('Payroll',         'expense'),
    ('Owner Equity',    'equity');

-- Sample transactions
-- Rule: for each transaction_id, SUM(amount) must = 0 (double-entry)
INSERT INTO ledger_entries (transaction_id, account_id, amount, description, entry_date) VALUES
    -- Sale: +Revenue, +Cash
    (gen_random_uuid(), 5, -1000, 'Product sale Jan 5',  '2024-01-05'),
    (gen_random_uuid(), 1,  1000, 'Product sale Jan 5',  '2024-01-05'),
    -- (note: each gen_random_uuid() generates a NEW uuid — in real use, you'd share one per transaction)
    -- For this exercise, use explicit UUIDs for grouped transactions:
    ('aaaaaaaa-0001-0001-0001-000000000001', 5, -1500, 'Sale Jan 10', '2024-01-10'),
    ('aaaaaaaa-0001-0001-0001-000000000001', 1,  1500, 'Sale Jan 10', '2024-01-10'),
    ('aaaaaaaa-0002-0002-0002-000000000002', 7,  5000, 'Payroll Jan 15', '2024-01-15'),
    ('aaaaaaaa-0002-0002-0002-000000000002', 1, -5000, 'Payroll Jan 15', '2024-01-15'),
    ('aaaaaaaa-0003-0003-0003-000000000003', 5, -2000, 'Sale Feb 3',  '2024-02-03'),
    ('aaaaaaaa-0003-0003-0003-000000000003', 1,  2000, 'Sale Feb 3',  '2024-02-03'),
    ('aaaaaaaa-0004-0004-0004-000000000004', 7,  5500, 'Payroll Feb 15', '2024-02-15'),
    ('aaaaaaaa-0004-0004-0004-000000000004', 1, -5500, 'Payroll Feb 15', '2024-02-15'),
    ('aaaaaaaa-0005-0005-0005-000000000005', 5, -3000, 'Sale Mar 1',  '2024-03-01'),
    ('aaaaaaaa-0005-0005-0005-000000000005', 1,  3000, 'Sale Mar 1',  '2024-03-01');
```

---

## Queries

**Q1.** What is the current balance of each account?
Sum all ledger entries per account. Show account name, type, and balance.

**Q2.** Show the running balance of the Cash account over time.
Show entry date, description, amount, and cumulative balance after each entry.

**Q3.** Verify the double-entry rule: for each `transaction_id`, the sum of
all entries should be 0. Find any transactions where this rule is violated.

**Q4.** What is the total revenue, total expenses, and net profit for each month?

**Q5.** Show the monthly balance of the Cash account at the END of each month
(the balance after all entries in that month).

---

## Constraints Challenge

**Q6.** Write a check constraint (or trigger) that prevents a ledger entry
from being inserted if it would make the related transaction_id unbalanced.
(Hint: this is hard with just CHECK — use a trigger that verifies the transaction
balance after every INSERT.)

**Q7.** Using a transaction, record a new sale of $2,500:
- Revenue goes up (credit Revenue: -2500)
- Cash goes up (debit Cash: +2500)
Make it atomic — either both entries happen or neither.

---

## Answer Hints

<details>
<summary>Q2 Running Balance</summary>

```sql
SELECT
    entry_date,
    description,
    amount,
    SUM(amount) OVER (ORDER BY entry_date, id) AS running_balance
FROM ledger_entries
WHERE account_id = 1  -- Cash
ORDER BY entry_date, id;
```
</details>

<details>
<summary>Q3 Balance Check</summary>

```sql
SELECT transaction_id, SUM(amount) AS net
FROM ledger_entries
GROUP BY transaction_id
HAVING SUM(amount) != 0;
-- Should return 0 rows if all transactions are balanced
```
</details>

<details>
<summary>Q7 Atomic Transaction</summary>

```sql
BEGIN;
DO $$
DECLARE
    txn_id UUID := gen_random_uuid();
BEGIN
    INSERT INTO ledger_entries (transaction_id, account_id, amount, description)
    VALUES (txn_id, 5, -2500, 'Sale March 20');  -- Credit Revenue
    
    INSERT INTO ledger_entries (transaction_id, account_id, amount, description)
    VALUES (txn_id, 1,  2500, 'Sale March 20');  -- Debit Cash
END;
$$;
COMMIT;
```
</details>
