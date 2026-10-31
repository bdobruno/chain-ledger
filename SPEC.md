# chainledger — specification v1

A double-entry engine for finance-engineering processes that sit next to the accounting
books: compensation (compi), payouts, expenses, revenue schedules, financial models — any
process where money (or any amount) moves through states and must be traceable.

Apps declare their ledgers in config (subjects, accounts, entry types) and post entries.
**Apps decide *what* should be posted; the ledger decides *whether it can be*.** chainledger
is a library, not a service: a storage-agnostic core plus storage adapters. The first
adapter installs into a Postgres database you point it at (e.g. Cloud SQL) and enforces
every rule inside the database; a SQLite adapter follows for local development and tests.

Scope: chainledger provides double-entry primitives only — balances, immutability,
idempotency, atomic batches. Domain logic (what counts as an accrual, when a line is
eligible) and integrity proofs (hashing) belong to the app; see §9 and §10.

Status: revised design, not yet implemented. Build order: §11.

---

## 1. Concepts

| Term | Meaning |
|---|---|
| **Ledger** | One independent book, e.g. `compi` or `stripe`: its own subjects, accounts and entry types. Carries a currency label. |
| **Subject kind** | A kind of thing money belongs to, with a fixed list of segments, e.g. `line = [owner, deal, component]`, `charge = [charge]`, or `stripe = []` (a single subject). |
| **Subject** | One concrete thing of a kind: `line:u123:d456:c789`, `charge:ch_1`, `stripe`. |
| **Account** | A named balance of one subject, declared per subject kind: `line:u123:d456:c789:eligible`, `charge:ch_1:balance`, `stripe:fees`. Has a balance and a sequence number (R8). |
| **Origin account** | An account where money enters the ledger (e.g. compi's `source`, Stripe's `gross`). It may go negative from sending (R5). |
| **Entry type** | A declared shape of movement: which accounts an entry debits/credits. The only movements apps can post (R1). A linear route is just a set of two-leg types. |
| **Entry** | One posted event of an entry type: two or more **legs**, each moving an amount out of (`from`) or into (`to`) one account. Legs may belong to different subjects. |
| **Batch** | The entries of one `post()` call. All are written or none. |
| **Ref** | Optional app-defined string on an entry grouping related entries, e.g. compi's `compi_line_id`. |
| **Tag** | A named text value attached to a ref or to one entry, e.g. `quota_period_name = "FY26 Q3"`. Append-only; the latest value wins. Doesn't touch balances. |
| **Core** | The storage-free part of the library: config validation, plan/diff, `describe`, and resolving an entry type + params + amounts into legs (§3.1). |
| **Adapter** | The storage-specific part: stores declarations and entries and guarantees the stateful rules (§3.2). |

## 2. Configuration (TOML)

compi — one subject kind, a linear flow of two-leg, same-subject entry types:

```toml
[ledgers.compi]
currency = "USD"            # label only; no conversion ever happens
decimals = 2                # display: 1 USD = 100 minor units

[ledgers.compi.subjects.line]
segments = ["owner", "deal", "component"]
accounts = ["source", "eligible", "earned", "payroll", "paid"]
origin   = ["source"]

# Shorthand for two legs on the same subject: from/to are account names of one kind;
# the entry's params are that kind's segments.
[ledgers.compi.entry_types]
accrue     = { from = "line.source",   to = "line.eligible" }
earn       = { from = "line.eligible", to = "line.earned" }
to_payroll = { from = "line.earned",   to = "line.payroll" }
clear      = { from = "line.payroll",  to = "line.paid" }
```

Stripe — several subject kinds, multi-leg and cross-subject entry types:

```toml
[ledgers.stripe]
currency = "EUR"
decimals = 2

[ledgers.stripe.subjects.charge]
segments = ["charge"]
accounts = ["gross", "balance"]
origin   = ["gross"]

[ledgers.stripe.subjects.payout]
segments = ["payout"]
accounts = ["in_transit", "paid_out"]

# Hot account: every charge posts here, so writers serialize on it (R11, §9).
# Fine for an example; shard it (e.g. by day) if write volume matters.
[ledgers.stripe.subjects.stripe]
segments = []
accounts = ["fees"]

# Full form: explicit legs with account patterns; {placeholders} are the entry's params.
[ledgers.stripe.entry_types.charge]
legs = [
  { from = "charge:{charge}:gross" },
  { to   = "charge:{charge}:balance" },
  { to   = "stripe:fees" },
]

[ledgers.stripe.entry_types.settle]
legs = [
  { from = "charge:{charge}:balance" },
  { to   = "payout:{payout}:in_transit" },
]

[ledgers.stripe.entry_types.pay_out]
legs = [
  { from = "payout:{payout}:in_transit" },
  { to   = "payout:{payout}:paid_out" },
]
```

