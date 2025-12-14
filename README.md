# 📐 Transaction System Design Principles

## Introduction

Why design this ? Well, it's the core of most system, right ?

Designing effective transactional systems is crucial for any application 
that manages changes in data, such as inventory, finance, or order processing. 
The following principles and patterns provide a practical foundation 
for building systems that are reliable, auditable, and scalable, regardless of the domain.

---

## 1. Main Table and Transaction Tables

A robust transaction system separates core entity data from transactional records. The main 
table holds essential, relatively static information about each entity, 
such as items or accounts, while transaction tables record every event that changes 
an entity’s state. This separation ensures efficient lookups and a clear audit trail.

By keeping the main table lean and allowing the transaction table to grow independently, 
you maintain both performance and traceability. The transaction table captures a 
detailed log of all events, making it possible to 
reconstruct the full history of any entity’s state.

To add some nuance, in strict ledger-based designs the transaction 
table is treated as the source of truth, while the main table represents 
a derived or cached state maintained primarily for performance.

**Illustration:**
```
[Main Table: Items]
+----+----------+---------+
| ID | Name     | Balance |
+----+----------+---------+
| 1  | Widget A |   100   |
+----+----------+---------+

[Transaction Table: IN/OUT]
+----+---------+--------+--------+--------+
| ID | Item_ID | Type   | Amount | Date   |
+----+---------+--------+--------+--------+
| 1  |   1     | IN     |  50    | ...    |
+----+---------+--------+--------+--------+
```

---

## 2. Real-Time vs. Computed Balances

Determining the current state of an entity can be done by real-time computation 
or by maintaining a running balance. 

Real-time computation aggregates all relevant transaction records 
on demand, ensuring up-to-date results but 
potentially slowing down as data grows. Running balance stores the current 
state in the main table and updates it with each transaction, 
offering fast queries but requiring careful consistency management.

Choosing between these approaches depends on your system’s scale and 
performance needs. Real-time computation is often sufficient for 
small to medium datasets, while running balance is preferable 
for larger systems or where performance is critical.

So as for me, given I know the system will not grow that big, I pick
on-the-fly approach first.

**Illustration:**
```
-- Real-time computation
SELECT SUM(CASE WHEN Type='IN' THEN Amount ELSE -Amount END)
FROM Transaction
WHERE Item_ID = 1;

-- Running balance
[Main Table: Items]
+----+----------+---------+
| ID | Name     | Balance |
+----+----------+---------+
| 1  | Widget A |   100   |
+----+----------+---------+
```

---

## 3. Handling Drafts and Final States

Not all transactions are immediately finalized; some may be drafts, 
pending approval, or cancelled. The most common approach is to 
include a status field in the transaction table, indicating 
whether a transaction is a draft, pending, finalized, 
or cancelled. Only finalized transactions are included in balance computations.

In more complex workflows like too many statuses, approvals and stages, 
separate tables for drafts and finalized transactions
can be used, especially if drafts are 
frequently edited or deleted. This approach prevents 
clutter in the main transaction table and simplifies 
queries on finalized transactions. This is discussed further in Section 8.

**Illustration:**
```
[Transaction Table]
+----+---------+--------+--------+--------+--------+
| ID | Item_ID | Type   | Amount | Status | Date   |
+----+---------+--------+--------+--------+--------+
| 1  |   1     | OUT    |  10    | Draft  | ...    |
| 2  |   1     | OUT    |  10    | Final  | ...    |
+----+---------+--------+--------+--------+--------+
```

---

## 4. Avoiding Race Conditions

Race conditions can occur when multiple processes update the 
same data simultaneously, leading to inconsistencies. The primary 
defense is using database transactions, which ensure that related 
operations are applied atomically. Row-level locking, such as 
`SELECT ... FOR UPDATE`, prevents other processes from modifying 
the same data until the transaction is complete.

However, heavy concurrent access can lead to deadlocks or transaction 
failures, so applications should be prepared to detect these cases and 
retry operations when failures are caused by temporary concurrency issues 
rather than logical errors.

Optimistic locking, using version numbers or timestamps, is an alternative 
approach that works well when conflicts are infrequent. By applying these 
techniques, transaction systems remain reliable and accurate even under load.

When using Laravel, we know it's a matured framework. Laravel 
lets you manage database transactions in code, without writing
transaction statements directly in SQL.

**Illustration:**
```
BEGIN TRANSACTION
  SELECT Balance FROM Items WHERE ID = 1 FOR UPDATE;
  UPDATE Items SET Balance = Balance - 10 WHERE ID = 1;
COMMIT;
```

