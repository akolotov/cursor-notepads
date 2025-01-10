## Initial Changes 

Although [naive-notepad.md](naive-notepad.md) was used to produce initial changes, in fact, it was the first time when I tried a notepad to assist me in the coding. That is why this experience could be considered as semi-successful.

Something was generated and looked promising, while the rest was finalized manually.

Here is the log of prompts that were made in the Cursor composer (_mode_: `normal`, _model_: `claude-3.5-sonnet-20241022`):

> Read @missing-fields-backfiller . Don't start implement any code but raise questions if any appear after the getting familiar with the document.

After that, the composer requested additional information.

```
After reviewing the provided context, I have a few clarifying questions about the missing fields backfiller functionality:
1. For the fetch_receipts function:
- Should we handle any specific error cases when fetching receipts from the RPC node?
- Is there a recommended batch size for chunking the block numbers when fetching receipts?
2. For the discover_blocks function:
- Should we implement any retry mechanism if RPC calls fail?
- What should be the behavior if some blocks are found to have missing fields but we fail to fetch their - updated data?
- Should we log any statistics about the number of blocks processed/updated?
3. Regarding database updates:
- Should we implement any mechanism to track progress of the backfilling process in case it needs to be resumed?
- Is there a specific timeout we should use for the database transactions?
- Should we implement any cleanup mechanism if a transaction fails partway through?
4. General questions:
- Should this functionality run as a one-time task or should it periodically check for blocks with missing fields?
- Are there any rate limiting considerations we need to account for when making RPC calls?
- Should we implement any monitoring/alerting for this process?
These questions would help ensure we implement the functionality correctly and handle all edge cases appropriately.
```

The responding prompt was:

> 1.a. Only which already have in the mentioned function `Indexer.Block.Fetcher.Receipts.fetch`
>
> 1.b. The batch size will be specified in `state` map passed to `discover_blocks`: `%{:config => ${:rollup_rpc => %{:chunk_size => non_neg_integer(), :json_rpc_named_arguments => EthereumJSONRPC.json_rpc_named_arguments()}}}`
> 
> 2.a. no. If `fetch_blocks_by_numbers` returns `{:error, ...}` the function must return `:error`.
> 
> 2.b. looks like `fetch_blocks_by_numbers` will return `{:error, reason :: term}` in the case if any block request fails.
> 
> 2.c no so far.
> 
> 3.a no so far.
> 
> 3.b. no, let's make as in the referenced code.
> 
> 3.c my assumption that it will be handled automatically by the DB driver behind Ecto.Multi, no?
> 
> 4.a `discover_blocks` must accepts `start_block`, `end_block` and `state`.
> 
> 4.b no so far
> 
> 4.c no so far. The function will be used in a Elixir task but it is out of scope of this module.

The coding agent confirmed its new understanding of the task after provided clarifications and asked to proceed with implementing the functionality.

> Yes, let's start.

> Let's move further and proceed with `discover_blocks` 

The rest of the changes were done manually. However, the documentation for new functions and modules was written with the help of the Cursor chats. The prompts like this were used:

**Example 1:**

> Write a doc comment for the private function `update_db`

**Example 2:**

> Write a module doc comment. When describe which exactly Arbitrum specific information is backfilled in the blocks and transactions, take a look at `update_blocks` and `update_transactions` functions of the module.

**Example 3:**

> Assuming that `closest_block_after_timestamp` is implemented in @db.ex . Write a doc comment for the `run` function.

**Example 4:**

> Suggest a draft of the module `Indexer.Fetcher.Arbitrum.DataBackfill` documentation comment.

> Consider to point out that the fetcher utilize the BufferedTask functionality since it is convinent to divide the entire block range to backfill on small tasks where BufferedTask is used to take care of each task - like adding the next batch of block range for the next task or retry task.

> Consider to add an information that when the first indexed block is identified, the tasks for backfill are represented by a tuple of a timout and the end block for the block range to backfill. Normally, the timeout always expires. Exception is the case when a block range batch cannot be handled due to absense of some blocks in the database. In this case the idea of the timout to wait for some time hoping that it is enough for the catchup block fetcher to index missing blocks.

----

Eventually, all the changes produced were ended with [the following commit](https://github.com/blockscout/blockscout/tree/63270cf37e626c89143c7ade271748f810d33ad3).

Later, after the trivial code review comments addressing a new state of the repo occurred, see [this one](https://github.com/blockscout/blockscout/tree/dec46804792eea2a2d6e7a76dec4fd44e70cdfb5).
