## Problem

During batch fetcher operations on the Blockscout instance indexing Arbitrum One chain, the following timeout occurs:

```
GenServer Indexer.Fetcher.Arbitrum.TrackingBatchesStatuses terminating
** (DBConnection.ConnectionError) ssl recv: closed (the connection was closed by the pool, possibly due to a timeout or because the pool has been terminated)
    (ecto_sql 3.12.1) lib/ecto/adapters/sql.ex:1096: Ecto.Adapters.SQL.raise_sql_call_error/1
    (ecto_sql 3.12.1) lib/ecto/adapters/sql.ex:994: Ecto.Adapters.SQL.execute/6
    (ecto 3.12.5) lib/ecto/repo/queryable.ex:232: Ecto.Repo.Queryable.execute/4
    (ecto 3.12.5) lib/ecto/repo/queryable.ex:19: Ecto.Repo.Queryable.all/3
    (ecto 3.12.5) lib/ecto/repo/queryable.ex:154: Ecto.Repo.Queryable.one/3
    (indexer 6.10.1) lib/indexer/fetcher/arbitrum/utils/db/settlement.ex:286: Indexer.Fetcher.Arbitrum.Utils.Db.Settlement.l1_blocks_to_expect_rollup_blocks_confirmation/1
    (indexer 6.10.1) lib/indexer/fetcher/arbitrum/workers/new_confirmations.ex:358: Indexer.Fetcher.Arbitrum.Workers.NewConfirmations.discover_historical_rollup_confirmation/1
    (stdlib 6.1) timer.erl:624: :timer.tc/3
```

The timeout occurs when the fetcher attempts to identify the parent chain's block range where missing rollup state confirmations should be found.

## Analysis

The Blockscout database has a table called `arbitrum_batch_l2_blocks` that stores settlement information for each rollup block. Each record links a rollup block to its batch and the parent chain transaction that confirmed the block's rollup state. The table includes three fields: `batch_number`, `confirmation_id`, and `block_number`. The `confirmation_id` references a record in the `arbitrum_lifecycle_l1_transactions` table, which stores settlement transaction details (transaction hash, block number, and timestamp). Due to Arbitrum's rollup architecture, a transaction confirms the state of all blocks up to the specified block. The fetcher reconstructs this confirmation history to track when each rollup block was first confirmed. Since one confirmation transaction covers multiple blocks, the Arbitrum One chain contains more than 300M of these records:

```
block_number: 307474716, confirmation_id: null
...
block_number: 307475194, confirmation_id: null
block_number: 307475193, confirmation_id: 30747
...
block_number: 307466335, confirmation_id: 30747
block_number: 307466334, confirmation_id: 30746
...
block_number: 307454351, confirmation_id: 30746
block_number: 307454350, confirmation_id: 30745
...
block_number: 307444475, confirmation_id: 30745
block_number: 307444474, confirmation_id: 30744
...
```

The rollup blocks `307_474_716..307_475_194` have not yet received a confirmation transaction in the parent chain, so their `confirmation_id` remains unset.

There are several reasons why blocks throughout the chain, not just at the top, lack associated confirmation transactions in the database. These gaps relate to how block confirmations are restored - not because the parent chain is missing confirmations for these blocks.

Reasons for Missing Confirmations:

- The Blockscout block fetcher has not indexed some rollup blocks due to service interruption (when either Blockscout or the RPC node used for indexing was temporarily offline). In such cases, the table `arbitrum_batch_l2_blocks` might look like this:

```
block_number: 5000, confirmation_id: null
...
block_number: 4957, confirmation_id: null
block_number: 4956, confirmation_id: 15
...
block_number: 4619, confirmation_id: 15
block_number: 4618, confirmation_id: null
...
block_number: 3991, confirmation_id: null
block_number: 3990, confirmation_id: 12
...
block_number: 3706, confirmation_id: 12
block_number: 3705, confirmation_id: 11
...
```

- The Arbitrum-specific entity indexer was started on a database that already contained indexed blocks, triggering the historical confirmation discovery process. The table `arbitrum_batch_l2_blocks` could look like this:

```
block_number: 5000, confirmation_id: null
...
block_number: 4957, confirmation_id: null
block_number: 4956, confirmation_id: 15
...
block_number: 877, confirmation_id: 3
block_number: 876, confirmation_id: null
...
block_number: 1, confirmation_id: null
```

These two scenarios can also occur in combination.

