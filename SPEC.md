# chainledger — specification v1

A double-entry engine for finance-engineering processes that sit next to the accounting
books: compensation (compi), payouts, expenses, revenue schedules, financial models — any
process where money (or any amount) moves through states and must be traceable and
tamper-evident.

Apps declare their ledgers in config (subjects, accounts, entry types) and post entries;
Postgres enforces every rule. chainledger is a library, not a service: it installs its
schema into a Postgres database you point it at (e.g. Cloud SQL) and talks to it over SQL.

Status: agreed design, not yet implemented. Build order: §10.

---

## 1. Concepts

| Term | Meaning |
|---|---|
| **Ledger** | One independent book, e.g. `compi` or `stripe`: its own subjects, accounts, entry types, chains and seals. Carries a currency label. |
| **Subject kind** | A kind of thing money belongs to, with a fixed list of segments, e.g. `line = [owner, deal, component]`, `charge = [charge]`, or `stripe = []` (a single subject). |
| **Subject** | One concrete thing of a kind: `line:u123:d456:c789`, `charge:ch_1`, `stripe`. |
| **Account** | A named balance of one subject, declared per subject kind: `line:u123:d456:c789:eligible`, `charge:ch_1:balance`, `stripe:fees`. |
| **Origin account** | An account where money enters the ledger (e.g. compi's `source`, Stripe's `gross`). It may go negative from sending (R5). |
| **Entry type** | A declared shape of movement: which accounts an entry debits/credits. The only movements apps can post (R1). Replaces fixed routes: a linear route is just a set of two-leg types. |
| **Entry** | One posted event of an entry type: two or more **legs**, each moving an amount out of (`from`) or into (`to`) one account. Legs may belong to different subjects. |
| **Chain** | The numbered, hash-linked sequence of postings of one subject. Each leg is recorded in the chain of its account's subject. |
| **Posting** | One chain record: the legs of one entry that fall in that chain, plus the digest of the whole entry. A same-subject entry is one posting; an entry across two subjects is two postings, one per chain. |
| **Roll-up** | Higher levels (owner + deal, owner, subject kind, the whole ledger) aggregate the level below: balances by sum, integrity by hashing children's hashes (a Merkle tree shaped like the subject hierarchy). |
| **Seal** | A checkpoint of the whole ledger: the roll-up's root hash, linked to the previous seal. |
| **Batch** | The entries of one `post()` call. All are written or none. |
| **Ref** | Optional app-defined string on an entry grouping related entries, e.g. compi's `compi_line_id`. Part of the entry digest. |
| **Tag** | A named text value attached to a ref or to one entry, e.g. `quota_period_name = "FY26 Q3"`. Append-only; the latest value wins. Doesn't touch balances or entry hashes. |

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

Optional archive (R18), per ledger:

```toml
[ledgers.compi.archive]
bucket    = "gs://chainledger-archive"   # bucket with a locked retention policy
keep_days = 90                           # sealed + archived entries older than this may be pruned from Postgres
```

Naming rules:
- Ledger ids, subject kinds, segment names, account names, entry type names and params
  match `[a-z][a-z0-9_]*`.
- Segment values match `[A-Za-z0-9_.@-]+` (no `:`). Every subject has exactly its kind's
  segments; apps use a placeholder (e.g. `_`) for a segment that doesn't apply.
- Account names are `{kind}:{segment values…}:{account}`, or `{kind}:{account}` for a kind
  without segments. Chain keys are the same without the `:{account}` suffix.
- Granularity (which kinds, which segments) is the app's choice. Rules apply per account,
  so the finest declared level is where R5 holds; chains always follow full subjects.

## 2a. Declared as code (plan / apply), built for agents

Ledgers are infrastructure-as-code, like Terraform: the TOML lives in the app's git repo,
changes go through review, and the database is brought in line with it. Humans and AI
agents use the same workflow.

| Command (CLI and client API) | Needs DB | What it does |
|---|---|---|
| `validate(config)` | no | checks the config offline (naming, patterns, placeholders, R14 shape rules) |
| `plan(config)` | read-only | diffs config against the database: what `apply` would add, and anything it would reject as `INCOMPATIBLE_CHANGE`; machine-readable JSON plus a human summary |
| `apply(config, plan_hash?)` | yes | runs `declare`; with `plan_hash`, refuses if the database changed since that plan (like a saved Terraform plan) |
| `describe(ledger)` | read-only | the declared model as JSON: subject kinds, accounts, entry types with their params and legs — what an agent may post |
| `post(..., dry_run = true)` | yes | runs every rule and returns the receipts it *would* produce, then rolls back; nothing is written |

