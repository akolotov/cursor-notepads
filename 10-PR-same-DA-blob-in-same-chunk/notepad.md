When indexing an Orbit chain that uses AnyTrust as a Data Availability (DA) solution, an issue has been identified where multiple batches within the same processing chunk can reference the same data blob, causing database import failures.

This issue prevents proper indexing of Arbitrum Orbit chains where the AnyTrust committee legitimately uses the same data blob for multiple batches. The indexing process errors out with uniqueness constraint violations:

```
GenServer Indexer.Fetcher.Arbitrum.TrackingBatchesStatuses terminating
** (Postgrex.Error) ERROR 21000 (cardinality_violation) ON CONFLICT DO UPDATE command cannot affect row a second time

    hint: Ensure that no rows proposed for insertion within the same command have duplicate constrained values.
    (ecto_sql 3.12.1) lib/ecto/adapters/sql.ex:1096: Ecto.Adapters.SQL.raise_sql_call_error/1
    (ecto_sql 3.12.1) lib/ecto/adapters/sql.ex:967: Ecto.Adapters.SQL.insert_all/9
    (ecto 3.12.5) lib/ecto/repo/schema.ex:59: Ecto.Repo.Schema.do_insert_all/7
    (explorer 7.0.2) lib/explorer/repo.ex:47: anonymous fn/5 in Explorer.Repo.safe_insert_all/3
    (elixir 1.17.3) lib/enum.ex:2531: Enum.\"-reduce/3-lists^foldl/2-0-\"/3
    (explorer 7.0.2) lib/explorer/chain/import.ex:320: Explorer.Chain.Import.insert_changes_list/3
    (explorer 7.0.2) lib/explorer/chain/import/runner/arbitrum/da_multi_purpose_records.ex:70: Explorer.Chain.Import.Runner.Arbitrum.DaMultiPurposeRecords.insert/3
    (stdlib 6.1) timer.erl:590: :timer.tc/2
Last message: :check_historical_batches
State: %{. . .}
```

## Analysis

The error `ERROR 21000 (cardinality_violation) ON CONFLICT DO UPDATE` occurs in PostgreSQL when an `INSERT ... ON CONFLICT DO UPDATE` statement attempts to modify the same row multiple times within a single command. PostgreSQL enforces that each row should be affected only once per statement, and when multiple conflicting rows exist in the input data, it leads to a cardinality violation.

In this case, the issue arises because multiple batches within the same processing chunk reference the same AnyTrust data blob.

```
Workers.Batches.Discovery.perform
..Workers.Batches.Discovery.handle_batches_from_logs <-- called for every chunk
....Workers.Batches.Discovery.execute_transaction_requests_parse_transactions_calldata <-- DA info for all batches in the chank extracted from calldata
....DA.Common.prepare_for_import
......DA.Anytrust.prepare_for_import <-- DA records are composed from DA info for entire chunk
..Explorer.Chain.import <-- called for the same chunk
....Explorer.Chain.Import.Runner.Arbitrum.DaMultiPurposeRecords.insert
```
** *Modules within `Workers` and `DA` are, in fact, under the `Indexer.Fetcher.Arbitrum` umbrella. This prefix was omitted in the code flow scheme above for readability.*

The current implementation in `handle_batches_from_logs` of `Indexer.Fetcher.Arbitrum.Workers.Batches.Discovery` accumulates all DA records for the chunk without deduplication. This means:

1. Each batch creates a separate DA record with the same `data_key` (the `data_hash`).
2. All DA records are passed to `Explorer.Chain.import` for database insertion.
3. During insertion, multiple records with the same `data_key` trigger the `ON CONFLICT DO UPDATE` clause, attempting to update the same row multiple times in a single transaction.

Since PostgreSQL cannot determine which of the conflicting rows should be applied, it raises the cardinality violation error, preventing successful indexing.