When these situations occur, the batch fetcher (more accurately called the settlement information fetcher) identifies which parent blocks need inspection to discover confirmations for the corresponding rollup blocks. For the examples above: 

- the confirmations for blocks `3991..4618` must be looked for between the parent chain blocks where the confirmations with ids `12` and `15` appear.
- the confirmations for blocks `1..876` must be looked for in the blocks before the parent chain block where the confirmation with id `3` appears.

The function `l1_blocks_to_expect_rollup_blocks_confirmation` in the `Indexer.Fetcher.Arbitrum.Utils.Db.Settlement` module identifies the most recent missing confirmations and determines which parent chain blocks to search. This is accomplished through a pipeline of queries in the function `l1_blocks_of_confirmations_bounding_first_unconfirmed_rollup_blocks_gap` of the `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement` module, executed within a single database transaction.

Here is a breakdown of the pipeline:

1. First query (`rollup_blocks_query`): Gets all confirmed rollup blocks with their confirmation IDs.
    
    Example data after this query:

    ```
    block_number: 1, confirmation_id: 100
    block_number: 2, confirmation_id: 100
    block_number: 3, confirmation_id: 100
    block_number: 7, confirmation_id: 102
    block_number: 8, confirmation_id: 102
    block_number: 9, confirmation_id: 102
    block_number: 12, confirmation_id: 104
    block_number: 13, confirmation_id: 104
    block_number: 12, confirmation_id: 105
    block_number: 15, confirmation_id: 105
    ```

2. Second query (`confirmed_ranges_query`): Groups the blocks by their confirmation transactions, showing the start and end block numbers for each confirmation range.
    
    Example result:

    ```
    confirmation_id: 100, min_block_num: 1, max_block_num: 3
    confirmation_id: 102, min_block_num: 7, max_block_num: 9
    confirmation_id: 104, min_block_num: 12, max_block_num: 13
    confirmation_id: 105, min_block_num: 14, max_block_num: 15
    ```

3. Third query (`confirmed_combined_ranges_query`): Employs a window function (`LAG`) to compare each confirmation with its previous record, helping identify gaps between confirmed blocks.
    
    Example result:

    ```
    confirmation_id: 100, min_block_num: 1, max_block_num: 3, prev_max_number: nil, prev_confirmation_id: nil
    confirmation_id: 102, min_block_num: 7, max_block_num: 9, prev_max_number: 3, prev_confirmation_id: 100
    confirmation_id: 104, min_block_num: 12, max_block_num: 13, prev_max_number: 9, prev_confirmation_id: 102
    confirmation_id: 105, min_block_num: 14, max_block_num: 15, prev_max_number: 13, prev_confirmation_id: 104
    ```

4. Final query (`main_query`): Identifies the most recent gaps where block numbers are not consecutive, focusing on higher rollup block numbers. In this example, there are two gaps:
    - Between blocks 3 and 7 (after confirmation 100)
    - Between blocks 9 and 12 (after confirmation 102)
    
    The query returns the parent chain block numbers where the gap-creating transactions occurred - specifically, the blocks containing confirmations 102 and 104.
    

In Arbitrum One, despite having only a few intervals with missing confirmations (such as the unconfirmed top blocks and the range `22_207_818..22_300_000`), the query must scan through hundreds of millions of confirmed rows to perform the grouping operation. This process is computationally intensive.

The database has an index on `confirmation_id`, but since the query filters on `NOT NULL`, groups by `confirmation_id`, and requires the `block_number` field - without having a composite index on `(confirmation_id, block_number)` - it must perform a full table scan.

The Arbitrum One table contains blocks from `22'207'818` to `>307'000'000`, with gaps near the beginning and end. Although gaps are infrequent (occurring only once per ~13,000 blocks), the query must examine all confirmed rows to identify the first gap. This means that even when searching for a small result set (a single tuple), the query still requires substantial computational resources.

## Proposed Solution

### DB Query Optimization

Based on two key observations:

1. The function `l1_blocks_to_expect_rollup_blocks_confirmation` only needs the boundaries of the most recent confirmation gap. Therefore, the extensive gap identification work performed by `l1_blocks_of_confirmations_bounding_first_unconfirmed_rollup_blocks_gap` is unnecessary—it doesn't need to find all historical gaps.
2. When calling `l1_blocks_to_expect_rollup_blocks_confirmation`, query execution speed is less critical than process stability. The primary concern is preventing query timeouts rather than optimizing performance.

