# chainledger — specification v1

A small double-entry engine for apps that need traceable, tamper-evident money movements
(first user: compi). Apps declare ledgers in config and post entries; Postgres enforces
every rule. chainledger is a library, not a service: it installs its schema into a
Postgres database you point it at (e.g. Cloud SQL) and talks to it over SQL.

Status: agreed design, not yet implemented.

---

## 1. Concepts

| Term | Meaning |
|---|---|
| **Ledger** | One independent book, e.g. `compi`: its own accounts, chains, seals and currency. |
| **Stage** | A step of the ledger's route, e.g. `source → eligible → earned → payroll → paid`. |
| **Subject** | Whose money it is: a fixed list of segments declared per ledger, e.g. `owner, deal, component`. A ledger may declare no segments. |
| **Account** | `{segment values}:{stage}`, e.g. `u123:d456:c789:eligible`. With no segments, the account is just `{stage}`. |
| **Entry** | Moves an amount from one account to the next stage of the same subject. Credits `from`, debits `to`. |
| **Chain** | The numbered, hash-linked sequence of entries of one full subject (the lowest level, e.g. owner + deal + component). Every entry belongs to exactly one chain (R2). A ledger with no segments has a single chain. Chains are written and verified independently. |
| **Roll-up** | Higher levels (owner + deal, owner, the whole ledger) are aggregations of the level below: balances by sum, integrity by hashing the children's hashes (a Merkle tree shaped like the subject hierarchy). |
| **Seal** | A checkpoint of the whole ledger: the roll-up's root hash, linked to the previous seal. |
| **Batch** | The entries of one `post()` call. All are written or none. |
| **Ref** | Optional app-defined string on an entry that groups related entries, e.g. compi's `compi_line_id`. Part of the entry and its hash. |
| **Tag** | A named text value attached to a ref or to one entry, e.g. `quota_period_name = "FY26 Q3"`. Append-only with its own small chain per target; the latest value wins. Doesn't touch balances or entry hashes. |

## 2. Configuration (TOML)

```toml
[ledgers.compi]
currency = "USD"
decimals = 2                                    # minor units: 1 USD = 100
subject  = ["owner", "deal", "component"]       # optional; [] or omitted = no segments
stages   = ["source", "eligible", "earned", "payroll", "paid"]

[ledgers.compi.archive]                         # optional, see R18
bucket    = "gs://chainledger-archive"          # bucket with a locked retention policy
keep_days = 90                                  # sealed + archived entries older than this may be pruned from Postgres
```

- Ledger ids, segment names and stage names match `[a-z][a-z0-9_]*`.
- Segment values match `[A-Za-z0-9_.@-]+` (no `:`). Every account has exactly the declared
  number of segments; apps use a placeholder (e.g. `_`) for a segment that doesn't apply.
- At least two stages.
- Granularity (which segments, how fine) is the app's choice. Rules apply per full account,
  so the finest declared level is where R5 holds.
- Chains always follow the finest level: one chain per full subject value. There is no
  separate chain setting.

## 3. Rules

**R1 Route.** An entry is valid only from stage *i* to stage *i+1* of its ledger's route.
No skipping, no going backwards. → `BAD_ROUTE`

**R2 Same subject.** `from` and `to` have identical segment values; only the stage differs.
→ `SUBJECT_MISMATCH`

**R3 Balanced by construction.** An entry credits `from` and debits `to` by the same amount.
The sum of all account balances of a ledger is always 0.

**R4 Signed integer amounts.** An amount is a non-zero integer in minor units (cents),
range of a signed 64-bit integer. Negative amounts are reductions flowing along the route.
No floats or decimals anywhere. → `ZERO_AMOUNT`, `BAD_AMOUNT`

**R5 No over-release.** For every account that sends a positive amount in a batch, its
balance after the batch must be ≥ 0. Accounts that only send negative amounts, or only
receive, may end negative. The first stage is exempt (it is always negative by design).
→ `OVER_RELEASE` (reports the account and the first entry index that sent from it)

Examples (eligible starts at 100):

