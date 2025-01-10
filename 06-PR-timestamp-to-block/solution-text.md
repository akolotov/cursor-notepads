## Problem

During testing of PR #11163, it was discovered that the functionality for getting block numbers after specified timestamps (`Explorer.Chain.timestamp_to_block_number`) doesn't work as expected. Here are the issues found during investigation.

### 1. Unpredictable Results When Multiple Blocks Share the Same Timestamp

For chains that produce blocks with intervals under 1 second (like Arbitrum chains), multiple blocks can share the same timestamp. However, the `gt_timestamp_query` and `lt_timestamp_query` queries in the `Explorer.Chain.timestamp_to_block_number` function don't account for this scenario. Since each query returns only one block number, there's no guarantee which block will be selected from the set of blocks with `given_timestamp`, as the selection depends on the database's internal ordering behavior and query execution plan.

**Example:**

The following records exist in `Explorer.Chain.Block` , and `given_timestamp` set to "2025-01-08 22:04:45"

```
5005, 2025-01-08 22:04:47 <-- will be chosen by gt_timestamp_query
5004, 2025-01-08 22:04:44
5003, 2025-01-08 22:04:44
5002, 2025-01-08 22:04:44 <-- will be chosen by lt_timestamp_query
```

### 2. Request Condition Violation

`Explorer.Chain.timestamp_to_block_number` executes `lt_timestamp_query |> subquery() |> union(^gt_timestamp_query)` to find the block closest to `given_timestamp`. This can lead to two incorrect scenarios:

- A block with a higher number is chosen when `:before` is passed as the parameter `closest`
- A block with a lower number is chosen when `:after` is passed as the parameter `closest`

Next, the chosen block is passed to `Explorer.Chain.get_block_number_based_on_closest`, which invokes either `Explorer.Chain.BlockNumberHelper.previous_block_number` or `Explorer.Chain.BlockNumberHelper.next_block_number`, depending on the case.

```
..Explorer.Chain.timestamp_to_block_number
....Explorer.Chain.get_block_number_based_on_closest
......Explorer.Chain.BlockNumberHelper.{previous_block_number,next_block_number}
........Explorer.Chain.BlockNumberHelper.neighbor_block_number
..........Explorer.Chain.BlockNumberHelper.move_by_one
```

These functions simply increment or decrement the block number based on the direction (`previous` or `next`), without verifying the timestamps of the resulting blocks.

Since `gt_timestamp_query` and `lt_timestamp_query` don't guarantee returning subsequent blocks (as noted in the first issue), `Explorer.Chain.get_block_number_based_on_closest` might return a block that violates the condition specified in the `closest` parameter.

For instance, in the example above where `closest` is `after`, block `5003` will be returned even though its timestamp is **before** the given timestamp.

### 3. Non-indexed Block

Consider another example where the database has only two records (blocks `5006` and `5007` are not yet indexed), `given_timestamp` is "2025-01-08 22:04:50", and `closest` is `after`:

```
5008, 2025-01-08 22:04:55 <-- will be chosen by gt_timestamp_query
5005, 2025-01-08 22:04:47 <-- will be chosen by lt_timestamp_query
```

The function `Explorer.Chain.timestamp_to_block_number` will return `5006` even though this block isn't in the Blockscout database. This occurs because after selecting candidate blocks from `Explorer.Chain.Block` using `gt_timestamp_query` and `lt_timestamp_query`, the adjustment process described in issue #2 doesn't verify whether the calculated block number exists in `Explorer.Chain.Block`.

## Proposed Solution

The first issue is resolved by including the `number` field of `Explorer.Chain.Block` in the ordering instructions for queries that identify blocks closest to the given timestamp.

The behaviors described in the second and third issues may be intentional, so it was decided to separate the current behavior from the behavior needed for changes in PR #11163. A new parameter `strict` to the `timestamp_to_block_number` function is introduced. When set to `true`, the function selects either `gt_timestamp_query` or `lt_timestamp_query` based on the direction specified in `closest`. This ensures the function will either return an indexed block that matches the `closest` requirement or return `:not_found` if no blocks exist before or after the given timestamp.