Therefore, the approach for identifying the most recent confirmation gap boundaries can be redesigned.

The new code flow follows these steps:

1. Find the most recent confirmed block `X`. If no such block exists, skip all steps and return `nil`.
2. Find the most recent unconfirmed block `Y` below `X`. If no such block exists, all blocks in the table are confirmed. In this case, skip steps 3 and 4, use the lowest confirmed block as `A`, and leave `B` unset.
3. Find the confirmed block `A` that is closest to and above `Y` (ideally `Y + 1`)
4. Find the most recent confirmed block `B` below `Y`
5. Get confirmation transaction IDs `I` and `J` for blocks `A` and `B` respectively. If `B` is not found, `J` remains unset.
6. Get the parent chain blocks `L` and `M` for transactions `I` and `J` respectively. If `J` is unset, `M` remains unset (`nil`).
7. Use `L` as the highest parent chain block and `M` as the lowest parent chain block

Let's examine how this new approach aligns with the current behavior of `l1_blocks_of_confirmations_bounding_first_unconfirmed_rollup_blocks_gap`:

**Case 1: No Confirmed Blocks or Empty Table**

Since step 1 finds no confirmed block, it returns `nil`. This matches the current behavior.

**Case 2: All Blocks Are Confirmed**

Step 1 finds a block, while step 2 finds no unconfirmed blocks. Block `A` is set to the lowest confirmed block, and `B` remains unset. Step 7 returns a block for `L` and `nil` for `M` - matching the current behavior.

**Case 3 - Unconfirmed Blocks Near the Top**

```
block_number: 2065, confirmation_id: null
. . .
block_number: 2027, confirmation_id: null
block_number: 2026, confirmation_id: 6
. . .
block_number: 1757, confirmation_id: 6
block_number: 1756, confirmation_id: null
. . .
block_number: 1448, confirmation_id: null
block_number: 1447, confirmation_id: 4
. . .
block_number: 1143, confirmation_id: 4
block_number: 1142, confirmation_id: 3
. . .
block_number: 877, confirmation_id: 3
block_number: 876, confirmation_id: null
. . .
block_number: 1, confirmation_id: null
```

Step 1 returns `2026`, Step 2 returns `1756`, Step 3 returns `1757`, Step 4 returns `1447`, and Step 7 returns the parent chain blocks where transactions with IDs 4 and 6 were included. This behavior aligns with the current implementation.

**Case 3 - Unconfirmed Blocks Near the Bottom**

```
block_number: 2065, confirmation_id: null
. . .
block_number: 2027, confirmation_id: null
block_number: 2026, confirmation_id: 6
. . .
block_number: 1757, confirmation_id: 6
block_number: 1756, confirmation_id: 5
. . .
block_number: 1448, confirmation_id: 5
block_number: 1447, confirmation_id: 4
. . .
block_number: 1143, confirmation_id: 4
block_number: 1142, confirmation_id: 3
. . .
block_number: 877, confirmation_id: 3
block_number: 876, confirmation_id: null
. . .
block_number: 1, confirmation_id: null
```

Step 1 returns `2026`, Step 2 returns `876`, Step 3 returns `877`, Step 4 returns `nil`, and Step 7 returns the highest parent chain block where transactions with ID 3 were included, with `nil` for the lowest parent chain block. This behavior matches the current implementation.

### New Index

Although an index for `confirmation_id` exists for the `arbitrum_batch_l2_blocks` table:

```elixir
create(index(:arbitrum_batch_l2_blocks, :confirmation_id))
```

it is not optimal because all the queries currently operating with this field search for either confirmed or unconfirmed blocks. They don't use the exact value of `confirmation_id` for ordering or filtering. At the same time, these same queries use `block_number` for either ordering or filtering. This is also applicable to the new functionality described above.

Instead, a new composite index is suggested: `(confirmation_id, block_number DESC)` for all records where `confirmation_id` is `null`. The current index can be dropped later.

The functions `highest_confirmed_block` (of `Explorer.Chain.Arbitrum.Reader.Common`) and `l1_block_of_latest_confirmed_block` (of `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement`) will benefit from the introduction of the new index. Although the function `count_confirmed_rollup_blocks_in_batch` (of `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement`) will not benefit directly from the new index, it will not be affected by the drop of the index that contains only `confirmation_id`. The function `l1_block_of_confirmation_for_rollup_block` (of `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement`) relies on the index of `block_number`, so it will not be impacted as well. The functionality calling `unconfirmed_rollup_blocks` (of `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement`) could be reworked so that the introduction of the new index will speed up the corresponding DB query as well.