| Batch | Eligible after | Result |
|---|---|---|
| `eligible→earned +100` | 0 | ok |
| `eligible→earned +200` | −100 | `OVER_RELEASE` |
| `source→eligible −30`, `eligible→earned +100` | −30 | `OVER_RELEASE` |
| `source→eligible −30`, `eligible→earned +70` | 0 | ok |
| `source→eligible −130` | −30 | ok (no positive amount sent; may end negative) |
| `source→eligible −130`, `eligible→earned −30` | 0 | ok (only negative amounts sent) |

**R6 Envelope.** Money enters a ledger only at the first stage. Follows from R1.

**R7 Append-only.** Entries are never updated or deleted. Corrections are new entries.
The only exception is `prune()` (R18): removing from Postgres entries that are sealed,
archived and verified in immutable storage.

**R8 Chains.** Every entry belongs to the chain of its subject (its full segment values;
the empty string for a ledger without segments). Each chain numbers its entries
`1, 2, 3, …` with no gaps. `hash = sha256(encode(entry) || prev_hash)` (see §5), where
`prev_hash` is the previous entry of the same chain; a chain's first `prev_hash` is 32 zero
bytes. An entry is identified by `(chain, seq)`.

**R9 Idempotency.** Each entry carries an app-supplied `key`, unique per ledger.
- Same key, same payload (`ref`, `from`, `to`, `amount`, `metadata`): the original receipt is
  returned with `replayed = true`; nothing is written.
- Same key, different payload → `DUPLICATE_KEY`.
- A batch may mix replayed and new entries; only the new ones are written.
- The same key twice within one batch → `DUPLICATE_KEY`.

**R10 Atomic batches.** On any rejection nothing from the batch is written. The error
carries the code and the index of the failing entry.

**R11 Parallel writers.** `post()` locks only the chains its batch touches, by row-locking
their `heads` rows (`SELECT … FOR UPDATE`, sorted by chain, after creating missing heads with
`INSERT … ON CONFLICT DO NOTHING`), so writers never deadlock. Row locks, not advisory
locks: advisory locks use Postgres's fixed-size shared lock table, which a batch touching
thousands of chains would exhaust; row locks have no such limit.
Writers wait for each other only when they touch the same chain (same full subject);
everything else runs in parallel. A batch spanning many chains is still atomic (one
transaction). No ledger-wide lock exists on the write path.

**R12 Postgres enforces everything.** All rules run inside Postgres (`post()` plus triggers).
Any INSERT, UPDATE, DELETE or TRUNCATE on chainledger tables outside `post()`/`declare()`
is rejected. → `DIRECT_WRITE`. Client libraries only make calling the SQL convenient.

**R13 Accounts from config.** An account is created on its first use if its name matches the
ledger's declared segments and stages. Any other name → `BAD_ACCOUNT`.
Unknown ledger → `UNKNOWN_LEDGER`.

**R14 Declare is idempotent.** `declare(config)` creates or updates ledgers.
- New ledger: created.
- Adding a stage at the end of the route: allowed.
- Changing currency, decimals, segments, or removing/renaming/reordering stages of a ledger
  that has entries → `INCOMPATIBLE_CHANGE`. Allowed while the ledger has no entries.

**R15 Verify.** Recomputes from entries alone: sequence (no gaps), hashes, account
balances, and roll-up hashes. Returns ok, or the first broken `(chain, seq)` / node and what
failed. Chains are independent, so every mode verifies them in parallel.
- `verify(ledger)` — **incremental, the routine check.** For each chain that changed since
  the last seal: walk from the sealed head `(seq, hash, balances)` to the current head.
  Cost is proportional to entries since the last seal, not to history.
- `verify(ledger, prefix)` — **one subtree**, e.g. one owner or one owner + deal: its chains
  from genesis, then its roll-up hash compared with the last seal.
- `verify(ledger, full = true)` — **everything** from genesis: all chains, every seal's
  roll-up and the seal chain. For scheduled deep checks (e.g. weekly).
- The trial balance (= 0) is checked in every mode.
Tag chains (R16) are verified the same way.

