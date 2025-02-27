# ACID (Atomicity, Consistency, Isolation, Durability)

ACID is an acronym that stands for Atomicity, Consistency, Isolation, and Durability. It is a set of operations that are guaranteed to maintain data integrity by adhering to the ACID properties


## Atomicity. 
In simple words **ALL OR NOTHING**
All the operations should complete in one go or if any one fails then all should fail.

For example:
If A is making transaction to B's Bank account.

Here are possible steps:
1. Verify Credentials.
2. Verify B's Bank account details.
3. Check if A Has amount that needs to be deducted 
4. Deduct from the A's Bank Account.
5. Credit to B's Bank Account.
6. Update A's Bank Account.

In above steps, especially if Step 5 fails then we need to roll back 4th step as well. 

We can achieve this using `transaction`, `abort`, `commit`.

## Consistency.
In simple words **Follow the Rules**
Database remains in a valid state before and after a transaction. Which means it should preserve the integrity constrains.

Example: 

```
Imagine we have a banking system where:
	*	The total balance across all accounts should remain constant.
	*	Negative balances are not allowed.

Before Transaction:
	*	Account A: ₹5000
	*	Account B: ₹3000
	*	Total Balance: ₹8000

Transaction:
	1.	Deduct ₹2000 from Account A.
	2.	Add ₹2000 to Account B.
	3.	Ensure the total balance remains ₹8000.

Valid Scenario:
    1. Account A: 3000.
    2. Account B: 5000.
    3. Total Balance: 8000

Invalid Scenario:
    1. Account A: 3000.
    2. Account B: 2000.
    3. Total Balance: 5000

    If Amount is deducted from Account A and not credited to Account B then it should roll back to avoid inconsistency.
```

## Isolation.
In simple words **No Interference**
It means that multiple transactions can occur concurrently without leading to the inconsistency of the database state. Transactions occur independently without interference.

*  Multiple transactions running at the same time do not affect each other.
*  Each transaction is executed as if it is the only transaction running — even if others are happening simultaneously.

Consider there is only one ticket present for a concert. 
Until and unless his transaction is completed other can't proceed. Only if his transaction is rolled back or failed then other can proceed for purchasing.

There are 4 levels of Isolation
1. Dirty Read (Read uncommitted)- Transactions can see uncommitted changes from others.
2. Read Committed - Transactions can see committed changes only.
3. Repeatable Read - Transactions "lock" data they read, so others can’t modify it.
4. Serializable - Transactions act as if they’re happening one after another (total isolation).

## Durability
In simple words **Permanent Once Done**
Once a transaction is confirmed, it stays saved even if the system crashes.



SQL Example.
```SQL
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance DECIMAL(10,2)
);

INSERT INTO accounts VALUES (1, 1000), (2, 500);
```


## Atomicity
```SQL
START TRANSACTION; -- Begin the transaction

-- Deduct $150 from Account 1
UPDATE accounts SET balance = balance - 150 WHERE id = 1;

-- Add $150 to Account 2
UPDATE accounts SET balance = balance + 150 WHERE id = 2;

-- If both succeed:
COMMIT; -- Finalize the changes

-- If any step fails (e.g., insufficient funds):
ROLLBACK; -- Undo all changes
```

**What MySQL Does:**

If the server crashes after UPDATE but before COMMIT, MySQL automatically rolls back the transaction.

No partial transfers—either both accounts update, or neither does.

## Consistency
**Add a Constraint:**
```SQL
ALTER TABLE accounts 
ADD CONSTRAINT balance_non_negative CHECK (balance >= 0);
```

```SQL
START TRANSACTION;

-- Attempt to deduct $1200 (balance would become -200)
UPDATE accounts SET balance = balance - 1200 WHERE id = 1;

-- MySQL rejects this due to the CHECK constraint:
-- Error: Check constraint 'balance_non_negative' is violated.

ROLLBACK; -- No changes are saved
```

## Isolation

Checking Isolation level-
```SQL
SELECT @@transaction_isolation; -- Default in MySQL is REPEATABLE-READ
```


Updating/Setting Isolation level
```SQL
SET SESSION TRANSACTION ISOLATION LEVEL <LEVEL>; -- e.g., READ UNCOMMITTED
```

### Read UnCommitted
Consider 2 session for same user

Session A - Transfers $100
```SQL
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
UPDATE bank_accounts SET balance = balance - 100 WHERE id = 1; -- New balance: $900 (uncommitted)
```

SESSION B - Checking Balance.
```SQL
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT balance FROM bank_accounts WHERE id = 1; -- Sees $900 (dirty read!)
```

SESSION A - ROLLBACKS AND RECHECKS BALANCE
```SQL
ROLLBACK; -- Cancels the transfer
SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT balance FROM bank_accounts WHERE id = 1; -- Sees $1000
```


### Read Committed.
SESSION 1 - Update $100.
```SQL
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
UPDATE bank_accounts SET balance = balance - 100 WHERE id = 1; -- Balance: $900 (uncommitted)
```
SESSION 2 - Check Balance.
```SQL
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT balance FROM bank_accounts WHERE id = 1; -- Still sees $1000 (no dirty read)
```

SESSION 1 - Commit
```SQL
COMMIT;
```

SESSION 2 - Check balance
```SQL
SELECT balance FROM bank_accounts WHERE id = 1; -- Now sees $900 (non-repeatable read)
```

### Repeatable Read (MySQL Default)

Session A - Check Balance
```SQL
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
START TRANSACTION;
SELECT balance FROM accounts WHERE id = 1; -- Sees $1000
```

Session B - Transfers $200
```SQL
START TRANSACTION;
UPDATE accounts SET balance = balance - 200 WHERE id = 1;
COMMIT; -- Balance is now $800
```


Session A - Rechecks Balance
```SQL
SELECT balance FROM accounts WHERE id = 1; -- Still sees $1000 (snapshot isolation)
COMMIT; -- After commit, it will see $800
```

### SERIALIZABLE
Transactions are fully isolated using locks. No concurrency issues, but slower.

```SQL
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
SELECT balance FROM bank_accounts WHERE id = 1 FOR UPDATE; -- Locks the row
```

Now Until committed we cant update that row

Session B 
```SQL
SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE;
START TRANSACTION;
UPDATE bank_accounts SET balance = balance - 100 WHERE id = 1; -- Waits for Session A's lock!
```

```SQL
COMMIT;
```