## High Level Scope of Changes

### Minimal Required Changes

1. Since the `Explorer.Chain` module exceeds 5,500 lines of code and smaller modules are preferable (see explanation here: [`apps/explorer/lib/explorer/chain/arbitrum/reader/README.md`](https://github.com/blockscout/blockscout/blob/7e522eb194dba4e141ad9b5ea965cb1e3c47f66b/apps/explorer/lib/explorer/chain/arbitrum/reader/README.md)), it is decided to move the functions `timestamp_to_block_number` and `get_block_number_based_on_closest` to a new module `Explorer.Chain.Block.Reader.General` (`apps/explorer/lib/explorer/chain/block/reader/general.ex`).

2. Since the `gt_timestamp_query` and `lt_timestamp_query` queries share much functionality, they are implemented through three functions: `build_directional_query`, `adjust_query_to_highest_block`, and `adjust_query_to_lowest_block`. The `direction` parameter passed to `build_directional_query` determines which query to compose. `build_directional_query` creates the common query structure, while `adjust_query_to_highest_block` and `adjust_query_to_lowest_block` add conditions for selecting blocks before or after the given timestamp and define the element ordering. As described in the "Proposed Solution" section, the ordering instructions now include block numbers.

3. The boolean parameter `strict` is added to the `Explorer.Chain.Block.Reader.General.timestamp_to_block_number function` specification. The previous implementation of `timestamp_to_block_number` remains under the clause `timestamp_to_block_number(given_timestamp, closest, from_api, false)`. The clause `timestamp_to_block_number(given_timestamp, closest, from_api, true)` composes one query through `build_directional_query` based on `closest`, runs the query using the repo according to `from_api`, and handles the response as follows:
    - if it's `nil`, return `{:error, :not_found}`
    - if a block number is found, return it.

4. It is assumed that if the `strict` parameter is not passed to the function, its default value will be `false`. Therefore, besides interacting with the new module, only minimal changes are needed in these modules:
    - `Explorer.Chain.AdvancedFilter` (`apps/explorer/lib/explorer/chain/advanced_filter.ex`)
    - `BlockScoutWeb.API.RPC.BlockController` (`apps/block_scout_web/lib/block_scout_web/controllers/api/rpc/block_controller.ex`)
    - `Explorer.Chain.Transaction.History.Historian` (`apps/explorer/lib/explorer/chain/transaction/history/historian.ex`)

These changes only involve updating the module reference for calling `timestamp_to_block_number`.

Additionally, the function `Indexer.Fetcher.Arbitrum.Utils.Db.Common.closest_block_after_timestamp` in `apps/indexer/lib/indexer/fetcher/arbitrum/utils/db/common.ex` has been modified to call `timestamp_to_block_number` with the `strict` parameter set to `true`.
    
### Consistency Support

Since the `strict` parameter set to `true` may be used in scenarios beyond Arbitrum in the future, it need to be ensure the `timestamp_to_block_number` function works correctly with other chain types. This is particularly important because returning the identified block number could be unsafe when Blockscout indexes a Filecoin family chain, as the block could be a null round. When `timestamp_to_block_number` runs with `strict` set to `false` in a `:filecoin` chain type environment, it checks if the current block is a null round and finds another suitable block.

```
..Explorer.Chain.Block.Reader.General.timestamp_to_block_number
....Explorer.Chain.Block.Reader.General.get_block_number_based_on_closest
......Explorer.Chain.BlockNumberHelper.{previous_block_number,next_block_number}
........Explorer.Chain.BlockNumberHelper.neighbor_block_number
..........Explorer.Chain.NullRoundHeight.neighbor_block_number
```

To maintain consistency, the implementation of `timestamp_to_block_number` when `strict` is set to `true` should be adjusted to avoid returning blocks that are null rounds.

The functions `Explorer.Chain.BlockNumberHelper.neighbor_block_number` and `Explorer.Chain.NullRoundHeight.neighbor_block_number` cannot be reused directly because they might return non-indexed blocks. To maintain the behavior of `timestamp_to_block_number` with `strict` set to `true` across different chain types, the following changes are proposed:

### Consistency Support

Since the `strict` parameter set to `true` may be used in scenarios beyond Arbitrum in the future, it need to be ensure the `timestamp_to_block_number` function works correctly with other chain types. This is particularly important because returning the identified block number could be unsafe when Blockscout indexes a Filecoin family chain, as the block could be a null round. When `timestamp_to_block_number` runs with `strict` set to `false` in a `:filecoin` chain type environment, it checks if the current block is a null round and finds another suitable block.

```
..Explorer.Chain.Block.Reader.General.timestamp_to_block_number
....Explorer.Chain.Block.Reader.General.get_block_number_based_on_closest
......Explorer.Chain.BlockNumberHelper.{previous_block_number,next_block_number}
........Explorer.Chain.BlockNumberHelper.neighbor_block_number
..........Explorer.Chain.NullRoundHeight.neighbor_block_number
```

To maintain consistency, the implementation of `timestamp_to_block_number` when `strict` is set to `true` should be adjusted to avoid returning blocks that are null rounds.

The functions `Explorer.Chain.BlockNumberHelper.neighbor_block_number` and `Explorer.Chain.NullRoundHeight.neighbor_block_number` cannot be reused directly because they might return non-indexed blocks. To maintain the behavior of `timestamp_to_block_number` with `strict` set to `true` across different chain types, the following changes are proposed:

**1. Discovering Only Indexed Blocks as Non-null Rounds**

The module `Explorer.Chain.NullRoundHeight` (`apps/explorer/lib/explorer/chain/null_round_height.ex`) is enhanced:

1.1. Renamed `neighbors_query` to more descriptive `neighboring_null_rounds_query` to better reflect that it queries null round heights

1.2. Added new private helper function `null_round?` to check if a given block height represents a null round

1.3. Added new function `find_next_non_null_round_block` that:

- Takes a block number and direction (`:previous` or `:next`) as parameters
- Checks if the given block is a null round and returns{:ok, block_number}. Since most blocks passed to this function are not null rounds, it makes sense to perform this check first to avoid making heavier database calls.
- Otherwise initiates a search process through `process_next_batch`

1.4. Implemented batch processing logic through new private functions:

- `fetch_neighboring_existing_blocks` to get a batch of `@existing_blocks_batch_size` existing block numbers from `Explorer.Chain.Block`, ensuring only consensus blocks are included and ordered appropriately (descending for previous, ascending for next direction)
- `fetch_neighboring_null_rounds` to get a batch of null round heights, utilizing the renamed `neighboring_null_rounds_query` function
- `process_next_batch` to orchestrate the search process by:
    - Calling `fetch_neighboring_existing_blocks` and `fetch_neighboring_null_rounds`
    - Passing batches of existing blocks and null rounds to `find_first_valid_block` for analysis together with the direction
- `find_first_valid_block` to identify the first valid block that:
    - Returns `{:error, :not_found}` if empty list of existing blocks (no blocks were found) is passed
    - Searches through existing blocks to find the first one that is not in the null rounds set
    - If all passed existing blocks are null rounds, recursively calls `process_next_batch` starting from the last block number
    - If a block that is not a null round is found, returns `{:ok, block_number}`

**2. Helper Extension**

The function `Explorer.Chain.BlockNumberHelper.find_next_non_null_round_block` in `apps/explorer/lib/explorer/chain/block_number_helper.ex` is updated to serve as an abstraction layer that:

- For Filecoin chains, delegates to `Explorer.Chain.NullRoundHeight.find_next_non_null_round_block`
- For other chain types, simply returns `{:ok, block_number}` since null rounds don't exist

**3. Block Number Adjustment**

The clause `Explorer.Chain.Block.Reader.General.timestamp_to_block_number(given_timestamp, closest, from_api, true)` is modified to use `BlockNumberHelper.find_next_non_null_round_block` after identifying a candidate block instead of returning the block.
