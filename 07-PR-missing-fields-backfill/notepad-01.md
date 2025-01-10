The [PR #10067](https://github.com/blockscout/blockscout/pull/10067) extended the DB tables `blocks` and `transactions` to store Arbitrum-specific fields: `send_count`, `send_root`, and `l1_block_number` for blocks, and `gas_used_for_l1` for transactions. The `EthereumJSONRPC` application was modified to properly parse JSON RPC responses and extract these fields from block and transaction receipt structures. When existing Blockscout instances are upgraded with the new backend release, the realtime block fetcher will populate these fields with Arbitrum-specific information for new blocks and transactions. For new instances, both realtime and catchup block fetchers will populate these Arbitrum-specific fields.

However, for existing Blockscout instances, while historical blocks and transactions are fully indexed, the newly added Arbitrum-specific fields will not be populated for content indexed by the previous release, making this information unavailable in the UI for historical entries.

### Proposed Solution

A new task, running in parallel with other backend processes such as realtime block indexer, will identify blocks where the `blocks` table contains null values in the `send_count`, `send_root`, or `l1_block_number` fields. It will process blocks sequentially from the most recently indexed block to the first indexed rollup block. For each identified block, the system will make an `eth_getBlockByNumber` RPC call to retrieve the block structure and an `eth_getBlockReceipts` call to fetch all transaction receipts. The data from these RPC calls will then update the corresponding records in the `blocks` and `transactions` tables.

The new task will be built on top of the `BufferedTask` rather than directly on `GenServer` (examples of direct `GenServer` implementations include `RollupMessagesCatchup` and `TrackingMessagesOnL1`). Here are the advantages and disadvantages:

- `BufferedTask` provides built-in retry and rescheduling mechanisms
- Existing `GenServer`-based fetchers require explicit retry logic and don't handle task failures gracefully
- Sequential processing of block ranges is naturally handled by `BufferedTask's` queue
- Future fetchers could reuse this pattern with different batch sizes and concurrency settings

The backfill fetcher starts from the most recently indexed block (head of the chain) for these reasons:

- Starting from the head eliminates expensive DB queries to find the highest block with missing data
- The head is already known, eliminating the need for manual configuration of the starting block
- Processing from recent to older blocks ensures that newer, more relevant data is updated first, while older, less frequently accessed blocks are updated later

The backfill process identifies missing data by querying the database for blocks that lack one or more fields (`send_count`, `send_root`, or `l1_block_number`) without checking for missing transaction fields. This approach works because block and transaction updates are atomic operations in the block fetcher. When a block lacks these fields, it's highly likely that the transactions within that block also lack their corresponding `gas_used_for_l1` values.

## Changelog

### `EthereumJSONRPC` application

1. Similarly to the module `EthereumJSONRPC.Block.ByNumber` in `apps/ethereum_jsonrpc/lib/ethereum_jsonrpc/block/by_number.ex`, the module `EthereumJSONRPC.Receipts.ByBlockNumber` is added and located in `apps/ethereum_jsonrpc/lib/ethereum_jsonrpc/receipts/by_block_number.ex`.

2. For consistency, the functionality to build the request for `eth_getTransactionReceipt` is moved from the module `EthereumJSONRPC.Receipts` (`apps/ethereum_jsonrpc/lib/ethereum_jsonrpc/receipts.ex`) to the module `EthereumJSONRPC.Receipts.ByTransactionHash` (`apps/ethereum_jsonrpc/lib/ethereum_jsonrpc/receipts/by_transaction_hash.ex`)

3. The function `EthereumJSONRPC.Receipts.reduce_receipt` is renamed to `EthereumJSONRPC.Receipts.harmonize_responses` and its behavior is changed to support handling both single receipts and lists of receipts. For the list of receipts, the list of receipts lists is transformed into one list with all the receipts.

4. The function `EthereumJSONRPC.Receipts.response_to_receipt` is renamed to `EthereumJSONRPC.Receipts.modify_response` and its behavior is changed as follows:
    - Each clause that previously handled only one receipt now includes parallel functionality for handling lists of receipts
    - Similar to how single receipts verify response data using `Map.fetch!(id_to_transaction_params, id)`, the new list handling includes a verification step that checks whether the request ID (which represents the block number for which receipts were requested) matches the block number in each receipt from the list. Each receipt in the list is also filled with a default gas value of 0.

5. An additional clause of `EthereumJSONRPC.sanitize_responses` (in `apps/ethereum_jsonrpc/lib/ethereum_jsonrpc.ex`) is added to handle a list of request IDs as the second argument, while maintaining the same logic of assigning unmatched IDs to responses with nil IDs but using the request IDs list instead of map keys.

6. The fact that all functions (`EthereumJSONRPC.sanitize_responses`,`EthereumJSONRPC.Receipts.modify_response`, and `EthereumJSONRPC.Receipts.harmonize_responses`) invoked by `EthereumJSONRPC.Receipts.process_responses` now are able to handle the second parameter as a list of request IDs means that the functionality of `EthereumJSONRPC.Receipts.fetch` to execute the requests, standardize the responses, and then parse them to extract receipts and logs can be moved without issues into a separate function `EthereumJSONRPC.Receipts.request_and_parse`. This function accepts:
    - the list of requests
    - a map of a list which allows restoration of correspondence between the response and the entity for which the information is requested
    - configuration for JSON-RPC connection

7. The function `EthereumJSONRPC.Receipts.fetch_by_block_numbers` is introduced. It builds the list of `EthereumJSONRPC.Receipts.ByBlockNumber.request` and calls `EthereumJSONRPC.Receipts.request_and_parse` with the list of blocks as the second parameter responsible for restoring correspondence between the response and the entity for which the information is requested.

### `Explorer` application

The module `Explorer.Chain.Arbitrum.Reader.Indexer.General` (in `apps/explorer/lib/explorer/chain/arbitrum/reader/indexer/general.ex`) introduces a new function `blocks_with_missing_fields` that accepts start and end block numbers as parameters. This function identifies consensus-enabled rollup blocks from `Explorer.Chain.Block` within the specified range that lack any of these fields: `send_count`, `send_root`, or `l1_block_number`.

### `Indexer` application

1. The module `Indexer.Fetcher.Arbitrum.Utils.Db.Common` (in `apps/indexer/lib/indexer/fetcher/arbitrum/utils/db/common.ex`) provides a wrapper function for `Explorer.Chain.Arbitrum.Reader.Indexer.General.blocks_with_missing_fields`.

2. The functionality of the backfill process is implemented within two modules:
    - `Indexer.Fetcher.Arbitrum.DataBackfill` in `apps/indexer/lib/indexer/fetcher/arbitrum/data_backfill.ex`. This module serves as the entry point and orchestrates the backfill process using `Indexer.BufferedTask` functionality.
    - `Indexer.Fetcher.Arbitrum.Workers.Backfill` in `apps/indexer/lib/indexer/fetcher/arbitrum/workers/backfill.ex`. This module is called by `Indexer.Fetcher.Arbitrum.DataBackfill` to handle the backfilling of missing Arbitrum-specific data for indexed blocks and their transactions.

3. The orchestrator module `Indexer.Fetcher.Arbitrum.DataBackfill` has the following components:
    - [**Process Initialization**] During initialization, `child_spec` reads several configuration parameters: `json_rpc_named_arguments` from init options, `first_block` from the indexer application configuration, `rollup_chunk_size` from the `Indexer.Fetcher.Arbitrum` module, and `backfill_blocks_depth` and `recheck_interval` from the `DataBackfill` module. It then constructs buffered task init options by merging `child_spec` init options with the module state containing a `config` field composed of these parameters. Finally, it calls the Supervisor's `child_spec` with a `transient` restart option, allowing the `DataBackfill` process to stop once the backfill process completes.
    - [**Queue Initialization**] The callback `init` puts the first task in the buffer. This task waits for a new indexed block in the database. Using the start timestamp, the system will identify this block as the first one that appears after the backfill process begins.
    - [**Waiting For New block**] The first clause of the callback `run` queries the database for a newly indexed block using `Indexer.Fetcher.Arbitrum.Utils.Db.Common.closest_block_after_timestamp`. If the block is found, the first backfill task is scheduled to check blocks up to the block preceding the new indexed block. If no block is found, the callback finishes with status `retry`, causing the same task to be added back to the tasks queue.
    - [**Backfill Task**] The second clause of the callback`run`operates with a task described as a tuple`{timeout, end_block}`. If the timeout is greater than the current time, the callback finishes with the status`retry`, so the task with the same description will be added back to the tasks queue. Otherwise, `Indexer.Fetcher.Arbitrum.Workers.Backfill.discover_blocks` is called with the given`end_block`and the process's state. Depending on the result of the`discover_blocks`call, one of the following occurs:
        - If the call succeeds and blocks in the database were inspected up to `end_block`, a helper function plans the next backfill task. This function receives the block preceding the first inspected block in the current range, compares it with the first rollup block, and determines whether to stop the backfill process or continue by adding a new task to the queue.
        - If the call indicates that some blocks in the database are not yet indexed, a new task to re-inspect the same block interval is added to the queue with its timeout shifted by `recheck_interval` from the current time.
        - If the call returns any other error, the callback returns `retry`, so the task with the same description will be added back to the tasks queue.
    - [**Buffered Task Configuration**] The backfill task executes every two seconds. Sequential processing is enforced through default configuration values (`@default_max_concurrency = 1`), ensuring only one task can run at a time to prevent overloading the RPC and database. Since each task depends on the result of the previous task (specifically, the start block of the checked range), combining tasks into batches would not be beneficial (`@default_max_batch_size = 1`).

4. The module `Indexer.Fetcher.Arbitrum.Workers.Backfill` consists of the following parts:
    - [**Entry Point**] The public function `discover_blocks` handles a single backfilling task. It calculates a block range using the given end block and backfill blocks depth. After ensuring the calculated start block is not lower than the rollup first block, it verifies that all blocks in this range are indexed by calling `Indexer.Fetcher.Arbitrum.Utils.Db.Common.indexed_blocks?`. Finally, it passes the start and end blocks along with the state to a helper function `do_discover_blocks`.
    - [**Main Helper**] The helper `do_discover_blocks` queries blocks with missing Arbitrum-specific fields using `Indexer.Fetcher.Arbitrum.Utils.Db.Common.blocks_with_missing_fields`. It then passes these block numbers to the `backfill_for_blocks` helper function.
    - [**Actualize Data**] The helper`backfill_for_blocks`retrieves both block data and transaction receipts for the specified block numbers. This data is then used to update the database.
    - [**Notes For RPC Calls**] To request both block data and receipts, the block numbers are split into chunks. For each chunk, either `EthereumJSONRPC.fetch_blocks_by_numbers` or `EthereumJSONRPC.Receipts.fetch_by_block_numbers` is called. If either function returns `{:error, reason}` for any chunk, the data retrieval process halts and returns that error tuple.
    - [**First Note For Database Update**] Both `EthereumJSONRPC.fetch_blocks_by_numbers` and `EthereumJSONRPC.Receipts.fetch_by_block_numbers` return lists of entities ready for database import. To optimize database operations, only required fields are updated. To maintain atomicity of block and transaction updates, `Ecto.Multi.update_all` accumulates all updates for the given block range before passing them to `Explorer.Repo.transaction`.
    - [**Second Note For Database Update**] The update query for blocks operates with `Explorer.Chain.Block`. For each block that matches the hash received from RPC, it updates `send_count`, `send_root`, and `l1_block_number`. The `updated_at` field must reflect the current date and time.
    - [**Third Note For Database Update**] The update query for transactions operates with `Explorer.Chain.Transaction`. For each transaction that matches the transaction hash of the receipt received from RPC, it updates the `gas_used_for_l1` field. The `updated_at` field must reflect the current date and time.

5. The module `Indexer.Supervisor` (in `apps/indexer/lib/indexer/supervisor.ex`) is updated to add `Indexer.Fetcher.Arbitrum.DataBackfill.Supervisor` to the list of `basic_fetchers` with options for `json_rpc_named_arguments` and `memory_monitor` (like most other fetchers in this module). The wrapper `configure` is not needed to add the fetcher to the list, as `DataBackfill.Supervisor` relies on the `disabled?` parameter to control supervisor startup. This works because `Indexer.Fetcher.Arbitrum.DataBackfill` uses the `Indexer.Fetcher` macros that define the supervisor behavior. The supervisor and its worker are configured with `:transient` restart strategy to ensure graceful shutdown once the backfill process completes.

### Backfiller configuration

The script `config/runtime.exs` is updated with the following configuration parameters:

1. For the module `Indexer.Fetcher.Arbitrum.DataBackfill` in the `indexer` application:
    - `recheck_interval` is set to the value of `INDEXER_ARBITRUM_DATA_BACKFILL_UNINDEXED_BLOCKS_RECHECK_INTERVAL`, defaulting to "120s" if not specified.
    - `backfill_blocks_depth` is set to the value of `INDEXER_ARBITRUM_DATA_BACKFILL_BLOCKS_DEPTH`, defaulting to "500" if not specified.

2. For the module `Indexer.Fetcher.Arbitrum.DataBackfill.Supervisor` in the `indexer` application:
    - `disabled?` is set to false only when the chain type is `arbitrum` and `INDEXER_ARBITRUM_DATA_BACKFILL_ENABLED` is true.