Naming rules:
- Ledger ids, subject kinds, segment names, account names, entry type names and params
  match `[a-z][a-z0-9_]*`.
- Segment values match `[A-Za-z0-9_.@-]+` (no `:`). Every subject has exactly its kind's
  segments; apps use a placeholder (e.g. `_`) for a segment that doesn't apply.
- Account names are `{kind}:{segment values…}:{account}`, or `{kind}:{account}` for a kind
  without segments. Subject keys are the same without the `:{account}` suffix.
- Granularity (which kinds, which segments) is the app's choice. Rules and locks apply per
  account, so the finest declared level is where R5 holds and where writers contend (R11).

## 2a. Declared as code (plan / apply), built for agents

Ledgers are infrastructure-as-code, like Terraform: the TOML lives in the app's git repo,
changes go through review, and the store is brought in line with it. Humans and AI agents
use the same workflow.

| Command (CLI and client API) | Needs store | What it does |
|---|---|---|
| `validate(config)` | no | checks the config offline (naming, patterns, placeholders, R14 shape rules) |
| `plan(config)` | read-only | diffs config against the stored declaration: what `apply` would add, and anything it would reject as `INCOMPATIBLE_CHANGE`; machine-readable JSON plus a human summary |
| `apply(config, plan_hash?)` | yes | runs `declare`; with `plan_hash`, refuses with `STALE_PLAN` if the declaration changed since that plan (like a saved Terraform plan) |
| `describe(ledger)` | read-only | the declared model as JSON: subject kinds, accounts, entry types with their params and legs — what an agent may post |
| `post(..., dry_run = true)` | yes | runs every rule and returns the receipts it *would* produce, then rolls back; nothing is written |

Guardrails for agents:
- Agents can only post declared entry types (R1), so the config is also the boundary of what
  an agent can do. Changing that boundary is a config change, which goes through review:
  `apply` against production is meant to run from CI after approval, not ad hoc.
- With an adapter that enforces rules in storage (Postgres), the boundary also holds at the
  database: app roles have no way to write anything else (R12, R19). With other adapters it is
  a credential boundary: agents get the library (or a later MCP server), never the database.
- Idempotency keys (R9) make agent retries safe; stable rejection codes with JSON detail (§7)
  let an agent understand and fix a refused entry.
- `dry_run` lets an agent test a batch against real balances and rules first.
- An MCP server exposing `describe`, `post` (with `dry_run`), `balance`, `history` and
  `check` as tools is a thin layer over the client API (later).

## 3. Architecture: core and adapters

### 3.1 Core (pure, no I/O)

- Parses and validates config (naming, patterns, placeholders, `BAD_CONFIG`).
- Diffs config against a stored declaration (`plan`, R14) and renders `describe`.
- Resolves `{type, params, amount | amounts}` into legs and checks everything that needs no
  state: R1 (declared type, valid params), R2 (legs, amounts per leg), R3 (balanced),
  R4 (signed integers, one sign per entry).
- Defines the rejection codes (§7).

Resolution is specified once, by the core. Every adapter must produce identical legs for the
same input (conformance test 20). An adapter may call the core (in-process adapters such as
SQLite) or re-implement the resolution where the core is not available — the Postgres
adapter does, because the database must not depend on its clients.

### 3.2 Adapter contract

An adapter owns storage and guarantees every rule that needs state. For each `post()` batch:

1. **Atomic** — all entries are written or none (R10).
2. **Idempotent** — keys are unique per ledger; replay and conflict behave as R9.
3. **No over-release** — R5 is evaluated at batch end against balances read under lock.
4. **Accounts from config** — accounts are created on first use, only from declared types (R13).
5. **Append-only** — written entries and legs are never changed (R7).
6. **Account sequence** — each account's legs are numbered 1, 2, 3, … without gaps and its
   balance equals the sum of its legs (R8).