Guardrails for agents:
- Agents can only post declared entry types (R1), so the config is also the boundary of
  what an agent can do. Changing that boundary is a config change, which goes through
  review: `apply` against production is meant to run from CI after approval, not ad hoc.
- Idempotency keys (R9) make agent retries safe; stable rejection codes with JSON detail
  (§7) let an agent understand and fix a refused entry.
- `dry_run` lets an agent test a batch against real balances and rules first.
- Per-identity permissions (which ledgers and entry types a caller may post, and who may
  `apply`) are planned (§9).
- An MCP server exposing `describe`, `post` (with `dry_run`), `balance`, `history` and
  `verify` as tools is a thin layer over the client API (later).

## 3. Rules

**R1 Declared entry types only.** Every entry names an entry type of its ledger and gives
the type's params. Its legs are exactly the type's legs, with params substituted.
→ `UNKNOWN_TYPE`, `BAD_PARAMS` (missing, extra, or invalid segment value)

**R2 Legs.** Each leg has a role (`from` or `to`), an account and an amount.
- A `from` leg lowers the account's balance by its amount; a `to` leg raises it.
- An entry has at least one `from` and one `to` leg.
- Amounts are given per leg (`amounts`, in the type's leg order), or as one `amount` for a
  type with exactly one `from` and one `to` leg. → `BAD_AMOUNTS`

**R3 Balanced.** The `from` amounts of an entry sum to its `to` amounts. So the sum of all
account balances of a ledger is always 0 (trial balance). → `UNBALANCED`

**R4 Signed integer amounts.** Amounts are non-zero integers in minor units, range of a
signed 64-bit integer. No floats or decimals anywhere. All legs of one entry have the same
sign: positive is a normal movement, negative is a reduction flowing the same way (e.g. a
negative commission delta). → `ZERO_AMOUNT`, `BAD_AMOUNT`, `MIXED_SIGNS`

**R5 No over-release.** For every non-origin account that is a `from` leg with a positive
amount anywhere in a batch, its balance after the batch must be ≥ 0. Accounts that only
send negative amounts, or only receive, may end negative. Origin accounts are exempt.
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

**R7 Append-only.** Entries, legs and postings are never updated or deleted. Corrections
are new entries. The only exception is `prune()` (R18): removing from Postgres entries that
are sealed, archived and verified in immutable storage.

**R8 Chains and postings.**
- Each subject has one chain. Every leg is recorded in the chain of its account's subject.
- An entry produces one **posting** per chain it touches, containing its legs in that chain.
- Each chain numbers its postings `1, 2, 3, …` with no gaps.
- Every entry has an `entry_digest` over all its content and all its legs (§5). Each
  posting's hash covers the entry digest, the indices of its legs, and the previous
  posting's hash. So every chain an entry touches commits to the whole entry: changing
  either side of a cross-subject transfer breaks the other side's check too.

**R9 Idempotency.** Each entry carries an app-supplied `key`, unique per ledger.
- Same key, same content (type, params, amounts, ref, metadata, and `effective_at` if
  given): the original receipt is returned with `replayed = true`; nothing is written.
- Same key, different content → `DUPLICATE_KEY`.
- A batch may mix replayed and new entries; only the new ones are written.
- The same key twice within one batch → `DUPLICATE_KEY`.

**R10 Atomic batches.** On any rejection nothing from the batch is written. The error
carries the code and the index of the failing entry.

**R11 Parallel writers.** `post()` locks only the chains its batch touches, by row-locking
their `heads` rows (`SELECT … FOR UPDATE`, sorted by chain id, after creating missing heads
with `INSERT … ON CONFLICT DO NOTHING`), so writers never deadlock. Row locks, not advisory
locks: advisory locks use Postgres's fixed-size shared lock table, which a batch touching
thousands of chains would exhaust. Writers wait for each other only when they touch a common
chain; everything else runs in parallel. No ledger-wide lock exists on the write path.

**R12 Postgres enforces everything.** All rules run inside Postgres (`post()` plus triggers).
Any INSERT, UPDATE, DELETE or TRUNCATE on chainledger tables outside chainledger's own
functions is rejected. → `DIRECT_WRITE`. Client libraries only make calling the SQL
convenient (they may pre-check for friendlier errors; Postgres has the final say).

**R13 Accounts from config.** Accounts and chains are created on first use when an entry
type's legs resolve to them. Nothing else creates accounts. → `UNKNOWN_LEDGER`

**R14 Declare is idempotent.** `declare(config)` creates or updates ledgers.
- Adding subject kinds, accounts to a kind, or entry types: allowed.
- Changing a ledger's currency or decimals, a kind's segments or origin accounts, removing
  or renaming anything, or changing an entry type that has been used →
  `INCOMPATIBLE_CHANGE`. (To change a used entry type, declare a new one.)
- Invalid config (unknown accounts in a type, placeholders not covering a pattern's
  segments, a type with no `from` or no `to` leg) → `BAD_CONFIG`.

**R15 Verify.** Recomputes from the stored entries alone: entry digests, posting hashes and
sequence (no gaps), account balances, roll-up hashes, and the trial balance (= 0). Returns
ok, or the first broken `(chain, seq)`, entry or node and what failed. Chains are
independent, so every mode verifies them in parallel.
- `verify(ledger)` — **incremental, the routine check**: each chain that changed since the
  last seal, from its sealed head `(seq, hash, balances)` to its current head. Cost is
  proportional to postings since the last seal, not to history.
- `verify(ledger, prefix)` — **one subtree**, e.g. one owner: its chains from genesis, then
  its roll-up hash against the last seal.
- `verify(ledger, full = true)` — **everything** from genesis, plus every seal. An audit,
  e.g. before a period close; optional with an archive (R18).
Tag chains (R16) are verified the same way.

**R16 Tags.** Values that change over time without touching entries (e.g. a line's quota
period being renamed).
- A tag is `{target, name, value}`. `target` is `ref:{ref}` (every entry with that ref,
  past and future) or `entry:{key}` (one entry). `name` matches `[a-z][a-z0-9_]*`; `value`
  is text.
- Append-only: setting a tag again appends a new row; the current value of `(target, name)`
  is the one with the highest `tag_seq`. History is kept.
- Appending a value equal to the current one is a no-op, so refreshing tags every run is
  idempotent.
- Each target has its own tag chain (`tag_seq` 1, 2, 3, … and hashes, §5), locked by
  row-locking its `tag_heads` row like R11.
- A target `entry:{key}` must exist → `UNKNOWN_TARGET`; a `ref:` target need not.
  Invalid name → `BAD_TAG`.
- Tags never affect balances, rules R1–R5, or entry hashes.

**R17 Roll-up and seals.**
- A **leaf** hash covers one chain: its head `(seq, hash)` and its accounts' balances.
- A **node** hash covers one prefix of the subject hierarchy (`line:u123:d456`, then
  `line:u123`, then `line`): the hashes of its children, sorted by key. The **root** covers
  the ledger. Integrity rolls up exactly like balances do. Encoding in §5.
- Roll-up hashes are computed at seal time, not on every post, so nothing shared is
  updated on the write path (keeps R11 parallel).
- `seal(ledger)` reads all chain heads at one consistent snapshot (it doesn't block
  writers), runs incremental verify up to that snapshot, then appends
  `{seal_no, sealed_at, entries_root, tags_root, prev_seal_hash, hash}` and stores the leaf
  and node hashes that changed. Seals are append-only and hash-chained; one `seal()` per
  ledger runs at a time.
- Run `seal()` on a schedule, e.g. after each app run or daily. Seal hashes are what later
  gets anchored outside the database.

**R18 Archive (optional, per ledger).** Makes sealed history immutable even for database
admins, and keeps Postgres small. Enabled by `[ledgers.<id>.archive]`.
- `archive(ledger)` (Rust client) writes each sealed-but-unarchived seal's `changes()` as
  Parquet (§6a) to `{bucket}/{ledger}/seal={seal_no:010}/` with create-only preconditions
  (`ifGenerationMatch=0`), so nothing is ever overwritten.
- It re-reads the uploaded files, recomputes their hashes against the seal's roll-up, and
  only then calls `mark_archived(ledger, seal_no, manifest_sha256)`. A failed check
  archives nothing.
- The bucket must have a **locked** retention policy (≥ the legal retention period).
- `prune(ledger)` deletes from Postgres the entries, legs, postings and tags of archived
  seals older than `keep_days`. Heads, accounts, seals and seal nodes are never pruned, so
  posting, R5 and incremental verify never need the archive.
- `verify(full)` reads archived seals from Parquet and the rest from Postgres.
- Without an archive, R7 holds in full and protection against admins is detection (R15).

**R19 Effective date and period close.** *(Designed now; `effective_at` is stored from
phase 1, period close is implemented in phase 6.)*
- Every entry has `effective_at` (the date it belongs to) besides `created_at` (when it
  was posted, from the database clock). Default: `effective_at = created_at`. Both are part
  of the entry digest.
- `close_period(ledger, until)` appends a close record: from then on, entries with
  `effective_at < until` are rejected → `PERIOD_CLOSED`. Closes are append-only, only move
  forward, and are included in the next seal.
- `balance(ledger, account, as_of)` returns the balance over entries with
  `effective_at < as_of` (point-in-time reporting).

## 4. Operations

All are SQL functions in schema `chainledger`; the Rust client and bindings wrap them.

| Operation | Input | Returns |
|---|---|---|
| `install()` | — | creates/upgrades the schema (versioned migrations) |
| `declare(config)` | parsed TOML | created / changed / unchanged per ledger (used by `apply`, §2a) |
| `plan(config)` / `describe(ledger)` | §2a | JSON diff / declared model |
| `post(ledger, entries[])` | entries: `{key, type, params, amount \| amounts, ref?, effective_at?, metadata}` | per entry: `{entry_id, digest, postings: [{chain, seq, hash}], replayed}` |
| `entry(ledger, key)` | | the entry with its legs and postings |
| `by_ref(ledger, ref)` | | entries with that ref, by `created_at`, then key |
| `balance(ledger, account, as_of?)` | full account name | balance |
| `balances(ledger, prefix)` | e.g. `line:u123` or `line:u123:d456` | balance per account name, summed over matching subjects |
| `history(ledger, account)` | full account name | that account's legs, in chain order |
| `chain(ledger, chain, from_seq?)` | e.g. `charge:ch_1` | the chain's postings in order |
| `tag(ledger, tags[])` | `{target, name, value}` | per tag: `{target, tag_seq, hash, unchanged}`; atomic |
| `tags(ledger, target)` | `ref:{ref}` or `entry:{key}` | current value per tag name |
| `tag_history(ledger, target, name)` | | every value in `tag_seq` order |
| `seal(ledger)` | | `{seal_no, sealed_at, entries_root, tags_root, hash}` |
| `changes(ledger, after_seal, upto_seal)` | seal numbers | every entry, leg, posting and tag between the two seals; exact, no gaps or duplicates (export/BigQuery) |
| `trial_balance(ledger)` | | total per account name; sum is 0 |
| `verify(ledger, prefix?, full?)` | R15 | ok / first failure |
| `close_period(ledger, until)` | R19 (phase 6) | the close record |
| `archive(ledger)` | Rust client; R18 | seals archived |
| `mark_archived(ledger, seal_no, manifest_sha256)` | called by `archive()` | — |
| `prune(ledger)` | R18 | rows removed from Postgres |

Wire formats inside SQL:
- Amounts travel as JSON **strings** of integers (e.g. `"-3000"`); stored as `bigint`.
- Timestamps are RFC 3339 UTC, truncated to microseconds.
- `metadata` travels as a JSON **string** containing canonical JSON (RFC 8785), produced by
  the client. Postgres stores those exact bytes, compressed; no `jsonb` copy. Querying on
  metadata happens downstream (BigQuery/DuckDB) or with an explicit `metadata::jsonb` cast.
- `post()` processes the whole batch with set-based SQL (`unnest` over arrays), not a loop
  per entry. Large imports go through `post()` in large batches; there is no bulk path that
  skips the rules.

## 5. Hash encoding

Primitives: `str(x)` = u32 big-endian byte length + UTF-8 · `u8/u16/u32/u64(x)` = unsigned
big-endian · `i64(x)` = signed big-endian (two's complement) · `ts(x)` = `str` of RFC 3339
UTC with microseconds, e.g. `2026-10-30T14:03:22.123456Z`.

**Entry digest** — covers the whole entry, legs in type order:

```
entry_digest = sha256(0x10 || str(ledger) || str(key) || str(type) || str(ref or "")
                     || ts(effective_at) || ts(created_at) || str(metadata)
                     || u32(n_legs)
                     || for each leg i: u16(i) || u8(role: 0 = from, 1 = to) || str(account) || i64(amount))
```

**Posting hash** — one per chain an entry touches:

```
posting_hash = sha256(0x11 || str(ledger) || str(chain) || u64(seq) || entry_digest
                     || u32(k) || u16(leg index) for each of the entry's k legs in this chain, ascending
                     || prev_hash)
```

A chain's first `prev_hash` is 32 zero bytes. Postgres computes digests and hashes in
`post()` (built-in `sha256()`); any client can recompute them from stored data.

**Tags**, per target chain:
`tag_hash = sha256(0x12 || str(ledger) || str(target) || u64(tag_seq) || str(name) || str(value) || ts(created_at) || prev_tag_hash)`,
first `prev_tag_hash` = 32 zero bytes.

**Roll-up** (R17):
- `leaf = sha256(0x00 || str(chain) || u64(seq) || head_hash || balances)`, where
  `balances` is `str(account) || i64(balance)` for each of the chain's accounts, sorted by
  account name (bytewise). A chain with no postings is not a leaf.
- `node = sha256(0x01 || str(prefix) || child_1 || … || child_n)`, children sorted by key
  (bytewise). Levels: root `""` → subject kind (`line`) → each segment prefix
  (`line:u123`, `line:u123:d456`) → chains (leaves). A kind without segments has its single
  chain as its only child.
- `tags_root = sha256(0x01 || str("") || tag_leaf_1 || … )`, tag leaves sorted by target,
  `tag_leaf = sha256(0x00 || str(target) || u64(tag_seq) || head_hash)`.
- `seal_hash = sha256(0x02 || str(ledger) || u64(seal_no) || ts(sealed_at) || entries_root || tags_root || prev_seal_hash)`,
  first `prev_seal_hash` = 32 zero bytes. (Period closes, R19, are added to the seal
  encoding when implemented.)

## 6. Storage (reference schema)

Implementers may adjust, as long as rules and operations hold. Storage goals: rows refer to
ledgers, kinds, types, chains and accounts by integer id; hashes are 32-byte `bytea`;
nothing derivable is stored twice. Target: ~150–250 bytes per leg plus compressed metadata.

Declarations:
- `ledgers` — `id text` PK, `no smallint` unique, `currency`, `decimals`, `config jsonb` (as declared), `created_at`
- `subject_kinds` — `(ledger_no, kind_no)` PK, `name`, `segments text[]`
- `account_types` — `(ledger_no, kind_no, account_no)` PK, `name`, `origin bool`
- `entry_types` — `(ledger_no, type_no)` PK, `name`, `legs jsonb` (resolved patterns), `used bool`

State (updated on every post; `fillfactor = 80` so updates stay HOT):
- `chains` — `id bigint` PK, `ledger_no`, `kind_no`, `key text`; unique `(ledger_no, key)`
- `heads` — `chain_id` PK, `seq`, `hash` (its row lock is the chain lock, R11)
- `accounts` — `id bigint` PK, `chain_id`, `account_no smallint`, `balance bigint`;
  unique `(chain_id, account_no)`. Names are derived (`chains.key || ':' || name`), not stored.

History (append-only):
- `entries` — `id bigint` PK, `ledger_no`, `key text`, `type_no`, `ref text` (indexed),
  `effective_at`, `created_at`, `metadata text` (compression `lz4`), `digest bytea`;
  unique `(ledger_no, key)`
- `legs` — PK `(entry_id, leg_index)`, `account_id`, `role` (0/1), `amount bigint`;
  index `(account_id, entry_id)`. The chain is the account's chain, not stored.
- `postings` — PK `(chain_id, seq)`, `entry_id`, `hash`; index `entry_id`. Which legs a
  posting holds is derived (legs of `entry_id` whose account is in `chain_id`); `prev_hash`
  is the previous posting's `hash`.
- `tag_heads` — PK `(ledger_no, target)`, `tag_seq`, `hash` (state, `fillfactor = 80`)
- `tags` — PK `(ledger_no, target, tag_seq)`, `name`, `value`, `created_at`, `hash`;
  index `(ledger_no, target, name, tag_seq desc)`
- `seals` — PK `(ledger_no, seal_no)`, `sealed_at`, `entries_root`, `tags_root`, `prev_hash`, `hash`
- `seal_nodes` — PK `(ledger_no, seal_no, kind, key)`: leaves (`chain`, `seq`, head
  `hash`), nodes (prefix) and tag leaves that changed at that seal, with their hash. A
  node's value at seal N is its latest row at or before N.
- `period_closes` — PK `(ledger_no, close_no)`, `until`, `closed_at` (R19, phase 6)
- `archives` — PK `(ledger_no, seal_no)`, `uri`, `manifest_sha256`, `archived_at` (R18)

Guards:
- Triggers on all tables reject writes unless `current_setting('chainledger.writing', true) = 'on'`,
  which only chainledger's functions set (`set_config(..., true)`, transaction-local).
- History tables reject every UPDATE, and every DELETE except from `prune()` on archived
  seals (R18).

Note: a database superuser can still disable triggers and rewrite rows. That is detectable
(R15 against the seals; later, external anchoring of seal hashes), not preventable — unless
the history is archived (R18).

## 6a. Archive format (R18)

Per seal, under `{bucket}/{ledger}/seal={seal_no:010}/`, zstd-compressed Parquet:

- `entries.parquet` — `ledger` · `key` · `type` · `ref` (nullable) · `effective_at` ·
  `created_at` (timestamps µs UTC) · `metadata` (canonical JSON bytes) · `digest` (fixed_size_binary(32))
- `legs.parquet` — `ledger` · `key` · `leg_index` · `role` · `account` · `chain` · `amount` int64
- `postings.parquet` — `ledger` · `chain` · `seq` int64 · `key` · `hash` (fixed_size_binary(32)),
  sorted by `(chain, seq)`
- `tags.parquet` — `ledger` · `target` · `tag_seq` · `name` · `value` · `created_at` · `hash`
- `manifest.json` — the seal record, the sha256 of each file, row counts, and the
  `(seq, hash)` each touched chain starts from, so a segment verifies on its own.

Full names are written out (not internal ids), so the archive can be read and verified
with DuckDB/BigQuery and no Postgres; §5 hashes recompute from these columns alone.

## 7. Client libraries

- **Rust crate** `chainledger`: config parsing (TOML) and validation, RFC 8785 metadata
  canonicalisation, typed errors from rejection codes, sync Postgres driver, an independent
  `verify` that recomputes hashes client-side across threads/connections, and `archive()` /
  `prune()` (R18; Parquet via `parquet`/`arrow`, GCS via `object_store`).
- **Python package** (PyO3 + maturin), same API:

```python
from chainledger import Client, Entry, Tag

cl = Client.connect(DATABASE_URL)
cl.install()
cl.declare_file("chainledger.toml")

line = {"owner": "u123", "deal": "d456", "component": "c789"}
cl.post("compi", [
    Entry(key="line-1:accrue:1", type="accrue", params=line, amount=5_000_00,
          ref="line-1", metadata={"comment": "..."}),
])
cl.post("stripe", [
    Entry(key="ch_1:charge", type="charge", params={"charge": "ch_1"},
          amounts=[100_00, 97_00, 3_00]),
    Entry(key="ch_1:settle", type="settle", params={"charge": "ch_1", "payout": "po_9"},
          amount=97_00),
])
cl.tag("compi", [Tag(target="ref:line-1", name="quota_period_name", value="FY26 Q3")])
cl.seal("compi")
assert cl.verify("compi").ok
```

- Node bindings: later, not v1.

Rejection codes: `UNKNOWN_LEDGER`, `UNKNOWN_TYPE`, `BAD_PARAMS`, `BAD_AMOUNTS`, `BAD_AMOUNT`,
`ZERO_AMOUNT`, `MIXED_SIGNS`, `UNBALANCED`, `OVER_RELEASE`, `DUPLICATE_KEY`, `DIRECT_WRITE`,
`INCOMPATIBLE_CHANGE`, `BAD_CONFIG`, `UNKNOWN_TARGET`, `BAD_TAG`, `PERIOD_CLOSED`.
In SQL they are raised with SQLSTATE class `CL` and a JSON message `{code, index, detail}`.

## 8. Acceptance tests

Core (phase 1):
1. compi happy path: accrue → earn → to_payroll → clear; balances and trial balance correct.
2. Negative delta flows through the same types; accounts may end negative.
3. R5: every row of the R5 table behaves as stated; origin accounts are exempt.
4. Stripe: `charge` with three legs, `settle` across subjects (charge → payout), `pay_out`;
   balances per subject correct; the settle entry appears in both chains.
5. `UNKNOWN_TYPE`, `BAD_PARAMS`, `BAD_AMOUNTS`, `UNBALANCED`, `MIXED_SIGNS`, `ZERO_AMOUNT`.
6. Idempotency: replay returns the same receipt with `replayed = true`; changed content →
   `DUPLICATE_KEY`; duplicate key within a batch → `DUPLICATE_KEY`.
7. Atomicity: a batch with one bad entry writes nothing; heads unchanged.
8. `DIRECT_WRITE` on INSERT/UPDATE/DELETE/TRUNCATE outside chainledger's functions.
9. Concurrency on one chain: N writers → no gaps, valid chain, correct balances.
10. Parallelism: writers on disjoint chains (and different ledgers) don't wait for each other;
    batches touching overlapping chains in opposite orders don't deadlock.
11. One batch touching 50,000 distinct chains succeeds on default Postgres settings.
12. `balances` by prefix sums correctly at each level.
13. `declare` is idempotent; incompatible changes rejected; `BAD_CONFIG` for invalid types.
14. Digests and hashes computed by the Rust client match those stored by Postgres.
15. `effective_at` defaults to `created_at`, is stored, and is part of the digest.
15a. `validate` catches invalid config offline; `plan` reports adds and incompatible changes
     without writing; `apply` with a stale `plan_hash` refuses; `describe` round-trips the config.
15b. `post(dry_run = true)` returns the same receipts/rejections as a real post and writes nothing.

Integrity (phase 4):
16. Tamper, as superuser with triggers disabled:
    - change a leg amount after the last seal → incremental `verify` reports it;
    - change one side of a cross-subject entry → the other chain's posting check fails too;
    - change an entry before the last seal → `verify(prefix)` and `verify(full)` report it;
    - change a stored balance → balance mismatch reported.
17. Roll-up: node hashes at every level recompute from their children; prefix balances equal
    the sum of their leaves.
18. `seal()` with concurrent writers: writers aren't blocked; one consistent snapshot; the
    next incremental `verify` covers exactly what came after it.
19. `changes(a, b)` returns everything between seals a and b, none twice, none missing,
    with concurrent writers.
20. Incremental `verify` time grows with postings since the last seal, not with history.

Tags (phase 5):
21. New value appends and becomes current; history keeps both; same value again is a
    no-op; `ref:` tags apply to entries posted later with that ref.
22. Tag tamper → `verify` reports that `tag_seq`; `UNKNOWN_TARGET`, `BAD_TAG`.

Periods (phase 6):
23. After `close_period(until)`, entries with `effective_at < until` → `PERIOD_CLOSED`;
    later ones succeed; closes only move forward; `balance(as_of)` correct.

Archive (phase 7):
24. Archived Parquet recomputes to the seal's roll-up; a corrupted upload is detected and
    nothing is marked archived.
25. Uploading to an existing archive path fails; deleting in a retention-locked bucket fails.
26. `prune()` removes only archived seals older than `keep_days`; posting, R5, balances and
    incremental verify keep working; `verify(full)` spans archive + Postgres.
27. Storage: average bytes per leg (excluding metadata) within the §6 target.

## 9. Not in v1

Currency conversion/FX (currency is a label) · pending/held transfers (model them as
accounts/entry types if needed) · chart of accounts, account kinds, financial statements
(that's the accounting system's job) · per-identity permissions for apps and agents
(planned, next after v1) · MCP server (later) ·
summary journals for the GL, e.g. NetSuite (planned) · external anchoring of seal hashes
(planned) · Node bindings.

## 10. Build phases

Each phase is usable on its own.

1. **SQL core** — schema, `install`, `declare`, `post` (R1–R14, chains, postings, digests,
   `effective_at` stored, `dry_run`), reads (`entry`, `by_ref`, `balance`, `balances`,
   `history`, `chain`, `trial_balance`, `describe`), and `validate`/`plan`/`apply`.
   Tests from Python against a local Postgres (tests 1–15b).
2. **First consumer: compi** — replay a month of real compi history; compare balances with
   today's BigQuery totals. Validates the model before building further.
3. **Rust client + Python bindings** — once the SQL behaviour is settled (test 14).
4. **Integrity** — seals, roll-up, incremental/subtree/full verify, `changes()` (tests 16–20).
5. **Tags** (tests 21–22).
6. **Second consumer: Stripe payouts** — validates multi-leg and cross-subject entries in
   real use; add period close (R19, test 23).
7. **Archive** (optional) — R18 (tests 24–27).
