# tamper-evident-inventory-architecture

[العربية](README.md)

**A device enters the warehouse → a numbered document with no gaps. The database itself refuses to change its ledger row.**

> **This system runs in production at my employer, and the code belongs to them. This repository documents the architecture, the engineering decisions, and how it was built.**

An IT custody and inventory system in 2,705 lines of PHP (Laravel backend).
It has 50 endpoints over 22 tables. 4 SQL triggers enforce immutability.
Every receipt, issue, and return makes a numbered document and an append-only ledger entry.

---

## The problem

Most teams keep device custody in Excel sheets. Anyone can edit a sheet later.
In a custody dispute, there is no proof that nobody touched the record.

Application-level protection alone is not enough.
Anyone who reaches the database file bypasses the whole application with one line of SQL.

And an auditor asks one hard question. Why is there a gap in the document numbers?
One gap in the numbering points to a deleted document.

So the system needs a log that rejects edits from inside the database.
It also needs numbering that never leaves a gap.

---

## How it works

### 1) Two append-only ledgers — enforced by the database engine itself

The `movements` ledger records 8 movement types: receive, issue, return, repair-out, repair-in, dispose, transfer, adjust.
The `audit_logs` ledger records user actions from 28 logging call sites in the code.

The protection is not a check in application code. It is 4 triggers inside SQLite, copied here from the migration:

```sql
CREATE TRIGGER trg_audit_logs_no_update BEFORE UPDATE ON audit_logs
BEGIN SELECT RAISE(ABORT, 'audit_logs is append-only'); END;

CREATE TRIGGER trg_audit_logs_no_delete BEFORE DELETE ON audit_logs
BEGIN SELECT RAISE(ABORT, 'audit_logs is append-only'); END;
```

The same pair guards `movements`. I tested this on a copy of the system database. I used direct access, with no application in the way:

```text
UPDATE audit_logs SET action='HACKED' WHERE id=1;  → Error: audit_logs is append-only
DELETE FROM audit_logs WHERE id=1;                 → Error: audit_logs is append-only
```

### 2) Gapless sequential numbering

A `doc_sequences` table holds one counter per (document type, scope, year) behind a unique constraint.
One transaction takes the number and creates the document and its movements.
If any line fails, the whole transaction rolls back. The counter rolls back with it, so no number is lost.

The system also has no delete path for documents. There are 0 delete endpoints on receipts, issues, and returns.
You fix a mistake with a compensating entry (an ADJUST movement). The history stays complete.
Number format: `GRN-2026-0001`. Type, then year, then a 4-digit counter.

### 3) One transaction per operation

Take a goods receipt. The number, the lines, the stock units, and the movements live in one `DB::transaction`.
So no document exists without its movements. No movement exists without its document. And no balance disagrees with its ledger.

---

## The key design decision

**The problem:** an audit log guarded by application code fails against direct database access.
A sysadmin, a leaked backup, or a maintenance session can rewrite history in silence.

**The decision:** move the enforcement to the lowest layer: triggers inside the database engine.
They reject UPDATE and DELETE on both ledgers. It does not matter who connects, or with what tool.

**Cost and payoff:** the cost is that you cannot edit a mistake away. You reverse it with a visible compensating entry.
The payoff is that an auditor checks the protection with one query on `sqlite_master`.
To bypass it, someone must run DROP TRIGGER. That leaves a trace. It is not a silent edit.

## The limits, honestly

- Anyone with schema privileges can drop the triggers and make them again. The mechanism makes tampering a deliberate, visible act. It does not make tampering mathematically impossible. If the threat gets bigger, the next step is a hash chain or WORM backups.
- `lockForUpdate()` is a no-op on SQLite. What really guards the counter is SQLite's single-writer model. The code stays correct if we move to MySQL or Postgres. There the lock becomes real.
- User-action logging swallows its own failures and writes a warning. This picks availability over completeness. The movements ledger works the opposite way. It sits inside the transaction, so its failure aborts the whole operation.

---

## What is in this repository

The code belongs to my employer, so it is not here. The production system's structure:

```
backend (Laravel + SQLite)
  migrations    a 262-line migration: 18 tables + 4 triggers, plus 4 tables for an AI vision layer
  Services      5 services: numbering, audit, receipts, asset-tag generation, vision
  Controllers   15 controllers behind 50 endpoints
  Models        23 Eloquent models
  Middleware    write permission separated from admin permission
frontend        React + Vite + Ant Design with an Arabic RTL interface
```

There are 3 roles: admin, storekeeper, viewer. Every write goes through a `CanWrite` middleware.

## Why I built it

I work as an IT supervisor. Device custody used to live in sheets, and people edited them later with no trace.
Now 8 movement types flow into one ledger that the database itself refuses to change.
