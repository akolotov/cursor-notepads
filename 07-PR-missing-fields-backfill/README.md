## Initial Changes 

Although [naive-notepad.md](naive-notepad.md) was used to produce initial changes, in fact, it was the first time when I tried a notepad to assist me in the coding. That is why this experience could be considered as semi-successful.

Something was generated and looked promising, while the rest was finalized manually.

Here is the log of prompts that were made in the Cursor composer (_mode_: `normal`, _model_: `claude-3.5-sonnet-20241022`):

> Read naive-notepad.md. Don't start implement any code but raise questions if any appear after the getting familiar with the document.

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

## Main Solution Text

The document [notepad-01.md](notepad-01.md) was written manually based on the changes which were produced at the end of the previous section.

The equivalence of the changes described in the document and the actual code functionality was verified by using the Cursor composer (in the `agent` mode).

Here is the log:

> Take a look at @by_block_number.ex , @by_transaction_hash.ex , @ethereum_jsonrpc.ex and @receipts.ex and verify that all important changes (changes that don't include updates in the documentation and specifications) are reflected in the section "`EthereumJSONRPC` application" of  notepad-01.md 

> Right, the function `check_equivalence` is not mentioned explicitely but it is covered by the second bullet in the step 4. Do you agree with this? Although the list of changes in the notepad do not require to introduce a separate function for this, it was written for better code readability. Do you still think that it needs to be reflected in the document?

> Take a look at @general.ex and verify that the functionality described in "Explorer application" of notepad-01.md is implemented.

> Take a look at @common.ex and confirm that the functionality described in the step 1 of "Indexer application" of notepad-01.md  is implemented.

> Take a look at @data_backfill.ex  and confirm that the functionality described in "Process Initialization" of the step 3 of "Indexer application" of notepad-01.md is implemented. If something important exists in the implementation but absent in the notepad, let me know. Also attaching @buffered_task.ex and @backfill.ex for the context.

Since the composer highlighted some uncertancy about the changes description, it was decided to involve it to the text improvement:

> What is your suggestion to adjust "Buffered Task Configuration" section to make it more clear?

> I changed it in the following form:
> ```
> [**Buffered Task Configuration**] The backfill task executes every two seconds. Sequential processing is enforced through default configuration values (`@default_max_concurrency = 1`), ensuring only one task can run at a time to prevent overloading the RPC and database. Since each task depends on the result of the previous task (specifically, the start block of the checked range), combining tasks into batches would not be beneficial (`@default_max_batch_size = 1`).
> ```

> Take a look at @data_backfill.ex  and confirm that the functionality described in "Queue Initialization" of the step 3 of "Indexer application" of notepad-01.md  is implemented. If something important exists in the implementation but absent in the notepad, let me know. Also attaching @buffered_task.ex and @backfill.ex for the context.

> > [A quote from the compoer's response] While this is an implementation detail that follows the BufferedTask behavior requirements (as seen in buffered_task.ex), it might be worth mentioning in the notepad for completeness, especially since it shows how the task initialization integrates with the BufferedTask framework. Would you like me to suggest an addition to make this clearer in the notepad?
> 
> Take a look at the step 2 of "Indexer application". It already states that the functionality of `BufferedTask` is utilized. Do you think it is not enough and the "Queue Initialization" should be adjusted additionally to clarify the changes?

> Take a look at @data_backfill.ex  and confirm that the functionality described in "Waiting For New block" of the step 3 of "Indexer application" of notepad-01.md is implemented. If something important exists in the implementation but absent in the notepad, let me know. Also attaching @buffered_task.ex and @backfill.ex for the context.

> Take a look at @data_backfill.ex  and confirm that the functionality described in "Backfill Task" of the step 3 of "Indexer application" of notepad-01.md  is implemented. If something important exists in the implementation but absent in the notepad, let me know. Also attaching @buffered_task.ex and @backfill.ex for the context.

> Take a look at @backfill.ex  and confirm that the functionality described in "Entry Point" of the step 4 of "Indexer application" of notepad-01.md is implemented. If something important exists in the implementation but absent in the notepad, let me know. Also attaching @data_backfill.ex and @common.ex  for the context.

> Take a look at @backfill.ex  and confirm that the functionality described in "Main Helper" of the step 4 of "Indexer application" of notepad-01.md is implemented. If something important exists in the implementation but absent in the notepad, let me know. Also attaching @data_backfill.ex and @common.ex  for the context.

> Take a look at @backfill.ex and confirm that the functionality described in "Actualize Data" of the step 4 of "Indexer application" of notepad-01.md  is implemented. Take into consideration the information from "Notes For RPC Calls". If something important exists in the implementation but absent in the notepad, let me know. Also attaching @ethereum_jsonrpc.ex , @receipts.ex for the context.

> Take a look at @backfill.ex and confirm that the functionality described in "Notes for Database Update" of the step 4 of "Indexer application" of notepad-01.md is implemented. If something important exists in the implementation but absent in the notepad, let me know. Also attaching @repo.ex , @block.ex and @transaction.ex  for the context.

> How the call of `safe_insert_all` correlates with the database update in that manner which is implemented in @backfill.ex ?

> Take a look at @supervisor.ex  and confirm that the functionality described in the step 5 of "Indexer application" of notepad-01.md  is implemented. If something important exists in the implementation but absent in the notepad, let me know. Also attaching @data_backfill.ex and @fetcher.ex for the context.

As part of the text with the verification steps, the composer produced the following:

```
The supervisor is configured with a `:transient` restart strategy by default through the `Indexer.Fetcher` macro:

`restart = Keyword.get(opts, :restart, :transient)`

While this is briefly mentioned in the notepad under the "Process Initialization" section ("calls the Supervisor's child_spec with a transient restart option"), it would be valuable to explicitly mention this in step 5 since it's an important part of the supervisor configuration that allows the process to stop gracefully once the backfill process completes.
```

The reaction was the following prompt:

> It is a good point let's discuss it. 
> 
> Indeed, there is a line `use Indexer.Fetcher, restart: :transient` in @data_backfill.ex as well as
> ```
> Supervisor.child_spec({BufferedTask, [{__MODULE__, buffered_task_init_options}, gen_server_options]},
>   id: __MODULE__,
>   restart: :transient
> )
> ```
> 
> Apply you best from the Elixir knowledge to figure out if they duplicate to each other or one complement another?

After that, the composer confirmed that all the required changes are covered in the document.

> So, with this clarification may you specify if the "Indexer application" section of the notepad should be adjusted or it contains all necessary information?

At this point, the composer mixed two concepts together, so it was necessary to highlight this to it.

> Not sure how "with `:transient` restart strategy" correlate with "disabled?" parameter? You extended the sentence explaning why "disabled?" makes sense.

> Take a look at @runtime.exs  and confirm that the functionality described in the section "Backfiller configuration" of notepad-01.md  is implemented. If something important exists in the implementation but absent in the notepad, let me know.