7. **No deadlocks** — concurrent batches on overlapping accounts never deadlock (R11).

Operations an adapter provides (details in §5): `install`/`version`, `declaration` (stored
config, its revision, which entry types have been used), `declare` (atomic, with an optional
expected revision), `post` (with `dry_run`), the reads (`balance`, `balances`, `history`,
`entry`, `by_ref`, `trial_balance`, `tags`), `changes`, `check`, and `close_period`.

### 3.3 Capabilities

Adapters declare which of these they have; the conformance suite (§8) runs the matching
tiers.

| Capability | Meaning | Postgres | SQLite |
|---|---|---|---|
| `rules_in_storage` | The store itself rejects anything but valid `post()` calls (R12, R19). | yes | no — enforced by the library; access to the file is the trust boundary |
| `shares_app_transaction` | An app can post inside its own transaction on the same database (R18). | yes | yes (same connection) |
| `concurrent_writers` | Writers on disjoint accounts do not wait for each other (R11). | yes | no — writers serialize, still no deadlocks |

Key-value stores (LevelDB, DynamoDB) are out of scope: the contract needs atomic writes
across several accounts with conditional checks and a unique key, which they cannot provide
without extra machinery (§10).

## 4. Rules

**R1 Declared entry types only.** Every entry names an entry type of its ledger and gives the
type's params. Its legs are exactly the type's legs, with params substituted.
→ `UNKNOWN_TYPE`, `BAD_PARAMS` (missing, extra, or invalid segment value)

