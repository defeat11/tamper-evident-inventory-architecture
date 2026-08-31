# tamper-evident-inventory-architecture

[العربية](README.md)

**A piece of hardware enters the warehouse → a gapless sequential document, plus a ledger entry the database itself refuses to alter.**

> **This system runs in production at my employer, and the code belongs to them. This repository documents the architecture, the engineering decisions, and how it was built.**

An IT custody and inventory system in 2,705 lines of PHP (Laravel backend).
It exposes 50 endpoints over 22 tables, with 4 SQL triggers enforcing immutability.
Every receipt, issue, and return produces a numbered document and an append-only ledger entry.

---

## The problem

Device custody usually lives in Excel sheets that can be edited retroactively.
When a custody dispute happens, there is no proof the record was never touched.

Application-level protection alone is not enough.
Anyone who reaches the database file bypasses the whole application with one line of SQL.

And an auditor asks one killer question: why is there a gap in the document sequence?
A single gap in the numbering suggests a deleted document.

Required: a log that rejects edits even from inside the database, and numbering that cannot produce gaps.

---

## How it works

### 1) Two append-only ledgers — enforced by the database engine itself

The `movements` ledger records 8 movement types: receive, issue, return, repair-out, repair-in, dispose, transfer, adjust.
The `audit_logs` ledger records user actions from 28 logging call sites in the code.

The protection is not a check in application code. It is 4 triggers inside SQLite, quoted verbatim from the migration:

```sql
CREATE TRIGGER trg_audit_logs_no_update BEFORE UPDATE ON audit_logs
BEGIN SELECT RAISE(ABORT, 'audit_logs is append-only'); END;

CREATE TRIGGER trg_audit_logs_no_delete BEFORE DELETE ON audit_logs
BEGIN SELECT RAISE(ABORT, 'audit_logs is append-only'); END;
```

The same pair guards `movements`. A real test on a copy of the system database, with direct access and no application in the way:

```text
UPDATE audit_logs SET action='HACKED' WHERE id=1;  → Error: audit_logs is append-only
DELETE FROM audit_logs WHERE id=1;                 → Error: audit_logs is append-only
```

### 2) Gapless sequential numbering

A `doc_sequences` table holds one counter per (document type, scope, year) behind a unique constraint.
The number is allocated inside the same transaction that creates the document and its movements.
If any line fails, the whole transaction rolls back and the counter rolls back with it — no number is ever lost.

The system also has no delete path for documents at all: 0 delete endpoints on receipts, issues, and returns.
Corrections happen through a compensating entry (an ADJUST movement), so history stays complete.
Number format: `GRN-2026-0001` — type, then year, then a 4-digit counter.

### 3) One transaction per operation

Take a goods receipt: the number, the lines, the stock units, and the movements all live in one `DB::transaction`.
So no document exists without its movements, no movement without its document, and no balance disagrees with its ledger.

---

## The key design decision

**The problem:** an audit log enforced by application code collapses against any direct database access.
A sysadmin, a leaked backup, a maintenance session — each can silently rewrite history.

**The decision:** push enforcement down to the lowest possible layer: triggers inside the database engine.
They reject UPDATE and DELETE on both ledgers regardless of who connects, and with what tool.

**Cost and payoff:** the cost is that mistakes cannot be edited away — only reversed by a visible compensating entry.
The payoff is that an auditor verifies the protection with a single query against `sqlite_master`.
And bypassing it requires an explicit act (DROP TRIGGER) that leaves a trace — not a silent edit.

## The limits, honestly

- Anyone with schema privileges can drop the triggers and recreate them. The mechanism makes tampering a deliberate, visible act; it does not make it mathematically impossible. A hash chain or WORM backups are the next step if the threat level rises.
- `lockForUpdate()` is a no-op on SQLite; what actually guards the counter is SQLite's single-writer model. The code stays correct on a move to MySQL or Postgres, because the lock becomes real there.
- User-action logging swallows its own failures with a warning — availability over completeness. The movements ledger is the opposite: inside the transaction, and its failure aborts the whole operation.

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

There are 3 roles: admin, storekeeper, viewer — and every write sits behind a `CanWrite` middleware.

## Why I built it

I work as an IT supervisor, and device custody used to live in sheets edited retroactively without a trace.
Now 8 movement types flow into one ledger that the database itself refuses to change.