**R16 Tags.** Tags hold values that change over time without touching entries (e.g. a
line's quota period being renamed).
- A tag is `{target, name, value}`. `target` is either `ref:{ref}` (applies to every entry
  with that ref, past and future) or `entry:{key}` (one entry). `name` matches
  `[a-z][a-z0-9_]*`; `value` is text.
- Tags are append-only: setting a tag again appends a new row; the current value of
  `(target, name)` is the one with the highest `tag_seq`. History is kept.
- Appending a value equal to the current one is a no-op (returns the existing tag), so
  refreshing tags on every run is idempotent.
- Each target has its own tag chain (`tag_seq` 1, 2, 3, … and hashes, same construction as
  R8, §5), locked per target by row-locking its `tag_heads` row, like R11. Tags on different
  targets never wait for each other.
- A target `entry:{key}` must exist → `UNKNOWN_TARGET`. A `ref:` target need not have entries yet.
- Tags never affect balances, rules R1–R5, or entry hashes.

**R17 Roll-up and seals.**
- A **leaf** hash covers one chain: its head `(seq, hash)` and its accounts' balances.
- A **node** hash covers one subject prefix (e.g. owner + deal, then owner): the hashes of
  its children, sorted by key. The **root** covers the whole ledger. So integrity rolls up
  exactly like balances do: owner = f(deals), deal = f(components). Encoding in §5.
- Roll-up hashes are computed at seal time, not on every post, so no shared node is
  updated on the write path (keeps R11 parallel).
- `seal(ledger)` reads all chain heads at one consistent snapshot (it does not block
  writers), runs incremental verify (R15) up to that snapshot, then appends a seal
  `{seal_no, sealed_at, entries_root, tags_root, prev_seal_hash, hash}`. It stores the leaf
  and node hashes that changed since the previous seal. Seals are append-only and form
  their own hash chain; only one `seal()` per ledger runs at a time.
- Run `seal()` on a schedule, e.g. after each app run or daily. Seal hashes are what later
  gets anchored outside the database.

**R18 Archive (optional, per ledger).** Makes sealed history immutable even for database
admins, and keeps Postgres small. Enabled by `[ledgers.<id>.archive]`.
- `archive(ledger)` (Rust client) takes each sealed-but-unarchived seal, writes
  `changes(prev, seal)` as Parquet (§6a), and uploads to
  `{bucket}/{ledger}/seal={seal_no:010}/` with create-only preconditions
  (`ifGenerationMatch=0`), so an existing object is never overwritten.
- It then re-reads the uploaded files, recomputes their hashes against the seal's roll-up,
  and only then calls `mark_archived(ledger, seal_no, manifest_sha256)`. A failed check
  archives nothing.
- The bucket must have a **locked** retention policy (≥ the legal retention period), so
  objects can't be changed or deleted by anyone until it expires.
- `prune(ledger)` deletes from Postgres the entries and tags of seals that are archived and
  older than `keep_days`. Chain heads, account balances, seals and seal nodes are never
  pruned, so posting, R5 and incremental verify never need the archive.
- `verify(full)` reads archived seals from Parquet and the rest from Postgres. Archived
  seals were checked once at archive time and can't change afterwards, so a scheduled full
  verify is optional (an audit), not part of normal operation.
- Without an archive, R7 holds in full (nothing is ever deleted) and protection against
  admins is detection only (R15).

## 4. Operations

All are SQL functions in schema `chainledger`; the Rust client and bindings wrap them.

| Operation | Input | Returns |
|---|---|---|
| `install()` | — | creates/upgrades the schema (versioned migrations) |
| `declare(config)` | parsed TOML | created / changed / unchanged per ledger |
| `post(ledger, entries[])` | entries: `{key, ref?, from, to, amount, metadata}` | per entry: `{chain, seq, hash, replayed}` |
| `tag(ledger, tags[])` | tags: `{target, name, value}` | per tag: `{target, tag_seq, hash, unchanged}`; atomic like `post` |
| `tags(ledger, target)` | `ref:{ref}` or `entry:{key}` | current value per tag name |
| `tag_history(ledger, target, name)` | | every value in `tag_seq` order |
| `by_ref(ledger, ref)` | | entries with that ref, ordered by `created_at, chain, seq` |
| `balance(ledger, account)` | full account name | balance |
| `balances(ledger, subject_prefix)` | e.g. `["u123"]` or `["u123","d456"]` | balance per stage, summed over matching accounts |
| `history(ledger, account)` | full account name | that account's entries in `seq` order |
| `seal(ledger)` | | the new seal `{seal_no, sealed_at, entries_root, tags_root, hash}` |
| `changes(ledger, after_seal, upto_seal)` | seal numbers | every entry and tag appended between the two seals — exact, no gaps or duplicates (for export/BigQuery) |
| `trial_balance(ledger)` | | total per stage; sum is 0 |
| `verify(ledger, prefix?, full?)` | see R15 | ok / first broken `(chain, seq)` or node |
| `archive(ledger)` | (Rust client; R18) | seals archived |
| `mark_archived(ledger, seal_no, manifest_sha256)` | called by `archive()` after its check | — |
| `prune(ledger)` | (R18) | entries/tags removed from Postgres |

Wire formats inside SQL:
- `amount` travels as a JSON **string** of an integer (e.g. `"-3000"`) to avoid JSON number
  precision issues; stored as `bigint`.
- `metadata` travels as a JSON **string** containing canonical JSON (RFC 8785), produced by
  the client. Postgres stores those exact bytes (for hashing), compressed; no `jsonb` copy.
  Querying on metadata happens downstream (BigQuery/DuckDB), or with an explicit
  `metadata::jsonb` cast.
- `post()` processes the whole batch with set-based SQL (`unnest` over arrays), not a loop
  per entry. Large imports (e.g. migrating history) go through `post()` in large batches;
  there is no bulk path that skips the rules.

## 5. Hash encoding

`encode(entry)` is the concatenation of, in order:

| Field | Encoding |
|---|---|
| ledger | u32 big-endian byte length + UTF-8 |
| chain | u32 BE length + UTF-8 (segment values joined by `:`; empty for no segments) |
| seq | u64 big-endian |
| key | u32 BE length + UTF-8 |
| ref | u32 BE length + UTF-8 (length 0 when absent) |
| from | u32 BE length + UTF-8 (full account name) |
| to | u32 BE length + UTF-8 |
| amount | i64 big-endian (two's complement) |
| created_at | u32 BE length + UTF-8 of RFC 3339 UTC with microseconds, e.g. `2026-10-30T14:03:22.123456Z` |
| metadata | u32 BE length + the stored canonical JSON bytes |

`hash = sha256(encode(entry) || prev_hash)`, 32 bytes. `created_at` is the database clock at
posting, truncated to microseconds. Postgres computes the hash in `post()`; any client can
recompute it independently.

Tags use the same construction on their target's chain: `tag_hash = sha256(encode(tag) || prev_tag_hash)`,
first `prev_tag_hash` = 32 zero bytes, where `encode(tag)` is: ledger, target (each u32 BE
length + UTF-8), tag_seq (u64 BE), name, value (each u32 BE length + UTF-8), created_at (as above).

Roll-up (R17), with `str(x)` = u32 BE length + UTF-8:
- `leaf = sha256(0x00 || str(chain) || u64(seq) || head_hash || balances)`, where
  `balances` is, for each of the chain's accounts sorted by stage index,
  `str(stage) || i64(balance)`. An empty chain is not a leaf.
- `node = sha256(0x01 || str(prefix) || child_1 || … || child_n)`, children sorted by their
  key (bytewise UTF-8). Children of a prefix at depth *d* are the prefixes at depth *d+1*
  (nodes) or, at the deepest level, the chains (leaves). The root has prefix `""`.
- `tags_root` is built the same way over tag chains, with one level: `0x01 || str("") ||`
  sorted tag leaves, `tag_leaf = sha256(0x00 || str(target) || u64(tag_seq) || head_hash)`.
- `seal_hash = sha256(0x02 || str(ledger) || u64(seal_no) || str(sealed_at) || entries_root || tags_root || prev_seal_hash)`;
  the first `prev_seal_hash` is 32 zero bytes.

## 6. Storage (reference schema)

Implementers may adjust, as long as rules and operations hold. Storage goals: entries
refer to chains and accounts by integer id, hashes are `bytea` of 32 bytes, nothing
derivable is stored twice. Target: ~150–250 bytes per entry plus compressed metadata.

- `chainledger.chains` — `id bigint` PK, `ledger_id`, `key text`; unique `(ledger_id, key)`

- `chainledger.ledgers` — `id text` PK, `no smallint` unique (compact internal id), `currency`,
  `decimals`, `subject text[]`, `stages text[]`, `created_at`
- `chainledger.heads` — PK `chain_id`, `seq`, `hash` (one row per chain; updated by
  `post()`; its row lock is the chain lock, R11)
- `heads`, `tag_heads` and `accounts` are updated on every post: create them with
  `fillfactor = 80` so updates stay HOT (in-page) and vacuum stays cheap.
- `chainledger.tag_heads` — PK `(ledger_id, target)`, `tag_seq`, `hash`
- `chainledger.seals` — PK `(ledger_id, seal_no)`, `sealed_at`, `entries_root`, `tags_root`,
  `prev_hash`, `hash`
- `chainledger.seal_nodes` — PK `(ledger_id, seal_no, kind, key)`: leaves (`chain`, `seq`,
  head `hash`), nodes (`prefix`) and tag leaves that changed at that seal, with their hash.
  A node's value at seal N is its latest row at or before N.
- `chainledger.accounts` — `id bigint` PK, `chain_id`, `stage_index smallint`, `balance bigint`;
  unique `(chain_id, stage_index)`. The account name is `chains.key || ':' || stage`, not stored.
- `chainledger.tags` — PK `(ledger_id, target, tag_seq)`, `name`, `value`, `created_at`,
  `hash`; index `(ledger_id, target, name, tag_seq desc)`
- `chainledger.entries` — PK `(chain_id, seq)`, `ledger_no smallint`, `key text`, `ref text` (indexed),
  `from_stage smallint`, `amount bigint`, `metadata text` (compression `lz4`),
  `created_at timestamptz`, `hash bytea`; unique `(ledger_no, key)`.
  - `to` is always stage `from_stage + 1` of the same chain (R1, R2), so it isn't stored.
  - `prev_hash` is the previous row's `hash`, and the payload for R9 is recomputed from the
    row, so neither is stored.
- `chainledger.archives` — PK `(ledger_id, seal_no)`, `uri`, `manifest_sha256`, `archived_at` (R18)
- Guard triggers on all tables: reject writes unless `current_setting('chainledger.writing', true) = 'on'`,
  which only `post()`/`tag()`/`seal()`/`declare()` set (`set_config(..., true)`, transaction-local).
  `entries`, `tags`, `seals` and `seal_nodes` additionally reject every UPDATE, and every
  DELETE except from `prune()` on archived seals (R18).

Note: a database superuser can still disable triggers and rewrite rows. That is detectable
(R15 against the seals, and later external anchoring of seal hashes), not preventable.

## 6a. Archive format (R18)

Per seal, under `{bucket}/{ledger}/seal={seal_no:010}/`:

- `entries.parquet` — one row per entry, sorted by `(chain, seq)`, zstd-compressed:
  `ledger` string · `chain` string · `seq` int64 · `key` string · `ref` string (nullable) ·
  `from` string · `to` string · `amount` int64 · `created_at` timestamp(µs, UTC) ·
  `metadata` string (the canonical JSON bytes) · `hash` fixed_size_binary(32)
- `tags.parquet` — `ledger` · `target` · `tag_seq` int64 · `name` · `value` ·
  `created_at` timestamp(µs, UTC) · `hash` fixed_size_binary(32)
- `manifest.json` — the seal record (§5 fields), the sha256 of each Parquet file, row counts,
  and the `(seq, hash)` each touched chain starts from (so a segment verifies on its own).

Full names and stages are written out (not internal ids), so the archive can be read and
verified with DuckDB/BigQuery and no Postgres. Rows are self-contained: §5 hashes recompute
from Parquet columns alone.

## 7. Client libraries

- **Rust crate** `chainledger`: config parsing (TOML), RFC 8785 metadata canonicalisation,
  typed errors mapped from rejection codes, sync Postgres driver, and an independent
  `verify` that recomputes hashes client-side, spreading chains over threads/connections.
  `archive()` and `prune()` (R18) also live here: Parquet via the `parquet`/`arrow` crates,
  GCS via `object_store`.
- **Python package** (PyO3 + maturin), same API:

```python
from chainledger import Client, Entry, Tag

cl = Client.connect(DATABASE_URL)
cl.install()
cl.declare_file("chainledger.toml")
receipts = cl.post("compi", [
    Entry(key="line-1:eligible:1", ref="line-1", from_="u123:d456:c789:source",
          to="u123:d456:c789:eligible", amount=5_000_00,
          metadata={"comment": "..."}),
])
cl.tag("compi", [Tag(target="ref:line-1", name="quota_period_name", value="FY26 Q3")])
seal = cl.seal("compi")
assert cl.verify("compi").ok
```

- Node bindings: later, not v1.

Rejection codes: `BAD_ROUTE`, `SUBJECT_MISMATCH`, `ZERO_AMOUNT`, `BAD_AMOUNT`, `OVER_RELEASE`,
`UNKNOWN_LEDGER`, `BAD_ACCOUNT`, `DUPLICATE_KEY`, `DIRECT_WRITE`, `INCOMPATIBLE_CHANGE`,
`UNKNOWN_TARGET`, `BAD_TAG`.
In SQL they are raised with SQLSTATE class `CL` and a JSON message `{code, index, detail}`.

## 8. Acceptance tests

1. Happy path: source → eligible → earned → payroll → paid; balances and trial balance correct.
2. Negative delta propagates along the route; accounts may end negative.
3. R5: each row of the table in R5 behaves as stated.
4. `BAD_ROUTE` for a skipped stage and for a backwards move.
5. `SUBJECT_MISMATCH` for different owner/deal/component.
6. Idempotency: replay returns the same `chain`/`seq`/`hash` with `replayed = true`; changed payload → `DUPLICATE_KEY`.
7. Atomicity: a batch with one bad entry writes nothing; heads unchanged.
8. `DIRECT_WRITE` on INSERT/UPDATE/DELETE/TRUNCATE outside `post()`/`tag()`/`seal()`/`declare()`.
9. Tamper, as superuser with triggers disabled:
   - change an entry posted after the last seal → incremental `verify` reports its `(chain, seq)`;
   - change an entry from before the last seal → `verify(prefix)` for its owner and `verify(full)`
     report it; the subtree's roll-up hash no longer matches the seal;
   - change a stored balance → reported as a balance mismatch.
10. Concurrency on one chain: N writers on the same subject → no gaps, valid chain, correct balances.
11. Parallelism: writers on different subjects (and different ledgers) don't wait for each
    other; a batch spanning many subjects stays atomic; opposite-order multi-chain batches don't deadlock.
12. `balances` by prefix sums correctly at each segment level.
13. `declare` is idempotent; incompatible changes rejected once entries exist.
14. Hashes computed by the Rust client match those stored by Postgres.
15. Tags: setting a new value appends and becomes current; history keeps both; setting the
    current value again is a no-op; `ref:` tags apply to entries posted later with that ref.
16. Tag tamper: changing a tag value with triggers disabled → `verify` reports that `tag_seq`.
17. `UNKNOWN_TARGET` for an `entry:` tag on a missing key.
18. Roll-up: node hashes at every level recompute from their children; balances per prefix
    equal the sum of their leaves.
19. `seal()` while writers are posting: writers aren't blocked; the seal reflects one
    consistent snapshot; the next incremental `verify` covers exactly what came after it.
20. `changes(a, b)` returns every entry/tag appended between seals a and b, none twice,
    none missing, also with concurrent writers.
21. Incremental `verify` time grows with entries since the last seal, not with total history.
22. One batch touching 50,000 distinct chains succeeds on default Postgres settings (no lock-table exhaustion).
23. Archive round trip: archived Parquet recomputes to the seal's roll-up; a corrupted
    upload is detected and nothing is marked archived.
24. Uploading to an existing archive path fails (create-only); deleting an object in a
    retention-locked bucket fails.
25. `prune()` removes only archived seals older than `keep_days`; posting, R5, balances and
    incremental verify keep working afterwards; `verify(full)` spans archive + Postgres.
26. Storage: average bytes per entry (excluding metadata) within the §6 target.

## 9. Not in v1

Multiple currencies/FX · pending/held transfers · chart of accounts, account kinds, reports ·
entries with more than two legs · database roles/users · external anchoring of seal hashes
(planned) · Node bindings.
