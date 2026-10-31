# chainledger — specification v1

A double-entry engine for finance-engineering processes that sit next to the accounting
books: compensation (compi), payouts, expenses, revenue schedules, rewards points, financial
models — any process where money (or any amount) moves through states and must be
traceable.

Apps declare their ledgers in config (subjects, accounts, entry types) and post entries.
**Apps decide *what* should be posted; the ledger decides *whether it can be*.**

chainledger is a Rust engine with thin Python and TypeScript clients:

```
Python / TypeScript  ──►  Rust engine  ──►  Store
  (thin clients)         (all the rules)    (plain tables: Postgres, SQLite;
                                             in-memory for tests)
```

- Clients never talk to the database. Only the engine does.
- All rules run in the engine, written once and testable without a database.
- The store is **dumb but authoritative**: no stored procedures, no rule logic, plain tables —
  but an entry is acknowledged only after the store has committed it, so after a crash the
  store is what you trust.
- Two modes with the same engine and the same client API: **library** (the engine runs inside
  the app process) and **service** (the engine runs in a server that clients call).

Scope: chainledger provides double-entry primitives only — balances, immutability,
idempotency, atomic batches. Domain logic (what counts as an accrual, when a line is
eligible) and integrity proofs (hashing) belong to the app; see §9 and §10.

Status: revised design, not yet implemented. Build order: §11.

---

## 1. Concepts

| Term | Meaning |
|---|---|
| **Ledger** | One independent book, e.g. `compi` or `stripe`: its own subjects, accounts and entry types. Carries a currency label. Also the unit of write ordering (R11). |
| **Subject kind** | A kind of thing money belongs to, with a fixed list of segments, e.g. `line = [owner, deal, component]`, `charge = [charge]`, or `stripe = []` (a single subject). |
| **Subject** | One concrete thing of a kind: `line:u123:d456:c789`, `charge:ch_1`, `stripe`. |
| **Account** | A named balance of one subject, declared per subject kind: `line:u123:d456:c789:eligible`, `charge:ch_1:balance`, `stripe:fees`. Has a balance and a sequence number (R8). |
| **Origin account** | An account where money enters the ledger (e.g. compi's `source`, Stripe's `gross`). It may go negative from sending (R5). |
| **Entry type** | A declared shape of movement: which accounts an entry debits/credits. The only movements apps can post (R1). A linear route is just a set of two-leg types. |
| **Entry** | One posted event of an entry type: two or more **legs**, each moving an amount out of (`from`) or into (`to`) one account. Legs may belong to different subjects. |
| **Batch** | The entries of one `post()` call. All are written or none. |
| **Ref** | Optional app-defined string on an entry grouping related entries, e.g. compi's `compi_line_id`. |
| **Tag** | A named text value attached to a ref or to one entry, e.g. `quota_period_name = "FY26 Q3"`. Append-only; the latest value wins. Doesn't touch balances. |
| **Engine** | The Rust library that validates config, enforces every rule, orders writes per ledger and talks to the store (§3.1). |
| **Store** | Plain tables in a database (or in memory) that hold declarations, entries, legs and balances. Knows no rules (§3.3). |

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

# Hot account: every charge touches it, so several engine instances writing this ledger
# will collide on it and retry (R11, §9). Fine for an example; shard it (e.g. by day) if
# several processes write at volume.
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

Rewards points — one subject per grant ("lot"), so expiry can be applied to old points
only; `source` lives inside the lot so no shared account is touched by every accrual:

```toml
[ledgers.rewards]
currency = "PTS"
decimals = 0

[ledgers.rewards.subjects.lot]
segments = ["customer", "grant"]
accounts = ["source", "pending", "available", "spent", "expired", "forfeited"]
origin   = ["source"]

[ledgers.rewards.entry_types]
accrue  = { from = "lot.source",    to = "lot.pending" }
release = { from = "lot.pending",   to = "lot.available" }
redeem  = { from = "lot.available", to = "lot.spent" }
expire  = { from = "lot.available", to = "lot.expired" }
forfeit = { from = "lot.pending",   to = "lot.forfeited" }
```