This differs from the previously fixed scenario in PR [#11485](https://github.com/blockscout/blockscout/pull/11485), which addressed the same issue for batches discovered in different chunks or indexing iterations.

## Proposed Solution

To fix the issue of duplicate DA records (i.e. multiple batches referencing the same data blob) within a single processing chunk, the solution introduces two deduplication steps:

1. **Module-Level Deduplication:**
    In both the `Indexer.Fetcher.Arbitrum.DA.Anytrust` and `Indexer.Fetcher.Arbitrum.DA.Celestia` modules, update the `prepare_for_import` function to deduplicate DA records during transformation.
    
    - **AnyTrust Records:** Among records with the same `data_key`, retain the one with the highest `timeout` value (embedded in the opaque `data` JSON) while still preserving all batch-to-blob associations.
    - **Celestia Records:** For duplicates based on `data_key`, simply preserve the first occurrence since Celestia records are not time-sensitive.

2. **Final Pre-Insertion Check:**
    After obtaining the deduplicated list from the DA modules, perform an additional application-layer filtering step before bulk insertion. For each record in the final deduplicated list:
    
    - **Existence Check:** Query the database to determine whether a record with the same `data_key` already exists.
    - **Conditional Inclusion:**
        - For **Celestia Records:** If a corresponding record exists in the database, remove the incoming record from the import list.
        - For **AnyTrust Records:** In addition to checking `data_key`, compare the `timeout` value extracted from the `data` field. Include the incoming record only if its timeout is higher than the existing record’s; otherwise, remove it from the import list.

Together, these steps ensure that only the most valid and up-to-date DA records reach the bulk insert stage, preventing the `ON CONFLICT DO UPDATE` clause from affecting the same row multiple times and thereby avoiding cardinality violations.

## High Level Scope of Changes

### New DB query

A new `da_records_by_keys` ****function to retrieve all `Explorer.Chain.Arbitrum.DaMultiPurposeRecord` entries for the given list of data keys is added to `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement`. It returns `nil` if no records are found in the database.

The corresponding wrapper function `da_records_by_keys` is added to `Indexer.Fetcher.Arbitrum.Utils.Db.Settlement`.

### `Indexer.Fetcher.Arbitrum.DA.Anytrust`

**- `prepare_for_import` modified:**

1. Instead of accepting a list of `Arbitrum.DaMultiPurposeRecord.to_import()`, it now accepts a map where `data_key` maps to `Arbitrum.DaMultiPurposeRecord.to_import()`.
2. If the provided `da_info` produces a DA record that duplicates an existing record in the map, the record with the highest `timeout` value is retained, ensuring longer data availability.

**- New function `resolve_conflict`:**

1. Accepts a list of `Arbitrum.DaMultiPurposeRecord` retrieved from the database and a map where `data_key` maps to `Arbitrum.DaMultiPurposeRecord.to_import()` (candidates for database import).
2. For each database record, compares the `timeout` value (extracted from the `data` field) with the `timeout` value of the corresponding element in the map. The candidate record is retained only if its `timeout` is higher; otherwise, it is removed.
3. Returns a flattened list of `Arbitrum.DaMultiPurposeRecord.to_import()`.

### `Indexer.Fetcher.Arbitrum.DA.Celestia`

- **`prepare_for_import` modified:**
1. Instead of accepting a list of `Arbitrum.DaMultiPurposeRecord.to_import()`, it now accepts a map where each `data_key` maps to `Arbitrum.DaMultiPurposeRecord.to_import()`.
2. If the provided `da_info` produces a DA record that duplicates an existing record in the map, the existing record remains unchanged.

**- New function `resolve_conflict`:**

1. Accepts a list of `Arbitrum.DaMultiPurposeRecord` retrieved from the database and a map where each `data_key` maps to `Arbitrum.DaMultiPurposeRecord.to_import()` (candidates for database import).
2. For each database record, checks if a corresponding entry exists in the map. Only records present in the map but absent in the database are retained.
3. Returns a flattened list of `Arbitrum.DaMultiPurposeRecord.to_import()`.

### `Indexer.Fetcher.Arbitrum.DA.Common`

**- `prepare_for_import`**

1. Instead of a flat list, `da_records` will now be organized as a map keyed by DA type (`:celestia` and `:anytrust`), and for each type further keyed by `data_key` to the corresponding `Arbitrum.DaMultiPurposeRecord.to_import()`.
2. Before calling `prepare_for_import` of `DA.Celestia` or `DA.AnyTrust`, the map of the corresponding type is taken from `da_records`. The map returned by `prepare_for_import` of `DA.Celestia` or `DA.AnyTrust` updates the map in `da_records`.
3. `da_records` is not returned from the function as is; instead, it is directed to `eliminate_conflicts`, and its result is returned from the function together with `batch_to_blobs`.

**- New function `eliminate_conflicts`**

For each DA type:

1. Collect all `data_key` keys from the candidates.
2. Query the database with `Db.Settlement.da_records_by_keys` to retrieve existing `DaMultiPurposeRecord` records corresponding to these keys.
3. Pass the candidate records and the found DB record to a `resolve_conflict` function defined in either the `DA.Celestia` or `DA.Anytrust` module. The expected result of the call is a flattened list of DA records to import, which includes:
    - New records (where the `data_key` wasn’t present in the DB).
    - Records that must update the DB.

It returns the list of `Arbitrum.DaMultiPurposeRecord.to_import()`.
