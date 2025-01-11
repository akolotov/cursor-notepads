## Problem

During code review of the changes for PR 11163 the following comment was received:

<comment>
I am just raising a concern here: what if the block contains so many transactions, that may cause to slow down request execution or even raising request timeout issue for concrete blocks or response payload size might exceed allowed limit? Should we try to fallback to fetching the requested data by transaction hash in that case?
</comment>

This was regarding the call to `EthereumJSONRPC.Receipts.fetch_by_block_numbers` during the invocation of `Indexer.Fetcher.Arbitrum.Workers.Backfill.fetch_receipts`. Here is the detailed call flow:

```
..Indexer.Fetcher.Arbitrum.DataBackfill.run
....Indexer.Fetcher.Arbitrum.Workers.Backfill.discover_blocks
......Indexer.Fetcher.Arbitrum.Workers.Backfill.do_discover_blocks
........Indexer.Fetcher.Arbitrum.Workers.Backfill.backfill_for_blocks
..........Indexer.Fetcher.Arbitrum.Workers.Backfill.fetch_receipts
............EthereumJSONRPC.Receipts.fetch_by_block_numbers
..............EthereumJSONRPC.Receipts.ByBlockNumber.request
..............EthereumJSONRPC.Receipts.request_and_parse
................EthereumJSONRPC.json_rpc
```

The following modules are involved in this call flow: @data_backfill.ex, @backfill.ex, @receipts.ex, @by_block_number.ex and @ethereum_jsonrpc.ex.

Indeed, since `EthereumJSONRPC.Receipts.fetch_by_block_numbers` processes multiple blocks and creates a batch call to the RPC node, the node's processing time can be substantial. Even if the response timeout isn't exceeded, the node might refuse to respond when the requested data volume is too large.

Unfortunately, we cannot reliably expect consistent error messages from different nodes to handle such errors at the `Indexer.Fetcher.Arbitrum.Workers.Backfill.fetch_receipts` level. There is no standardized error code or message we could catch to trigger a fallback mechanism that would fetch transaction receipts through individual `eth_getTransactionReceipt` RPC calls for each transaction in each block.

## Proposed Solution

In the current implementation, the batch size of `eth_getBlockReceipts` requests sent to the RPC node is specified by `chunk_size` in `fetch_receipts`. When `fetch_by_block_numbers` fails, the code could be modified to divide the chunk size by 2 and send `eth_getBlockReceipts` batches for each half of the initial chunk. If either half fails, it continues dividing chunks by 2 until only one block remains for `fetch_by_block_numbers`. If a single block request fails, the code falls back to requesting individual receipts using `eth_getTransactionReceipt`.

This approach is particularly suitable for Arbitrum chains since individual blocks typically contain few transactions. It isolates blocks with large transaction volumes that cause issues and fetches individual receipts only for those blocks. In all other cases, the code continues using `eth_getBlockReceipts` with smaller chunks.

## High Level Scope of Changes

The module `Indexer.Fetcher.Arbitrum.Workers.Backfill` in 
`apps/indexer/lib/indexer/fetcher/arbitrum/workers/backfill.ex` is updated to narrow down problematic blocks by recursively splitting block ranges into smaller chunks upon errors and finally falls back to per-transaction receipt requests if single-block `eth_getBlockReceipts` calls still fail.

1. **Per-Transaction Receipts**
    A new private function `fetch_individual_receipts` fetches transaction receipts one at a time for a given block.
    - It retrieves the `Explorer.Chain.Block` with transactions using `Indexer.Fetcher.Arbitrum.Utils.Db.Common.rollup_blocks/1`.
    - It extracts transaction hashes from the block and groups them into chunks based on the original `chunk_size` configuration.
    - For each chunk, it calls `EthereumJSONRPC.Receipts.fetch` to retrieve individual receipts.
    - If any receipt calls fail, it returns `{:error, reason}`, triggering the existing retry logic.
    - On success, it returns `{:ok, [receipts]}`

2. **Recursive Halving Logic**
    A new private function `fetch_receipts_with_fallback` replaces thge implementation of `fetch_receipts` so now it handles receipt fetching for block numbers using the current `chunk_size` similarly
    - It splits the block numbers into chunks of size `chunk_size`.
    - It calls `EthereumJSONRPC.Receipts.fetch_by_block_numbers` for each chunk:
        - On success, it adds the receipts to an accumulator.
        - On error, it halves `chunk_size` and recursively attempts to fetch that chunk's block numbers by calling `fetch_receipts_with_fallback` until either succeeding or reaching `chunk_size == 1`.
        - If `chunk_size == 1` and the error persists, it falls back to `fetch_individual_receipts`.
    - It combines all successful results into a single `{:ok, all_receipts}` or returns `{:error, reason}` if any chunk fails.
    - It maintains compatibility with existing upstream code in `backfill.ex` by returning either `{:ok, receipts}` or `{:error, reason}`.

3. **Adapt `backfill_for_blocks`**
    In the `backfill_for_blocks/3` function, replace the existing `fetch_receipts` call with the new `fetch_receipts_with_fallback`.
    
4. **Logging**
    Throughout the new fallback logic in `fetch_receipts_with_fallback`, add logging statements using `log_info` or `log_warning` to track: 
    - When chunk processing fails and triggers recursive halving of the `chunk_size`
    - When the system falls back to per-transaction processing for an individual block
