The changes were made without using a notepads. All the implementation details were provided through the prompts in one Composer session.

**The Composer mode:** agent

### First Iteration

Initial state of the code base was as per [this commit](https://github.com/blockscout/blockscout/commit/e3033cf3ad905aa01f7e607715116c4e4f7be5af).

**The composer model:** `claude-3-5-sonnet-20241022`

----

> Take a look at `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement.l1_block_of_latest_confirmed_block` and write a function that will allow to get L1 block of the confirmation for the specific rollup block. 
>
> @batch_block.ex and @lifecycle_transaction.ex are added for reference. 

----

> Consider to return a tuple of `status` and `block` to differentiate a normal case when the block is not confirmed yet and when an inconsistency discovered.

----

> In fact "not found" must be equal to "not confirmed". No need to differentiate these two cases.

----

> In @settlement.ex  write a wrapper for `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement.l1_block_of_confirmation_for_rollup_block` which will return nil for `:not_confirmed` or `:error`.

----

*In fact, this prompt was created by first drafting the function and then asking OpenAI O1 to prepare the function implementation description as if it were instructing a junior developer.*

> Implement a private function in @tasks.ex  that determines the lowest L1 block number from which to start discovering block confirmations in a rollup system.
> 
> Input:
> - A state map containing:
>   - config: Configuration with L1 rollup initialization block and first rollup block numbers
>   - data: Optional cached L1 block confirmation number
> 
> Expected behavior:
> 1. Check if there's a cached L1 block confirmation number in the state
> 2. If cached number exists, return it with unchanged state
> 3. If no cached number:
>    - Determine the lowest rollup block (1 if first block is 0, otherwise use the first block value)
>    - Query database by `Indexer.Fetcher.Arbitrum.Utils.Db.Settlement.l1_block_of_confirmation_for_rollup_block` (@settlement.ex ) for L1 confirmation block for this rollup block
>    - If no confirmation found, use L1 rollup init block
>    - If confirmation found, cache it in state and return it
> 
> Return value: Tuple with {block_number, updated_state}

----

As it it clear from the prompt below, the agent did not follow the instructions properly.

> Wait! the block number must be cached only if `l1_block_of_confirmation_for_rollup_block` returned not nil.

----

> It makes no sense to discover unprocessed confirmations in `Indexer.Fetcher.Arbitrum.Workers.Confirmations.Tasks.check_unprocessed` of @tasks.ex  below confirmation of the very first rollup block. So, utilize `get_lowest_l1_block_for_confirmations` to find the corresponding L1 block and use it instead of `l1_rollup_init_block` in `check_unprocessed`.

----

After this prompt changes [were committed](https://github.com/blockscout/blockscout/commit/81c35ade52d17bfd2e36ae52b00ce5f70a1f332d).

### Second Iteration

Initial state of the code base for the second iteration was as per [this commit](https://github.com/blockscout/blockscout/commit/7568df393cd9e6e14b02476bc09b1c751cebaf32).

**The composer model** is still `claude-3-5-sonnet-20241022`

---- 

> Implement the apprach similar to `Indexer.Fetcher.Arbitrum.Workers.Confirmations.Tasks.check_unprocessed` identifying lowest block for confirmation in `Indexer.Fetcher.Arbitrum.Workers.Batches.Tasks.check_historical` and `Indexer.Fetcher.Arbitrum.Workers.Batches.Tasks.inspect_for_missing` to identify lowest block for batch commitments. Obviously, it should be the parent chain block of the batch comitting the first rollup block. Similar to confirmations, the rollup block 0 is not part of any batch.
> 
> Before changing the code, think step-by-step for the plan which modules the changes must be implemented in.

----

Probably, because the agent was not instructed properly on the previous step, it provided an unappropriate solution. That is why it was directed explicitely:

> 1. Instead of implementing changes for `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement` and `Indexer.Fetcher.Arbitrum.Utils.Db.Settlement` evaluate if they already have functionality which can be re-used.
> 
> 2. Are you sure that the state passed to `check_historical` and `inspect_for_missing` contains all required data to get lowest l1 block?

----

> Honestly, I am not sure about existance of `rollup_first_block` in the state. Double check in @tracking_batches_statuses.ex where these functions are being invoked.

----

And interesting observation here - befire the previous prompt, I missed the fact that some functionality already exist. The agent don't agree with me and pointed me to that.

> Ah, ok. I missed the part that `rollup_first_block` already initialized to be used to limit the parent chain block to discover confirmations. Thanks! So, summarize the plan about the changes to implement required functionality in @tasks.ex 

----

> Proceed with implementation.

----

> Ok. The state is updated within these functions but it is not returned to the callers which are in @tracking_batches_statuses.ex . Plan the corrsponding changes.

----

> Implement these changes

----

> `l1_rollup_init_block` is not needed any more to be matched in the definition of the changed functions in @tasks.ex  , right?

----

> I would keep it in the specification since it will indicate that it is required by functions which are below in the stack

----

After this prompt changes [were committed](https://github.com/blockscout/blockscout/commit/6e6b9882aa25fa15d8eec92a270f21c83e8e1b7c).


### Third Iteration

The code base for the third iteration is the commit of the previous iteration.

**The composer model** is still `claude-3-5-sonnet-20241022`

----

> Consider the fact that cross-chain messages executions on the parent chain cannot be performed before the confirmations of the corresponding rollup blocks where these messages were included.
> 
> Therefore, it looks logical to make `Indexer.Fetcher.Arbitrum.Workers.Confirmations.Tasks.get_lowest_l1_block_for_confirmations` public and re-use it in `Indexer.Fetcher.Arbitrum.Workers.NewL1Executions.discover_historical_l1_messages_executions`. May plan the corresponding changes?

----

> Should the corresponding changes in @tracking_batches_statuses.ex also be done?

----

> You don't need in a new handler. Take a look at the existing handler defined as `def handle_info(:check_historical_executions, state)`

----

The next prompt with `claude-3-5-sonnet-20241022` did not work properly, that is why I switched to `o3-mini`.

> implement these changes. Make sure that you understand the files where the changes must be done:
> - @tasks.ex 
> - @new_l1_executions.ex 
> - @tracking_batches_statuses.ex 

The model prepared wery good plan for changes, so implement the plan I switched back to `claude-3-5-sonnet-20241022` (Cursor was not able to generate code with `o3-mini` model by some reason).

----

> proceed with these changes

----

Even with the very good plan, the agent struggled to modify Elixir code without issues:

> 1. Consider to move the part of the documentation comment "This function is used to optimize historical discovery processes . . . both confirmation and message execution discovery" from @tasks.ex to  `Indexer.Fetcher.Arbitrum.Workers.NewL1Executions.discover_historical_l1_messages_executions` as a comment before invocation of `get_lowest_l1_block_for_confirmations` to explain why it is needed.
> 
> 2. Consider to add the state as returning value in the specification of `discover_historical_l1_messages_executions`.
> 
> 3. Return sending of `:check_lifecycle_transactions_finalization` instead of `check_new_executions` back in the handler of `:check_historical_executions`.

----

After this prompt changes [were committed](https://github.com/blockscout/blockscout/commit/710fe8f40161c1c2d147569b76ce00888438b22a).