**R2 Legs.** Each leg has a role (`from` or `to`), an account and an amount.
- A `from` leg lowers the account's balance by its amount; a `to` leg raises it.
- An entry has at least one `from` and one `to` leg.
- Amounts are given per leg (`amounts`, in the type's leg order), or as one `amount` for a type
  with exactly one `from` and one `to` leg. → `BAD_AMOUNTS`

**R3 Balanced.** The `from` amounts of an entry sum to its `to` amounts. So the sum of all
account balances of a ledger is always 0 (trial balance). → `UNBALANCED`

**R4 Signed integer amounts.** Amounts are non-zero integers in minor units, range of a
signed 64-bit integer. No floats or decimals anywhere. All legs of one entry have the same
sign: positive is a normal movement, negative is a reduction flowing the same way (e.g. a
negative commission delta). → `ZERO_AMOUNT`, `BAD_AMOUNT`, `MIXED_SIGNS`

**R5 No over-release.** For every non-origin account that is a `from` leg with a positive
amount anywhere in a batch, its balance after the batch must be ≥ 0. Accounts that only send
negative amounts, or only receive, may end negative. Origin accounts are exempt.
→ `OVER_RELEASE` (reports the account and the first entry index that sent from it)

Examples (compi; eligible starts at 100):

| Batch | Eligible after | Result |
|---|---|---|
| `earn +100` | 0 | ok |
| `earn +200` | −100 | `OVER_RELEASE` |
| `accrue −30`, `earn +100` | −30 | `OVER_RELEASE` |
| `accrue −30`, `earn +70` | 0 | ok |
| `accrue −130` | −30 | ok (nothing positive sent; may end negative) |
| `accrue −130`, `earn −30` | 0 | ok (only negative amounts sent) |

**R6 Origin.** Money enters a ledger only through origin accounts; every other account can
only hold what an entry moved into it from another account of the ledger.

**R7 Append-only.** Entries, legs and tags are never updated or deleted. Corrections are new
entries.

**R8 Account sequence.** Each account numbers the legs applied to it `1, 2, 3, …` with no
gaps, in the order they were applied under the account's lock; legs of one entry that fall in
the same account are applied in leg order. An account's balance is always the sum of its legs
(`to` adds, `from` subtracts), so any balance can be recomputed from the legs alone (R15).

**R9 Idempotency.** Each entry carries an app-supplied `key`, unique per ledger.
- Same key, same content (type, params, amounts, ref, metadata, and `effective_at` if given):
  the original receipt is returned with `replayed = true`; nothing is written.
- Same key, different content → `DUPLICATE_KEY`.
- A batch may mix replayed and new entries; only the new ones are written.
- The same key twice within one batch → `DUPLICATE_KEY`.

**R10 Atomic batches.** On any rejection nothing from the batch is written. The error carries
the code and the index of the failing entry.

**R11 Parallel writers.** `post()` locks only the accounts its batch touches. In Postgres:
row-lock their `accounts` rows (`SELECT … FOR UPDATE`, sorted by account id, after creating
missing accounts with `INSERT … ON CONFLICT DO NOTHING`), so writers never deadlock. Row
locks, not advisory locks: advisory locks use Postgres's fixed-size shared lock table, which a
batch touching thousands of accounts would exhaust. Writers wait for each other only when they
touch a common account; everything else runs in parallel. No ledger-wide lock exists on the
write path. Adapters without `concurrent_writers` serialize writers but must still never
deadlock.

**R12 Rules live in the adapter.** Every rule runs inside the adapter's `post()`.
- Postgres (`rules_in_storage`): all rules run inside the database (`post()` plus triggers), so
  any client is bound by them. Any INSERT, UPDATE, DELETE or TRUNCATE on chainledger tables
  outside chainledger's own functions is rejected. → `DIRECT_WRITE`. Client libraries only make
  calling the SQL convenient (they may pre-check for friendlier errors; Postgres has the final
  say).
- Other adapters enforce the rules in the library; whoever can open the store directly can
  bypass them, so access to the store is the boundary (§3.3).

**R13 Accounts from config.** Accounts are created on first use when an entry type's legs
resolve to them. Nothing else creates accounts. → `UNKNOWN_LEDGER`

**R14 Declare is idempotent.** `declare(config)` creates or updates ledgers and increments the
ledger's declaration `revision` when something changed.
- Adding subject kinds, accounts to a kind, or entry types: allowed.
- Changing a ledger's currency or decimals, a kind's segments or origin accounts, removing or
  renaming anything, or changing an entry type that has been used → `INCOMPATIBLE_CHANGE`.
  (To change a used entry type, declare a new one.)
- Invalid config (unknown accounts in a type, placeholders not covering a pattern's segments,
  a type with no `from` or no `to` leg) → `BAD_CONFIG`.
- With an expected revision (from `plan`), a changed declaration → `STALE_PLAN`.

**R15 Check.** `check(ledger)` recomputes from the stored legs alone: every account's balance
and sequence (no gaps), and the trial balance (= 0). Returns ok, or the first failing account
and what failed. It finds bugs and storage corruption; it is **not** tamper evidence — anyone
who can rewrite legs can rewrite what `check` reads (§10). Cost is proportional to history, so
run it on demand, e.g. before a period close.

**R16 Tags.** Values that change over time without touching entries (e.g. a line's quota
period being renamed).
- A tag is `{target, name, value}`. `target` is `ref:{ref}` (every entry with that ref, past
  and future) or `entry:{key}` (one entry). `name` matches `[a-z][a-z0-9_]*`; `value` is text.
- Append-only: setting a tag again appends a new row; the current value of `(target, name)` is
  the one with the highest `tag_seq`. History is kept.
- Appending a value equal to the current one is a no-op, so refreshing tags every run is
  idempotent.
- Each target has its own tag sequence (`tag_seq` 1, 2, 3, …), locked like R11 (Postgres:
  row-lock its `tag_heads` row).
- A target `entry:{key}` must exist → `UNKNOWN_TARGET`; a `ref:` target need not. Invalid name
  → `BAD_TAG`.
- Tags never affect balances or rules R1–R5.

**R17 Effective date and period close.** *(`effective_at` is stored from phase 1; period close
is implemented in phase 6.)*
- Every entry has `effective_at` (the date it belongs to) besides `created_at` (when it was
  posted, from the store's clock). Default: `effective_at = created_at`.
- `close_period(ledger, until)` appends a close record: from then on, entries with
  `effective_at < until` are rejected → `PERIOD_CLOSED`. Closes are append-only and only move
  forward.
- `balance(ledger, account, as_of)` returns the balance over entries with `effective_at < as_of`
  (point-in-time reporting).

**R18 Transaction participation.** *(Adapters with `shares_app_transaction`.)* `post()` can be
called inside the app's own transaction on the same database: the app's writes and the ledger
entries commit together or not at all, with no outbox. A rejected `post()` raises (R10 still
holds for its batch); whether the app's transaction carries on is the app's choice (e.g. a
savepoint). The locks of R11 are held until the app's transaction ends, so apps post at the end
of their transaction (§9).

**R19 Permissions.** *(Adapters with `rules_in_storage`.)* Callers connect as store roles:
- the **installer** role runs `install`, `declare` and `grant`/`revoke`; `apply` against
  production runs as the installer, from CI after approval;
- **poster** roles may call `post` only for the ledgers (and, optionally, entry types)
  granted to them with `grant(role, ledger, entry_type?)`;
- **reader** roles may call the read operations;
- no role has direct access to chainledger tables; they are reached only through chainledger's
  functions (R12).

A poster calling outside its grants → `PERMISSION_DENIED`.

**R20 Versioning.** `version()` returns the installed schema version (semver). Within a major
version changes are additive only: new functions, new nullable or defaulted columns, new
rejection codes. Existing function signatures, rejection codes and rule semantics don't
change. `install()` upgrades in place through versioned migrations and is idempotent. Clients
assert a minimum version when they connect and fail fast on a mismatch (a client-side error,
not a rejection).

## 5. Operations

All are SQL functions in schema `chainledger` in the Postgres adapter; the Rust client and
bindings wrap them, and the other adapters provide the same operations.

| Operation | Input | Returns |
|---|---|---|
| `install()` / `version()` | — | creates/upgrades the schema (versioned migrations) / semver |
| `declare(config, expected_revision?)` | parsed TOML | created / changed / unchanged per ledger, with the new `revision` (used by `apply`, §2a) |
| `plan(config)` / `describe(ledger)` | §2a | JSON diff / declared model |
| `grant(role, ledger, entry_type?)` / `revoke(...)` | R19 | — |
| `post(ledger, entries[], dry_run?)` | entries: `{key, type, params, amount \| amounts, ref?, effective_at?, metadata}` | per entry: `{entry_id, legs: [{account, seq}], replayed}` |
| `entry(ledger, key)` | | the entry with its legs |
| `by_ref(ledger, ref)` | | entries with that ref, by `created_at`, then key |
| `balance(ledger, account, as_of?)` | full account name | balance |
| `balances(ledger, prefix)` | e.g. `line:u123` or `line:u123:d456` | balance per account name, summed over matching subjects |
| `history(ledger, account)` | full account name | that account's legs, in sequence order |
| `tag(ledger, tags[])` | `{target, name, value}` | per tag: `{target, tag_seq, unchanged}`; atomic |
| `tags(ledger, target)` | `ref:{ref}` or `entry:{key}` | current value per tag name |
| `tag_history(ledger, target, name)` | | every value in `tag_seq` order |
| `changes(ledger, after_cursor?, limit?)` | an opaque cursor | every entry, leg and tag row after the cursor, plus `next_cursor`; exact, no gaps or duplicates, even with concurrent writers (export/BigQuery). The cursor never advances past a transaction that may still commit. |
| `trial_balance(ledger)` | | total per account name; sum is 0 |
| `check(ledger)` | R15 | ok / first failure |
| `close_period(ledger, until)` | R17 (phase 6) | the close record |

Wire formats:
- Amounts travel as JSON **strings** of integers (e.g. `"-3000"`); stored as `bigint`.
- Timestamps are RFC 3339 UTC, truncated to microseconds.
- `metadata` travels as a JSON **string** containing canonical JSON (RFC 8785), produced by the
  client, so the content comparison of R9 is exact. Postgres stores those exact bytes,
  compressed; no `jsonb` copy. Querying on metadata happens downstream (BigQuery/DuckDB) or
  with an explicit `metadata::jsonb` cast.
- `post()` processes the whole batch with set-based SQL (`unnest` over arrays), not a loop per
  entry. Large imports go through `post()` in large batches; there is no bulk path that skips
  the rules.

## 6. Postgres adapter

Requires PostgreSQL 13 or later (`xid8`, `pg_snapshot`).

### 6.1 Install and roles

`install()` creates schema `chainledger` (tables, functions, triggers) and the roles of R19.
Chainledger's functions are `SECURITY DEFINER` with a fixed `search_path`, so app roles reach
the tables only through them. On managed Postgres (Cloud SQL), `install()` runs as a role that
may create schemas and roles; document that as the installer's required privilege.

### 6.2 Reference schema

Implementers may adjust, as long as rules and operations hold. Storage goals: rows refer to
ledgers, kinds, types and accounts by integer id; nothing derivable is stored twice. Reference
point: pgledger reports about 743 bytes per transfer; measure and publish the real figure
(test 25).

Declarations:
- `ledgers` — `id text` PK, `no smallint` unique, `currency`, `decimals`, `config jsonb` (as declared), `revision int`, `created_at`
- `subject_kinds` — `(ledger_no, kind_no)` PK, `name`, `segments text[]`
- `account_types` — `(ledger_no, kind_no, account_no)` PK, `name`, `origin bool`
- `entry_types` — `(ledger_no, type_no)` PK, `name`, `legs jsonb` (resolved patterns), `used bool`
- `grants` — `(role, ledger_no, type_no)`; `type_no` null means every type of the ledger (R19)

State (updated on every post; `fillfactor = 80` so updates stay HOT):
- `accounts` — `id bigint` PK, `ledger_no`, `kind_no`, `subject text` (e.g. `line:u123:d456:c789`), `account_no smallint`, `balance bigint`, `seq bigint`; unique `(ledger_no, subject, account_no)`. Its row lock is the account lock (R11). Names are derived (`subject || ':' || name`), not stored.
- `tag_heads` — PK `(ledger_no, target)`, `tag_seq`

History (append-only):
- `entries` — `id bigint` PK, `ledger_no`, `key text`, `type_no`, `ref text` (indexed), `effective_at`, `created_at`, `metadata text` (compression `lz4`), `xid xid8` (inserting transaction; bounds the `changes()` cursor); unique `(ledger_no, key)`
- `legs` — PK `(entry_id, leg_index)`, `account_id`, `role` (0/1), `amount bigint`, `account_seq bigint`; unique `(account_id, account_seq)`
- `tags` — PK `(ledger_no, target, tag_seq)`, `name`, `value`, `created_at`, `xid`; index `(ledger_no, target, name, tag_seq desc)`
- `period_closes` — PK `(ledger_no, close_no)`, `until`, `closed_at` (R17, phase 6)

### 6.3 Guards

- Triggers on all tables reject writes unless `current_setting('chainledger.writing', true) = 'on'`,
  which only chainledger's functions set (`set_config(..., true)`, transaction-local).
- History tables reject every UPDATE and DELETE (R7).
- App roles have no table privileges (R19).

Note: a database superuser can still disable triggers and rewrite rows. v1 does not detect
that (§10); production access to the database is the control.

### 6.4 Export cursor

`changes()` must not skip rows that commit out of order. Rows record the id of the transaction
that wrote them (`xid`); the cursor is `(xid, id)` and an export only reads rows whose `xid` is
below the lowest still-running transaction (`pg_snapshot_xmin(pg_current_snapshot())`), so a
row that is not yet visible can never be passed over.

## 7. Client libraries

- **Rust crate** `chainledger`: the core (§3.1), the adapter trait, the Postgres and SQLite
  adapters, typed errors from rejection codes, a sync driver, and the conformance harness (§8)
  so any adapter can be tested the same way.
- **Python package** (PyO3 + maturin), same API. The connection URL picks the adapter:

```python
from chainledger import Client, Entry, Tag

cl = Client.connect(DATABASE_URL)        # postgresql://… or sqlite:///dev.db
cl.require_version(">=1.0")              # fail fast on a mismatch (R20)
cl.install()

plan = cl.plan_file("chainledger.toml")  # read-only diff; shows adds and INCOMPATIBLE_CHANGE
cl.apply_file("chainledger.toml", plan_hash=plan.hash)

line = {"owner": "u123", "deal": "d456", "component": "c789"}
batch = [
    Entry(key="line-1:accrue:1", type="accrue", params=line, amount=5_000_00,
          ref="line-1", metadata={"comment": "..."}),
]
cl.post("compi", batch, dry_run=True)    # same rules and receipts, nothing written
cl.post("compi", batch)

cl.post("stripe", [
    Entry(key="ch_1:charge", type="charge", params={"charge": "ch_1"},
          amounts=[100_00, 97_00, 3_00]),
    Entry(key="ch_1:settle", type="settle", params={"charge": "ch_1", "payout": "po_9"},
          amount=97_00),
])
cl.tag("compi", [Tag(target="ref:line-1", name="quota_period_name", value="FY26 Q3")])
assert cl.check("compi").ok
```

To post inside the app's own transaction (R18), pass the app's connection or transaction to
`post` (e.g. `cl.post(..., conn=conn)`); the exact form is settled in phase 3.

- Node bindings: later, not v1.

Rejection codes: `UNKNOWN_LEDGER`, `UNKNOWN_TYPE`, `BAD_PARAMS`, `BAD_AMOUNTS`, `BAD_AMOUNT`,
`ZERO_AMOUNT`, `MIXED_SIGNS`, `UNBALANCED`, `OVER_RELEASE`, `DUPLICATE_KEY`, `DIRECT_WRITE`,
`INCOMPATIBLE_CHANGE`, `BAD_CONFIG`, `STALE_PLAN`, `PERMISSION_DENIED`, `UNKNOWN_TARGET`,
`BAD_TAG`, `PERIOD_CLOSED`. In Postgres they are raised with SQLSTATE class `CL` and a JSON
message `{code, index, detail}`; other adapters surface the same `{code, index, detail}`.

## 8. Conformance suite

The acceptance tests are the conformance suite: one set of tests, run against every adapter.
An adapter either passes the tiers of the capabilities it declares (§3.3) or is not supported.
Tests run from Python against a local database.

**Tier A — every adapter** (phase 1 on Postgres, phase 4 on SQLite):

01. compi happy path: accrue → earn → to_payroll → clear; balances and trial balance correct.
02. Negative delta flows through the same types; accounts may end negative.
03. R5: every row of the R5 table behaves as stated; origin accounts are exempt.
04. Stripe: `charge` with three legs, `settle` across subjects (charge → payout), `pay_out`;
    balances per subject correct; the settle entry appears in the history of accounts of both
    subjects.
05. `UNKNOWN_TYPE`, `BAD_PARAMS`, `BAD_AMOUNTS`, `UNBALANCED`, `MIXED_SIGNS`, `ZERO_AMOUNT`.
06. Idempotency: replay returns the same receipt with `replayed = true`; changed content →
    `DUPLICATE_KEY`; duplicate key within a batch → `DUPLICATE_KEY`.
07. Atomicity: a batch with one bad entry writes nothing; balances and sequences unchanged.
08. N concurrent writers on one account → no sequence gaps, correct balances; two writers that
    each try to release the account's whole balance never over-release (R5 under concurrency).
09. `balances` by prefix sums correctly at each level.
10. `declare` is idempotent and bumps `revision` only on change; incompatible changes are
    rejected; `BAD_CONFIG` for invalid types; `declare` with a stale expected revision →
    `STALE_PLAN`.
11. `describe` round-trips the config. (`validate` offline and `plan` reporting adds and
    incompatible changes without writing arrive with the core, phase 3.)
12. `post(dry_run = true)` returns the same receipts/rejections as a real post and writes
    nothing.
13. `effective_at` defaults to `created_at` and is stored; `created_at` comes from the store's
    clock.
14. `check` passes on a healthy ledger and reports the first account whose stored balance or
    sequence disagrees with its legs.
15. One batch touching 50,000 distinct accounts succeeds on default settings.
16. `version()` reports the version; `install()` on an existing store is idempotent and keeps
    data; a client requiring a higher version fails fast.

**Tier P — `concurrent_writers`:**

17. Writers on disjoint accounts (and on different ledgers) don't wait for each other; batches
    touching overlapping accounts in opposite orders don't deadlock.

**Tier S — `rules_in_storage`:**

18. `DIRECT_WRITE` on INSERT/UPDATE/DELETE/TRUNCATE outside chainledger's functions.
19. Roles: a reader can't post; a poster can post only to its granted ledgers and entry types
    (`PERMISSION_DENIED` otherwise); no app role can read or write chainledger tables
    directly.
20. Resolution parity: for a corpus of configs, entries and amounts (including generated ones),
    the adapter's resolved legs and rejections equal the core's (phase 3).

**Tier T — `shares_app_transaction`:**

21. An app write plus `post()` in one transaction commit together; rolling back the app
    transaction leaves no entries; a rejected `post()` leaves the app free to roll back or
    continue.

**Export and tags** (phase 5):

22. `changes(after_cursor)` returns every entry, leg and tag row after the cursor, none twice,
    none missing, with concurrent writers that commit out of order.
23. Tags: a new value appends and becomes current; history keeps both; the same value again is
    a no-op; `ref:` tags apply to entries posted later with that ref; `UNKNOWN_TARGET`,
    `BAD_TAG`.

**Periods** (phase 6):

24. After `close_period(until)`, entries with `effective_at < until` → `PERIOD_CLOSED`; later
    ones succeed; closes only move forward; `balance(as_of)` correct.

**Storage** (Postgres):

25. Measure and publish average bytes per leg (excluding metadata); set the target from the
    first measurement.

## 9. Design guidelines

- **Apps decide what; the ledger decides whether.** An app computes "this line earned 5,000"
  and posts `earn`; the ledger accepts or rejects it with a stable code. The app never touches
  balances directly.
- **Keep domain logic out of the engine.** Quota periods, eligibility and similar rules live in
  the app, and in refs, tags and metadata — never in the core or an adapter.
- **Keep subjects fine-grained.** R11 gives parallelism only when writers touch different
  accounts. An account that every entry touches serializes all writers; shard it.
- **Post at the end of the app's transaction** (R18): the account locks are held until commit.
- **Corrections are new entries** (R7). After a period close, an adjustment is an entry with
  `effective_at` in the open period and a `ref` pointing back to the original.
- **Cross-ledger flows are the app's job.** Ledgers are independent; the app posts to each and
  links the entries with a shared `ref`.
- **Models and scenarios.** A model run is the output of a computation, not something to edit.
  Use one ledger per scenario or run, with idempotency keys that include a hash of the
  assumptions so reruns are safe. Forking a ledger is copy-and-replay in v1.
- **Own the integrity story in the app.** Until the ledger offers an integrity layer (§10),
  apps that need tamper evidence add their own (e.g. a signed export of `changes()`).

## 10. Not in v1

Currency conversion/FX (currency is a label) · pending/held transfers (model them as
accounts/entry types if needed) · chart of accounts, account kinds, financial statements
(that's the accounting system's job) · **integrity proofs**: hash chains, seals, Merkle
roll-ups, external anchoring and the immutable Parquet archive were removed from v1 — because
entries are immutable and ordered per account, an optional integrity layer can be added later
without changing the model · key-value adapters (LevelDB, DynamoDB): the contract needs atomic
multi-account writes with conditional checks, a unique key and, for storage-level rules,
triggers · adapters for hosted ledger services (Formance, Fragment): not planned, only if
asked for · MCP server (later) · summary journals for the GL, e.g. NetSuite (planned) ·
scenario branching (fork without replay) · Node bindings.

## 11. Build phases

Each phase is usable on its own.

1. **Postgres adapter in SQL** — schema, `install`, `version`, `declare` (with `revision`),
   `post` (R1–R14, R18–R20, `dry_run`, `effective_at` stored), reads, `check`, `describe`,
   roles and grants. Conformance tiers A, P, S and T against a local Postgres, from Python
   (tests 01–19, 21).
2. **First consumer: compi** — replay a month of real compi history; compare balances with
   today's BigQuery totals. Validates the model before building further.
3. **Core library and Python bindings (Rust)** — `validate`, `plan`, `apply`, `describe`
   offline, the adapter trait and conformance harness, the exact form of transaction
   participation in the client API (test 20, rest of test 11).
4. **SQLite adapter** — passes tiers A and T; fast local tests for apps.
5. **Export and tags** — `changes()` with its cursor, tags (tests 22–23).
6. **Second consumer: a financial-model flow** (e.g. an ARR bridge or revenue schedule) —
   validates negative deltas, ledger-per-scenario and agent-proposed entries (`describe`,
   `dry_run`); add period close (R17, test 24). Stripe payouts stay as the multi-leg
   reference ledger in the conformance suite.