## High Level Scope of Changes

### Extending Reader Functionality

1. Similar to the function `highest_confirmed_block` in `Explorer.Chain.Arbitrum.Reader.Common`, the function to retrieve the most recent unconfirmed block below a given one is implemented in the module `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement`.

2. The function to implement step 3 of the code flow described above is designed as follows:

  - It calls `l1_block_of_confirmation_for_rollup_block` of `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement` to check if the next block immediately after the given one exists. It is assumed that this code branch will be used in most cases, and a simple increment of the block number allows us to avoid an unnecessary query to the DB in the case of consecutive blocks in the table.
  - If it does not, it executes a query to find the L1 block of the confirmed rollup that is closest to and above the given block.

3. Similar to the function `highest_confirmed_block`, the function to retrieve the most recent confirmed block before a given one is implemented. This function also includes functionality similar to `l1_block_of_confirmation_for_rollup_block` to retrieve the L1 block of the found rollup block in the same query.

4. The function `l1_blocks_of_confirmations_bounding_first_unconfirmed_rollup_blocks_gap` is removed from the module `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement`.

### Altering Wrapper Function

The function `l1_blocks_to_expect_rollup_blocks_confirmation` of `Indexer.Fetcher.Arbitrum.Utils.Db.Settlement` is modified to use the function `highest_confirmed_block` and new functions from the `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement` module to implement the code flow described above. The function will receive the result of the call flow - `nil` or the tuple of L1 blocks (`{lower_block, higher_block}` or `{nil, higher_block}`) and process it in the same manner as it is processing now the results of `l1_blocks_of_confirmations_bounding_first_unconfirmed_rollup_blocks_gap`.

### New Index Introduction

A new index `(confirmation_id, block_number DESC)` where `confirmation_id` is `null` for the `arbitrum_batch_l2_blocks` table is created as a Heavy DB Index Operation. This approach is chosen over a standard Ecto.Migration because:

1. The table contains hundreds of millions of records in production environments like Arbitrum One
2. The index creation process could take hours and potentially cause database timeouts
3. The application needs to remain operational during index creation

<mark>The index is implemented through the `Explorer.Migrator.HeavyDbIndexOperation.CreateArbitrumBatchL2BlocksUnconfirmedBlocksIndex` module, which:</mark>
- <mark>Creates the index with the `concurrently` option to minimize downtime</mark>
- <mark>Provides status tracking of the index creation process</mark>
- <mark>Updates a cache value in `Explorer.Chain.Cache.BackgroundMigrations` when complete</mark>

_The hightlighted part is added to the notepad after the main code changes have been made with the Agent._

### New Index Usage

The function `unconfirmed_rollup_blocks` in `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement` is rewritten to utilize the new index; the order of results is changed to descending.

The function `unconfirmed_rollup_blocks` in `Indexer.Fetcher.Arbitrum.Utils.Db.Settlement` is adapted to reverse the order of values returned by `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement.unconfirmed_rollup_blocks`. This ensures that the logic of calling functions which relies on the previous functionality of the function is not impacted.

<mark>To prevent database timeouts during index creation, the confirmation discovery process in `Indexer.Fetcher.Arbitrum.Workers.Confirmations.Tasks` is modified to:</mark>
1. <mark>Check the index readiness in a new `plan` function by requesting the cached status of the heavy index operation</mark>
2. <mark>Skip confirmation discovery if the index is not ready</mark>
3. <mark>Log warnings to inform operators about skipped operations</mark>

<mark>This change affects `Indexer.Fetcher.Arbitrum.TrackingBatchesStatuses`: instead of calling `Indexer.Fetcher.Arbitrum.Workers.Confirmations.Tasks.check_new` and `Indexer.Fetcher.Arbitrum.Workers.Confirmations.Tasks.check_unprocessed` directly, they are called through `Indexer.Fetcher.Arbitrum.Workers.Confirmations.Tasks.plan` function. This modification:</mark>
- <mark>Prevents timeouts that previously caused the GenServer to crash</mark>
- <mark>Allows the batch tracking process to continue running even when the index is not ready</mark>
- <mark>Automatically resumes confirmation discovery once the index is created</mark>
- <mark>Maintains the state between iterations, ensuring no confirmations are missed during the index creation period</mark>

_The hightlighted part is added to the notepad after the main code changes have been made with the Agent._