Tier rates, expiry schedules and what to do when a purchase is returned after its points were
released are the app's decisions (§9); the ledger only sees the resulting entries. A negative
`accrue` models a returned purchase (R4).

Naming rules:
- Ledger ids, subject kinds, segment names, account names, entry type names and params
  match `[a-z][a-z0-9_]*`.
- Segment values match `[A-Za-z0-9_.@-]+` (no `:`). Every subject has exactly its kind's
  segments; apps use a placeholder (e.g. `_`) for a segment that doesn't apply.
- Account names are `{kind}:{segment values…}:{account}`, or `{kind}:{account}` for a kind
  without segments. Subject keys are the same without the `:{account}` suffix.
- Granularity (which kinds, which segments) is the app's choice. Rules and version checks
  apply per account, so the finest declared level is where R5 holds and where concurrent
  writers can collide (R11).

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
| `post(..., dry_run = true)` | yes | runs every rule against current balances and returns the receipts it *would* produce; nothing is written |

Guardrails for agents:
- Agents can only post declared entry types (R1), so the config is also the boundary of what
  an agent can do. Changing that boundary is a config change, which goes through review:
  `apply` against production is meant to run from CI after approval, not ad hoc.
- In service mode the boundary is enforced where the credentials are: agents call the server,
  which holds the database credentials and checks grants (R19); agents never see the database.
  In library mode the app process holds the credentials and is trusted.
- Idempotency keys (R9) make agent retries safe; stable rejection codes with JSON detail (§7)
  let an agent understand and fix a refused entry.
- `dry_run` lets an agent test a batch against real balances and rules first.
- An MCP server exposing `describe`, `post` (with `dry_run`), `balance`, `history` and
  `check` as tools is a thin layer over the engine's API (later, with service mode).

## 3. Architecture

### 3.1 Engine

One Rust library crate. It:
- parses and validates config, diffs it against the stored declaration, renders `describe`;
- resolves `{type, params, amount | amounts}` into legs and enforces every rule, R1–R17;
- orders writes per ledger (§3.2) and talks to the store through the store contract (§3.3);
- defines the rejection codes (§7) and the client API (§5).

Everything above is the same code in both modes and in every store. There is no second
implementation of the rules anywhere (in particular none in SQL).

### 3.2 Write path

1. The client names the ledger in `post()`. The engine appends the post to **that ledger's
   queue** (library mode: in the app process; service mode: in the server).
2. The ledger's single writer takes posts in arrival order, possibly several at once (group
   commit).
3. It reads `(balance, seq)` of the accounts the posts touch and checks every rule in Rust.
   Each post keeps its own outcome (R10); a rejected post is answered at once with its code.
4. It commits the accepted posts in one store transaction (§3.3, §6.2).
5. Only after the store reports success does the engine acknowledge, with receipts.
   **Acknowledged means committed.**
6. If the store reports a conflict — another engine instance changed an account since it was
   read — the writer re-reads and retries (bounded). A persistent conflict answers `CONFLICT`
   and nothing is written.

The engine may keep balances in memory as a cache. The store's check (R11) makes a stale cache
safe: it costs a retry, never a wrong balance. Memory is a cache; the store is the truth.

A queue orders writes only inside the process that owns it. Several processes (two apps in
library mode, two server instances during a deploy) each have their own queue, so correctness
across processes comes from the store's check, not from the queues. A message broker is not
needed (§10).

### 3.3 Store contract

A store provides only what the engine needs:

1. **Atomic commit** — one transaction writes entries, legs and updated account rows, or
   nothing.
2. **Unique key** — `(ledger, key)` is unique across entries; a violation is reported as a
   duplicate key.
3. **Compare-and-set per account** — an account row is updated only if its `seq` still equals
   the value the engine read; a missing account may be inserted only if still missing. A
   failure is reported as a conflict.
4. **Reads** — accounts (`balance`, `seq`) by name; entries, legs and tags by the keys in §5;
   prefix scans for balances.
