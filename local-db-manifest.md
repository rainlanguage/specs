# Local DB Manifest

Specification of the YAML manifest that coordinates synchronisation between Rain
Local DB instances in the browser and periodically published SQLite dumps.

## Design goals

- Allow clients to reset and reseed the entire local database when breaking
  changes such as schema migrations occur.
- Allow clients to reset an individual network when upstream data is rebuilt,
  e.g. after a reorg.
- Provide per-network metadata for deciding between continuing incremental
  indexing or replacing local data with the latest dump.
- Remain simple to host and distribute (plain YAML served over HTTPS).
- Keep the manifest append-only in intent: values only ever increase/advance,
  avoiding ambiguity about whether data should be rolled back.

## Format overview

- A single manifest covers every network that has a published dump.
- Clients SHOULD treat unknown fields as opaque metadata for forward
  compatibility.
- The document MUST contain the fields described in this specification. Any
  additional fields remain implementation defined.

### Example

```yaml
schema_version: 1
networks:
  42161:
    dump_url: https://example.invalid/releases/download/sync-18345928447/42161.sql.gz
    dump_timestamp: 2025-10-08T13:19:15.577198557+00:00
    seed_generation: 1
```

## Top-level fields

### `schema_version`

- Monotonic: every published manifest MUST set a value greater than or equal to
  the previous manifest.
- Purpose: signals breaking changes to the local database schema or other
  conditions that require clients to discard all existing network data.
- Client behaviour:
  - Clients MUST persist the last applied `schema_version`.
  - If the manifest reports a higher `schema_version` than stored locally, the
    client MUST drop every local network database, download the latest dump for
    each network, and seed from scratch.
  - After the global drop, clients only reseed networks present in
    `networks`; locally configured networks that are absent restart their
    incremental indexing flows without first applying a dump.
  - Decreasing `schema_version` values MUST be ignored; clients retain the
    highest value they have already applied.

### `networks`

- Clients MUST ignore networks whose chain ID they do not track locally.
- Clients MUST continue indexing any locally configured network that is absent
  from the manifest, using incremental indexing only. This allows user- or
  community-defined networks that lack published dumps to keep functioning,
  albeit without the performance benefits of reseeding.
- Each network entry MUST contain the fields described below.

## Network entries

Each value under `networks` describes a single chain.

### `dump_url`

- MUST reference an HTTPS endpoint serving a gzip-compressed SQLite dump.
- The SQLite schema MUST match the schema implied by the current
  `schema_version`.
- Clients SHOULD treat the URL as opaque; hosting location is unconstrained
  (GitHub Releases, CDN, etc.).
- A republished dump MUST advance the accompanying `dump_timestamp`. The
  `dump_url` MAY remain stable or change (e.g. when using immutable GitHub
  Release assets). Clients treat the combination of `dump_url` and
  `dump_timestamp` as the identity of the artifact when deciding whether to
  redownload.

### `dump_timestamp`

- Monotonic per network: newer manifests MUST publish timestamps that strictly
  increase for a given chain ID.
- Purpose: allows clients to decide whether their local incremental index is
  fresh enough to continue or whether they should reload the dump.
- Client behaviour:
  - Clients MUST persist the last applied `dump_timestamp` for each network.
  - Clients derive the "last fully indexed time" from their own persisted sync
    metadata, typically recording a wall-clock timestamp at the end of each
    incremental index cycle.
  - Clients compare the manifest timestamp with the local notion of "last fully
    indexed time". If the elapsed time exceeds an implementation-defined
    freshness threshold (intended to be on the order of minutes), the client
    MUST drop the local database for that network and reseed from the dump.
  - If the manifest timestamp is earlier than the client's last fully indexed
    time, treat the elapsed time as zero and continue without resetting.
  - Manifest updates alone do not trigger resets: when the difference between
    the manifest timestamp and the client's last fully indexed time is less than
    or equal to the freshness threshold, clients continue incremental indexing
    without applying the new dump.

### `seed_generation`

- Monotonic per network: manifests MUST never decrease this value.
- Purpose: network-scoped reset signal, e.g. when upstream data is rebuilt after
  a reorg or other corrective action.
- Client behaviour:
  - Clients MUST persist the last applied `seed_generation` per network.
  - When the manifest reports a higher `seed_generation`, the client MUST drop
    the local database for that chain, redownload the dump, and resume indexing.
    This reset bypasses freshness heuristics; the dump is reapplied even if the
    published artifacts match prior values.
  - Decreasing values MUST be ignored; clients retain the highest value they
    have already applied.

## Client state persistence

Clients persist manifest data in their local SQLite database. A canonical table
definition used by local db implementation in orderbook repository is:

```sql
CREATE TABLE IF NOT EXISTS remote_seeds (
    chain_id INTEGER PRIMARY KEY,
    schema_version INTEGER NOT NULL,
    seed_generation INTEGER NOT NULL,
    seed_timestamp TEXT NOT NULL,
    seed_url TEXT NOT NULL,
    seed_applied_at TIMESTAMP
);
```

- `seed_timestamp` stores the applied `dump_timestamp`.
- `seed_url` stores the applied `dump_url`.
- `seed_applied_at` SHOULD be set to the local timestamp whenever a dump is
  successfully applied, even if the contents are identical to the previous dump.

Clients MAY record additional bookkeeping (e.g. last indexed block) alongside
this table to implement their freshness heuristics.

## Fetching and refresh policy

- Manifests are regenerated approximately every 10–15 minutes alongside new
  dumps. Clients SHOULD refresh on a schedule aligned with dump publication and
  MUST retry failed fetches until successful.
- A failed fetch MUST NOT clear local state. Clients continue operating with
  the last successfully applied manifest until a new one is fetched.
- Implementations SHOULD use exponential backoff or bounded retry intervals to
  avoid tight retry loops during outages.

## Applying dumps

Clients MUST follow this order of operations when processing a manifest:

1. Fetch and parse the manifest.
2. Compare the top-level `schema_version` to the stored value.
  - If it increases, perform a global reset before evaluating per-network
    fields.
    - The global reset consists of dropping every local network database and
      redownloading dumps for all networks present in the manifest.
3. For each locally tracked network:
   - If absent from `networks`, continue incremental indexing using existing
     local configuration once the database has been reset; no dump is applied
     in this case. Clients rely on the orderbook configuration's
     `deployment-block` (see `ob-yaml.md`) to determine the block height to
     resume from. If a global reset removed local state, the client rebuilds
     from that deployment block using incremental indexing alone.
   - Otherwise, compare `seed_generation`.
     - If it increases, drop the local database for that chain and reseed from
       the dump.
   - Compare `dump_url` and `dump_timestamp`.
     - If `dump_timestamp` differs from the stored value, evaluate the freshness
       threshold using the client's last fully indexed time. When the threshold
       or other policy demands reseeding, redownload and apply the dump even if
       `dump_url` is unchanged.
     - If the manifest also publishes a different `dump_url`, redownload and
       apply the dump to pick up the new location before resuming incremental
       indexing.
     - After applying a dump, update `remote_seeds` with the manifest values and
       the current local timestamp in `seed_applied_at`.
4. Resume incremental indexing from the last known block subsequent to the dump.

## Non-goals

- The manifest does not describe incremental updates or log ranges. Each entry
  only references full dumps.
- The manifest does not carry cryptographic integrity data; trust in the dump is
  derived from the publication process and the monotonic versioning fields.
- Hosting details (domains, authentication) are intentionally unspecified.