---

## 5. Preventing Duplicate Transactions

Duplicate transactions can cause serious errors, such as double 
billing or overstocking. 

Preventing duplicates is achieved by using 
unique constraints or idempotency keys, ensuring each transaction is 
unique. Validation logic in the application layer can also help by checking 
for existing records with the same attributes before processing a transaction.

Audit trails and correction logs are important for rectifying errors 
after the fact. If a duplicate is detected, the system should allow 
for correction, either by voiding the duplicate or adjusting the balance accordingly.

**Illustration:**
```
[Transaction Table]
+----+---------+--------+--------+----------+
| ID | Item_ID | Type   | Amount | Ref_No   |
+----+---------+--------+--------+----------+
| 1  |   1     | IN     |  50    | ABC123   |
+----+---------+--------+--------+----------+
```

---

## 6. Entity Identification

Stable, unique identifiers are essential for robust transaction systems. 
Auto-incrementing IDs provide a reliable way to uniquely identify each 
entity, simplifying database design and ensuring relationships remain 
intact even if business codes change. User-facing codes should be 
stored as unique fields, but not used as primary keys.

All relationships should reference the ID field, not the code, ensuring 
data integrity and supporting best practices in database normalization.

This is well done by Laravel as it has the default auto-incrementing IDs.

**Illustration:**
```
[Items]
+----+----------+----------+
| ID | Code     | Name     |
+----+----------+----------+
| 1  | WGT-A    | Widget A |
+----+----------+----------+
```

---

## 7. Migrating from Code-Based to ID-Based References

Legacy systems and even modern systems ( and even myself ) 
might have the tendency to use business codes as primary references, but migrating 
to auto-incrementing IDs improves data integrity and maintainability. 
Start by adding an ID column, update foreign key relationships, 
and gradually refactor queries and application logic to use IDs instead of codes.

During the transition, support both code-based and ID-based 
references, ensuring all new relationships use the ID. Over time, 
phase out the use of business codes for internal references, 
thoroughly testing and validating data throughout the process.

**Illustration:**
```
-- Old query
SELECT * FROM Transactions WHERE Item_Code = 'WGT-A';

-- New query
SELECT * FROM Transactions WHERE Item_ID = 1;
```

---

## 8. When to Use Separate Draft / Final Tables

While a single transaction table with a status field is usually sufficient, 
separate tables for drafts and finalized transactions are warranted in 
systems with complex workflows, high draft volumes, or strict audit 
requirements. This separation prevents clutter, improves 
performance, and supports advanced features like 
workflow automation and multi-stage approvals.

Records are moved from the draft table to the final table upon approval, 
providing greater control and flexibility in managing transaction lifecycles.
Or it may be simply copied to the Final table and keep the draft version too.

**Illustration:**
```
[Draft_Transactions]
[Final_Transactions]
-- Move record from Draft to Final upon approval
```

---

## 9. General Transaction Workflow Patterns

Transaction systems must manage the lifecycle of each transaction, 
including status transitions, approvals, and audit trails. 
Status fields track the state of each transaction, and the system 
enforces rules for transitioning between 
states, such as requiring approval before finalization.

Audit trails log every change, preserving the full history of 
each transaction. Event sourcing, where every change is recorded 
as an immutable event, provides maximum auditability 
and supports features like undo/redo and historical analysis.

**Illustration:**
```
[Transaction Status Flow]
Draft → Pending Approval → Final → Cancelled
```

---

## Those Not Covered

Of course, some common concerns were intentionally not discussed in detail. 
Filtering, sorting, pagination and indexing are standard practices for handling 
large datasets and apply broadly beyond transactional system design.

---

## 📝 Summary and Key Takeaways

Designing transactional systems requires careful planning and proven 
principles. Separating core and transaction data, choosing appropriate 
balance strategies, handling drafts correctly, and preventing race 
conditions and duplicates ensure data integrity, performance, 
and auditability.

**Recap of Key Principles:**

1. **Main and Transaction Tables:** separate core data from events
2. **Balance Computation:** choose real-time or running balance wisely
3. **Drafts and Final States:** use statuses or separate tables for workflow
4. **Race Conditions:** prevent with transactions and locking
5. **Duplicate Prevention:** enforce uniqueness and idempotency
6. **Entity Identification:** use stable, auto-incrementing IDs
7. **Migrating References:** move from code-based to ID-based links
8. **Separate Tables When Needed:** for complex or high-volume workflows
9. **Workflow Patterns:** support status transitions and audit trails