5. **Declarations** — each ledger's stored config, its `revision` and which entry types have
   been used; writes conditional on the expected revision.
6. **Export order** — rows can be read in an order that never skips a row that commits late
   (§6.5).
7. **`install` / `version`** — creates and migrates the schema.

Nothing else: no stored procedures, no rule logic, no triggers required.

Stores in v1:

| Store | For | Notes |
|---|---|---|
| **In-memory** | tests | process-local; runs the whole conformance suite inside `cargo test` with no database |
| **SQLite** | local development, single-machine jobs, small single-instance apps | embedded: bundled into the package and run in the engine's process, so no network hop (§3.4) |
| **Postgres** | production shared by several apps or instances (e.g. Cloud SQL) | one network hop, absorbed by group commit and batching |

Other SQL databases (MySQL, …) can be added when they meet the contract. Key-value stores
(LevelDB, DynamoDB) don't: the contract needs atomic multi-account commits and a unique key
(§10). A store is accepted when it passes the conformance suite (§8).

### 3.4 Modes

| | Library | Service |
|---|---|---|
| Where the engine runs | inside the app process (Python, Node or Rust) | in a server; clients call it over HTTP/gRPC |
| Connect with | `postgresql://…`, `sqlite:///file.db` | `https://…` |
| Database credentials | held by the app process | held by the server only |
| Permissions (R19) | not applicable: the host app is trusted | yes |
| Extra thing to run | nothing | one server |
| Posting inside the app's own transaction (R18) | Rust apps only | no |
| Fits | local development, tests, jobs, single-team apps | agent-facing and multi-team production |

The client API is identical in both modes; only `connect()` differs. Python and TypeScript
never speak SQL in either.

**Embedded SQLite.** The Python wheel and the Node addon bundle the engine and SQLite
together, so `pip install` is enough and a post is a local file commit. Limits that come from
it being a file: one machine only (not over network filesystems, not shared between
containers or pods); a short-lived container loses it unless it has a persistent volume, so
back up or replicate the file; several processes on one machine can share it, but writers take
turns (the version check, R11, still applies).

**Where the network hop goes.** Library mode removes the app-to-engine hop; with Postgres the
engine-to-database hop remains. To reduce the cost of a hop without giving up separation:
post in large batches (one round trip per batch, §9); run the server as a sidecar next to the
app (localhost or a Unix socket) or in the same region as the database; or run one service
that owns a local store. Choose by what must be separate: credentials, one shared truth, or
independent scaling.

### 3.5 Testing

The in-memory store implements the same contract as the SQL stores, so the conformance suite
(§8) runs in `cargo test` with no database. The same suite then runs against SQLite and
Postgres, and every future store.

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
entries. The engine never issues an UPDATE or DELETE on history.

**R8 Account sequence.** Each account numbers the legs applied to it `1, 2, 3, …` with no
gaps, in the order they were committed; legs of one entry that fall in the same account are
applied in leg order. An account's balance is always the sum of its legs (`to` adds, `from`
subtracts), so any balance can be recomputed from the legs alone (R15). The sequence is also
the account's version: a commit succeeds only if it was computed against the current one
(R11).

**R9 Idempotency.** Each entry carries an app-supplied `key`, unique per ledger.
- Same key, same content (type, params, amounts, ref, metadata, and `effective_at` if given):
  the original receipt is returned with `replayed = true`; nothing is written.
- Same key, different content → `DUPLICATE_KEY`.
- A batch may mix replayed and new entries; only the new ones are written.
- The same key twice within one batch → `DUPLICATE_KEY`.
- Uniqueness of `(ledger, key)` is enforced by the store, so two engine instances posting the
  same key at once write one entry and the other gets the original receipt.

**R10 Atomic batches.** On any rejection nothing from the batch is written. The error carries
the code and the index of the failing entry. When the writer groups several `post()` calls
into one store transaction (group commit), each call keeps its own all-or-nothing outcome.

**R11 Concurrency.** Correctness holds with any number of engine instances on one store.
- **Within an instance**, posts to a ledger go through that ledger's queue and a single writer,
  in arrival order (§3.2). Posts to different ledgers don't wait for each other.
- **Across instances** (several processes, a rolling deploy), the store arbitrates: a batch
  commits only if every account it touched still has the `seq` the engine read (R8). Otherwise
  the commit fails as a conflict, the engine re-reads and retries (bounded), and a persistent
  conflict is reported as `CONFLICT` with nothing written; retrying with the same key is safe.
- The engine writes touched accounts in sorted order inside the store transaction, so
  concurrent commits cannot deadlock. No ledger-wide lock exists in the store.

**R12 Rules live in the engine; the store is dumb.** Every rule is implemented once, in the
engine, and tested without a database. Any store that meets the contract (§3.3) behaves the
same. The boundary is who holds write credentials: only the engine does (the app process in
library mode, the server in service mode); whoever else can write the tables can bypass the
rules. The Postgres store adds an optional guard: triggers that reject UPDATE and DELETE on
history tables → `DIRECT_WRITE`.

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
- Each target has its own tag sequence (`tag_seq` 1, 2, 3, …), advanced with the same
  compare-and-set as R11.
- A target `entry:{key}` must exist → `UNKNOWN_TARGET`; a `ref:` target need not. Invalid name
  → `BAD_TAG`.
- Tags never affect balances or rules R1–R5.

**R17 Effective date and period close.** *(`effective_at` is stored from phase 1; period close
is implemented in phase 6.)*
- Every entry has `effective_at` (the date it belongs to) besides `created_at` (when it was
  committed, from the store's clock). Default: `effective_at = created_at`.
- `close_period(ledger, until)` appends a close record: from then on, entries with
  `effective_at < until` are rejected → `PERIOD_CLOSED`. Closes are append-only and only move
  forward.
- `balance(ledger, account, as_of)` returns the balance over entries with `effective_at < as_of`
  (point-in-time reporting).

**R18 Transaction participation.** *(Rust apps embedding the engine.)* A Rust app can pass its
own store transaction to `post()`: the app's writes and the ledger entries commit together or
not at all, with no outbox. Python and TypeScript apps, and any app in service mode, can't —
the engine owns the connection. They post after their own commit and retry with the same key
until acknowledged (R9); if a post must never be lost, they write it to an outbox in the same
transaction as their own change and let a relay post it (§9). A rejection that arrives after
the app has committed has to be compensated by the app.

**R19 Permissions.** *(Service mode.)* Callers authenticate with the server. Grants say which
ledgers, and optionally which entry types, an identity may post to, and who may read or run
`install`, `declare` and `apply` (from CI, after approval). A call outside the grants →
`PERMISSION_DENIED`. In library mode the host app holds the credentials and is trusted, so
R19 doesn't apply. The authentication mechanism is decided in phase 7.

**R20 Versioning.** `version()` returns the engine version and the store schema version
(semver). Within a major version changes are additive only: new operations, new nullable or
defaulted columns, new rejection codes. Client API signatures, rejection codes, wire formats
and rule semantics don't change. `install()` migrates the store in place and is idempotent.
Clients assert a minimum version when they connect and fail fast on a mismatch (a client-side
error, not a rejection); in service mode the check is between client and server.

## 5. Operations

Operations are methods of the engine; the client API is identical in library and service mode.

| Operation | Input | Returns |
|---|---|---|
| `install()` / `version()` | — | creates/migrates the store schema / versions |
| `declare(config, expected_revision?)` | parsed TOML | created / changed / unchanged per ledger, with the new `revision` (used by `apply`, §2a) |
| `plan(config)` / `validate(config)` / `describe(ledger)` | §2a | JSON diff / validation result / declared model |
| `post(ledger, entries[], dry_run?)` | entries: `{key, type, params, amount \| amounts, ref?, effective_at?, metadata}` | per entry: `{entry_id, legs: [{account, seq}], replayed}` |
| `entry(ledger, key)` | | the entry with its legs |
| `by_ref(ledger, ref)` | | entries with that ref, by `created_at`, then key |
| `balance(ledger, account, as_of?)` | full account name | balance |
| `balances(ledger, prefix, group_by?)` | e.g. `line:u123` or `line:u123:d456`; `group_by = "subject"` lists each subject separately | balance per account name, summed over matching subjects (or per subject) |
| `history(ledger, account)` | full account name | that account's legs, in sequence order |
| `tag(ledger, tags[])` | `{target, name, value}` | per tag: `{target, tag_seq, unchanged}`; atomic |
| `tags(ledger, target)` | `ref:{ref}` or `entry:{key}` | current value per tag name |
| `tag_history(ledger, target, name)` | | every value in `tag_seq` order |
| `changes(ledger, after_cursor?, limit?)` | an opaque cursor | every entry, leg and tag row after the cursor, plus `next_cursor`; exact, no gaps or duplicates, even with concurrent writers (export/BigQuery). The cursor never advances past a transaction that may still commit. |
| `trial_balance(ledger)` | | total per account name; sum is 0 |
| `check(ledger)` | R15 | ok / first failure |
| `close_period(ledger, until)` | R17 (phase 6) | the close record |
| `grant(...)` / `revoke(...)` | R19 (service mode) | — |

Wire formats and encoding:
- Amounts are 64-bit integers. On the wire and in the TypeScript client they travel as strings
  (or `bigint`), because JavaScript numbers can't hold 64-bit integers exactly.
- Timestamps are RFC 3339 UTC, truncated to microseconds.
- The engine canonicalises `metadata` (RFC 8785) so the content comparison of R9 is exact, and
  the store keeps those exact bytes (compressed where the store supports it; no `jsonb` copy).
  Querying on metadata happens downstream (BigQuery/DuckDB) or with an explicit cast.
- A store commits a batch with set-based statements (multi-row inserts), not a round trip per
  leg. Large imports go through `post()` in large batches; there is no bulk path that skips the
  rules.

## 6. Stores

### 6.1 Portable schema

Used by the SQLite and Postgres stores. Implementers may adjust, as long as the contract
(§3.3) and the rules hold. Rows refer to ledgers, kinds, types and accounts by integer id;
nothing derivable is stored twice.

Declarations:
- `ledgers` — `id text` PK, `no` unique, `currency`, `decimals`, `config` (as declared), `revision`, `created_at`
- `subject_kinds` — `(ledger_no, kind_no)` PK, `name`, `segments`
- `account_types` — `(ledger_no, kind_no, account_no)` PK, `name`, `origin`
- `entry_types` — `(ledger_no, type_no)` PK, `name`, `legs` (resolved patterns), `used`

State (updated on every commit):
- `accounts` — `id` PK, `ledger_no`, `kind_no`, `subject text` (e.g. `line:u123:d456:c789`), `account_no`, `balance`, `seq`; unique `(ledger_no, subject, account_no)`. `seq` is the version of R8/R11. Names are derived (`subject || ':' || name`), not stored.
- `tag_heads` — PK `(ledger_no, target)`, `tag_seq`

History (append-only):
- `entries` — `id` PK, `ledger_no`, `key`, `type_no`, `ref` (indexed), `effective_at`, `created_at`, `metadata`; unique `(ledger_no, key)`
- `legs` — PK `(entry_id, leg_index)`, `account_id`, `role` (0/1), `amount`, `account_seq`; unique `(account_id, account_seq)`
- `tags` — PK `(ledger_no, target, tag_seq)`, `name`, `value`, `created_at`; index `(ledger_no, target, name, tag_seq desc)`
- `period_closes` — PK `(ledger_no, close_no)`, `until`, `closed_at` (R17, phase 6)

Grants (R19) are added with service mode.

### 6.2 Commit

One store transaction per group of accepted posts:
1. Insert the entries; a violation of the unique `(ledger_no, key)` is a duplicate key.
2. Insert the legs with their `account_seq`.
3. For each touched account, in sorted order: update `balance` and `seq` where `seq` equals the
   value the engine read; for an account that did not exist, insert it. No row updated, or an
   insert that finds the row already there, is a conflict: roll back.
4. Commit, then acknowledge.

Reference point: pgledger reports about 743 bytes per transfer; measure and publish the real
figure (test 28).

### 6.3 Postgres

Requires PostgreSQL 13 or later. Extras on top of the portable schema: `fillfactor = 80` on
`accounts` and `tag_heads` (updates stay HOT), compressed `metadata` (`lz4`), `xid8` columns on
`entries` and `tags` for the export cursor (§6.5), and the optional append-only triggers (R12).
Only the engine's role has table privileges. A database superuser can still rewrite rows; v1
does not detect that (§10), so production access to the database is the control.

### 6.4 SQLite

Embedded, bundled into the Python and Node packages (§3.4). One writer at a time, so id order
is commit order. File access is the boundary; back up or replicate the file.

### 6.5 Export cursor

`changes()` must not skip rows that commit out of order. In Postgres, rows record the id of the
transaction that wrote them (`xid`); the cursor is `(xid, id)` and an export only reads rows
whose `xid` is below the lowest still-running transaction, so a row that is not yet visible can
never be passed over. In SQLite, writers are serialized, so id order is enough.

## 7. Clients

- **Rust crate** `chainledger`: the engine, the store trait, the in-memory, SQLite and Postgres
  stores, typed errors, and the conformance harness (§8) so any store can be tested the same
  way. A `chainledger-server` binary wraps the engine for service mode.
- **Python package** (PyO3 + maturin): library mode, with the engine and SQLite bundled; the
  same package has an HTTP/gRPC client for service mode.
- **TypeScript package**: a native Node addon (napi-rs) for library mode; an HTTP/gRPC client
  for service mode. Amounts are `bigint` or strings. (WebAssembly doesn't fit library mode,
  because the engine owns the database connection.)

Tooling for the bindings is confirmed when each is built (phases 3 and 6).

```python
from chainledger import Client, Entry, Tag

cl = Client.connect(DATABASE_URL)        # postgresql://… or sqlite:///dev.db → library mode
# cl = Client.connect("https://ledger.internal", token=TOKEN)   # service mode, same API
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
cl.post("compi", batch)                  # returns once the store has committed

cl.post("stripe", [
    Entry(key="ch_1:charge", type="charge", params={"charge": "ch_1"},
          amounts=[100_00, 97_00, 3_00]),
    Entry(key="ch_1:settle", type="settle", params={"charge": "ch_1", "payout": "po_9"},
          amount=97_00),
])
cl.tag("compi", [Tag(target="ref:line-1", name="quota_period_name", value="FY26 Q3")])
assert cl.check("compi").ok
```

Rejection codes: `UNKNOWN_LEDGER`, `UNKNOWN_TYPE`, `BAD_PARAMS`, `BAD_AMOUNTS`, `BAD_AMOUNT`,
`ZERO_AMOUNT`, `MIXED_SIGNS`, `UNBALANCED`, `OVER_RELEASE`, `DUPLICATE_KEY`, `DIRECT_WRITE`,
`INCOMPATIBLE_CHANGE`, `BAD_CONFIG`, `STALE_PLAN`, `PERMISSION_DENIED`, `UNKNOWN_TARGET`,
`BAD_TAG`, `PERIOD_CLOSED`, `CONFLICT`. Every error carries `{code, index, detail}`, surfaced as
a Rust error type, a Python exception, a TypeScript error, or an HTTP error body.
`CONFLICT` is transient: nothing was written, and retrying with the same key is safe.

## 8. Conformance suite

The acceptance tests are the conformance suite: one set of tests, run against every store.
A store is accepted when it passes the tiers that apply to it.

**Tier A — every store** (in-memory in `cargo test`; SQLite and Postgres against a local
database):

01. compi happy path: accrue → earn → to_payroll → clear; balances and trial balance correct.
02. Negative delta flows through the same types; accounts may end negative.
03. R5: every row of the R5 table behaves as stated; origin accounts are exempt.
04. Stripe: `charge` with three legs, `settle` across subjects (charge → payout), `pay_out`;
    balances per subject correct; the settle entry appears in the history of accounts of both
    subjects.
05. `UNKNOWN_TYPE`, `BAD_PARAMS`, `BAD_AMOUNTS`, `UNBALANCED`, `MIXED_SIGNS`, `ZERO_AMOUNT`.
06. Idempotency: replay returns the same receipt with `replayed = true`; changed content →
    `DUPLICATE_KEY`; duplicate key within a batch → `DUPLICATE_KEY`.
07. Atomicity: a batch with one bad entry writes nothing; a failed store commit leaves balances
    and sequences unchanged; nothing is acknowledged before the store has committed.
08. Queue: many concurrent posts to one ledger commit in arrival order, in fewer store
    transactions than posts; a rejected post does not affect its neighbours in the same group;
    posts to different ledgers do not wait for each other.
09. N concurrent writers (threads) on one account → no sequence gaps, correct balances; two
    writers that each try to release the account's whole balance never over-release.
10. `balances` by prefix sums correctly at each level; `group_by = "subject"` lists each
    subject.
11. `declare` is idempotent and bumps `revision` only on change; incompatible changes are
    rejected; `BAD_CONFIG` for invalid types; `declare` with a stale expected revision →
    `STALE_PLAN`.
12. `validate` catches invalid config offline; `plan` reports adds and incompatible changes
    without writing; `apply` with a stale `plan_hash` → `STALE_PLAN`; `describe` round-trips
    the config.
13. `post(dry_run = true)` returns the same receipts/rejections as a real post and writes
    nothing.
14. `effective_at` defaults to `created_at` and is stored.
15. `check` passes on a healthy ledger and reports the first account whose stored balance or
    sequence disagrees with its legs.
16. One batch touching 50,000 distinct accounts succeeds on default settings.
17. `version()` reports the versions; `install()` on an existing store is idempotent and keeps
    data; a client requiring a higher version fails fast.

**Tier D — stores on a shared database** (SQLite, Postgres), with several engine instances or
processes at once:

18. The races of test 09, across instances: no sequence gaps, correct balances, R5 never
    violated.
19. The same key posted from two instances at once → one entry written, the other replayed.
20. A forced version conflict is retried transparently and succeeds; a persistent conflict is
    reported as `CONFLICT` with nothing written; overlapping batches in opposite account
    orders never deadlock.

**Tier S — Postgres:**

21. The append-only guard: UPDATE and DELETE on history tables are rejected (`DIRECT_WRITE`).

**Tier M — service mode:**

22. Callers authenticate; a caller can post only to its granted ledgers and entry types
    (`PERMISSION_DENIED` otherwise); a reader can't post.
23. Client parity: the same corpus of configs, entries and amounts, run through library mode and
    service mode, in Python and in TypeScript, gives identical receipts and rejections.

**Tier T — Rust apps embedding the engine:**

24. An app write plus `post()` in one store transaction commit together; rolling back the app
    transaction leaves no entries.

**Export and tags** (phase 5):

25. `changes(after_cursor)` returns every entry, leg and tag row after the cursor, none twice,
    none missing, with concurrent writers that commit out of order.
26. Tags: a new value appends and becomes current; history keeps both; the same value again is
    a no-op; `ref:` tags apply to entries posted later with that ref; `UNKNOWN_TARGET`,
    `BAD_TAG`.

**Periods** (phase 6):

27. After `close_period(until)`, entries with `effective_at < until` → `PERIOD_CLOSED`; later
    ones succeed; closes only move forward; `balance(as_of)` correct.

**Measurements** (reported per store, not pass/fail):

28. Average bytes per leg, excluding metadata.
29. Posts per second for single-entry posts and for large batches, with one and with several
    writers. Store choices are made with these numbers.

## 9. Design guidelines

- **Apps decide what; the ledger decides whether.** An app computes "this line earned 5,000"
  and posts `earn`; the ledger accepts or rejects it with a stable code. The app never touches
  balances directly.
- **Keep domain logic out of the engine.** Quota periods, eligibility, reward tiers and expiry
  schedules live in the app, and in refs, tags and metadata — never in the engine or a store.
- **Post in batches.** A round trip, and a commit, is paid per batch. A run that posts 50,000
  entries should be one `post()` call (or a few large ones), not 50,000.
- **Acknowledged means committed.** Apps that act on an acknowledgement (paying out, telling a
  customer) can rely on it. Apps that don't get one (timeout, crash) retry with the same key.
- **Posting around the app's own commit.** Post after the app's own commit and retry until
  acknowledged. If the post must never be lost, write it to an outbox table in the same
  transaction as the app's change and let a relay post it; keys make the retries safe. Only
  Rust apps can post inside their own transaction (R18).
- **Keep subjects fine-grained.** Writers to one ledger are ordered by the engine, so the cost
  of a hot account shows up as version conflicts when several processes write it. Avoid an
  account that every entry touches, and put `source` accounts inside the subject (as compi and
  the rewards ledger do).
- **Corrections are new entries** (R7). After a period close, an adjustment is an entry with
  `effective_at` in the open period and a `ref` pointing back to the original.
- **Cross-ledger flows are the app's job.** Ledgers are independent; the app posts to each and
  links the entries with a shared `ref`.
- **Models and scenarios.** A model run is the output of a computation, not something to edit.
  Use one ledger per scenario or run, with idempotency keys that include a hash of the
  assumptions so reruns are safe. Forking a ledger is copy-and-replay in v1.
- **Own the integrity story in the app.** Until the engine offers an integrity layer (§10),
  apps that need tamper evidence add their own (e.g. a signed export of `changes()`).

## 10. Not in v1

Currency conversion/FX (currency is a label) · pending/held transfers (model them as
accounts/entry types if needed) · chart of accounts, account kinds, financial statements
(that's the accounting system's job) · **integrity proofs**: hash chains, seals, Merkle
roll-ups, external anchoring and the immutable Parquet archive were removed from v1 — because
entries are immutable and ordered per account, an optional integrity layer can be added later
without changing the model · **message broker / external queue**: queues are in the engine,
and the store's version check keeps several instances correct · key-value stores (LevelDB,
DynamoDB): the contract needs atomic multi-account commits, a unique key and compare-and-set ·
**Turso Database (Rust SQLite rewrite)** as a store: a candidate, but at the time of writing
its docs list MVCC as not production ready, no multi-process access, and indexes as
experimental, while the contract needs a unique index (R9); adopt it if and when it passes the
conformance suite · MySQL and other SQL stores until needed · stores backed by hosted ledger
services (Formance, Fragment) · stored procedures or rule logic in any database · a
WebAssembly client · summary journals for the GL, e.g. NetSuite (planned) · scenario branching
(fork without replay).

## 11. Build phases

Each phase is usable on its own.

1. **Engine and in-memory store (Rust)** — config parsing and validation; `declare`, `plan`,
   `apply`, `describe`; `post` with R1–R14, the per-ledger queue and group commit, `dry_run`,
   `effective_at` stored; reads, `check`, `version`; the conformance suite (Tier A, tests
   01–17), all in `cargo test` with no database.
2. **Postgres store** — portable schema, commit with compare-and-set, `install`, append-only
   guard, the Rust embedding path (R18); Tier A on Postgres, Tier D, S and T (tests 18–21, 24),
   measurements (28–29).
3. **Python package and first consumer: compi** — PyO3 library mode with SQLite bundled; replay
   a month of real compi history through it against Postgres; compare balances with today's
   BigQuery totals. Validates the model before building further.
4. **SQLite store** — embedded local development and single-machine jobs; Tier A and D,
   measurements.
5. **Export and tags** — `changes()` with its cursor, tags (tests 25–26).
6. **TypeScript package and second consumer** — native Node addon; a financial-model flow
   (e.g. an ARR bridge or revenue schedule) validating negative deltas, ledger-per-scenario
   and agent-proposed entries (`describe`, `dry_run`); period close (R17, test 27).
7. **Service mode** — server binary, wire API, authentication and grants (R19), client parity
   (tests 22–23); the MCP server on top.
