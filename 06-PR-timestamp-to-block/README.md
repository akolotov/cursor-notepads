## Initial Code Adjustments

The Cursor's notepad feature was not used to implement the changes. Instead, Cursor chat sessions were used.

Below are the prompts that were used initially through the chat sessions. Then, manual adjustments were made: 
- Moving the function to a new module.
- Renaming the functions.
- Changes in `apps/explorer/lib/explorer/chain/block_number_helper.ex`.

Eventually, [this commit](https://github.com/blockscout/blockscout/pull/11633/commits/5923b2bdccfd5383d71ceabbb3e29180353ecabf) was prepared.

### Chat to Modify `chain.ex`

Initially, the lines from 2465 to 2601 of `apps/explorer/lib/explorer/chain.ex` were attached to the chat.

> Do you think it is possible to introduce a new parameter like `strict` to `timestamp_to_block_number` to manage if the situation whether the function behaves as it is now (as per the notes in the function documentation) or always provides the existing block strictly after or before the given timestamp?

> Since both queries used to determine the block before or after are almost identical (difference is the sign of comparison between `block.timestamp` and `^given_timestamp`. Consider to move them into a separate function.

> Looks like `consensus_blocks_query` can be part of this new function as well.

> For the line: `:after -> {>= , :asc}` I receive the following error:
> ```
> ** (SyntaxError) invalid syntax found on apps/explorer/lib/explorer/chain.ex:2637:20:
>       error: syntax error before: '>='
>       │
>  2637 │         :after -> {>= , :asc}
>       │                    ^
>       │
>       └─ apps/explorer/lib/explorer/chain.ex:2637:20Elixir
> ```

> Will it be more safer to introduce two new functions:
> - adjust_query_to_higest_block
> - adjust_query_to_lowest_block
> 
> operators `where` and `order_by` will be added there to the input query depending on the direction?

> can `limit` and `select` be the part of the query which is passed to `adjust_query_to_highest_block` and `adjust_query_to_lowest_block`? 

> Can `timestamp_to_block_number` be divided on two implementation with clauses
> - def timestamp_to_block_number(given_timestamp, closest, from_api, false)
> - def timestamp_to_block_number(given_timestamp, closest, from_api, true)

> Currently, the logic among `build_directional_query`, `adjust_query_to_highest_block` and `adjust_query_to_lowest_block` is built as so it does not guarantee that proper block will be picked up if there are several blocks with the same timestamp. The proper block for `adjust_query_to_highest_block` must be a block with the highest number, for `adjust_query_to_lowest_block` must be a block with the lowest number. Do you agree?

### Chat to Modify `null_round_height.ex` 

The next prompts in a new chat session were against `apps/explorer/lib/explorer/chain/null_round_height.ex`.

> Intorduce a new public function that will check if the given block number in a null round

> Is the query in the function more efficient than `Repo.exists?`?

> Introduce a new function in @null_round_height.ex  that will check if the given block is a null round and if it is so, try to find the next block which a) exists in the database b) is not a null round. Attaching @block.ex for the context about the block table schema.

> Does it make sense to optimize the process of querying the existing block by requesting `@existing_blocks_batch_size` at once rather than having `@existing_blocks_batch_size` of `block_exists?`? The same could be applied to the call of `!null_round?`. Instead of calling it `@batch_size`, `@batch_size` of height could be received from `null_round_heights` at once.

> It is not correct to use `start_block` and `end_block` in `fetch_existing_blocks` and `fetch_null_rounds` since sequences could be spare. In both cases consider to limit amount of elements rather than specifying the range of lookup. Look at `neighbors_query` as an example. Differentiate `@existing_blocks_batch_size` and `@batch_size`. The first is for the blocks, the second is for the null rounds. 

> Will it simplify the logic if just `b.number` will be included in the result of `fetch_existing_blocks` instead of the tuple `{b.number, b.timestamp}`?

> Consider to add the direction to `find_next_valid_block` similarly to the function `neighbor_block_number`

> Instead of introducting `build_block_where_clause` and `build_order_clause` consider:
> 1. Re-use `neighbors_query` in `fetch_next_null_rounds`.
> 2. Implement similar approach for `fetch_existing_blocks`. 

> Should the function `find_first_valid_block` take into account `direction` as well? Especially, it is required to pass it to `find_next_valid_block` if no valid blocks found.

> Are sure that `start_block` still necessary in `find_first_valid_block(existing_blocks, null_rounds, start_block, direction)`?

> Move `with .. end` from `find_next_non_null_round_block` to a separate function which will be called from `find_first_valid_block` to avoid double check (and double DB query) of the block number as the last element of `existing_blocks` in `find_first_valid_block` and as the block number passed to `find_next_non_null_round_block`.

## Solution Text

Most of the content in [solution-text.md](solution-text.md) was written in Notion to use its "Ask AI" to improve the writing of the text. However, a draft for items 1, 2, and 3 of the section "Consistency Support" was prepared with Claude (OpenAI's model 01 was not able to follow the instructions of such a prompt adequately).

The following prompt was used:

```
<documents>
  <document filename="solution-text.md">
. . .
. . .
. . .
### Consistency Support

Since the `strict` parameter set to `true` may be used in scenarios beyond Arbitrum in the future, it need to be ensure the `timestamp_to_block_number` function works correctly with other chain types. This is particularly important because returning the identified block number could be unsafe when Blockscout indexes a Filecoin family chain, as the block could be a null round. When `timestamp_to_block_number` runs with `strict` set to `false` in a `:filecoin` chain type environment, it checks if the current block is a null round and finds another suitable block.

-- three backticks are here ---
..Explorer.Chain.Block.Reader.General.timestamp_to_block_number
....Explorer.Chain.Block.Reader.General.get_block_number_based_on_closest
......Explorer.Chain.BlockNumberHelper.{previous_block_number,next_block_number}
........Explorer.Chain.BlockNumberHelper.neighbor_block_number
..........Explorer.Chain.NullRoundHeight.neighbor_block_number
-- three backticks are here ---

To maintain consistency, the implementation of `timestamp_to_block_number` when `strict` is set to `true` should be adjusted to avoid returning blocks that are null rounds.

The functions `Explorer.Chain.BlockNumberHelper.neighbor_block_number` and `Explorer.Chain.NullRoundHeight.neighbor_block_number` cannot be reused directly because they might return non-indexed blocks. To maintain the behavior of `timestamp_to_block_number` with `strict` set to `true` across different chain types, the following changes are proposed:

1. 
  </document>
  <document filename="null_round_height.ex">
. . .
. . .
  </document>
  <document filename="block_number_helper.ex">
. . .
. . .
  </document>
  <document filename="general.ex">
. . .
. . .
  </document>
</documents>

<examples>
  <example id="1">
. . .
. . .
  </example>
  <example id="2">
. . .
. . .
  </example>
  <example id="3">
. . .
. . .
  </example>
  <example id="4">
. . .
. . .
 </example>
</examples>

Take a role of the author of `solution-text.md`. As you can see the section "Consistency Support" is not finished - it does not contain the proposed changes description.
Try to analyze the code in `null_round_height.ex`, `block_number_helper.ex` and `general.ex` and proceed with the proposed changes description in the section "Consistency Support". 
Don't modify or suggest a new code. Just complete the section of the design document based on the existing code.

Above are examples how proposed changes were described in other design documents. Try to follow the same style.
```

After Claude's response, it was adjusted with (the output of `git diff master apps/explorer/lib/explorer/chain/null_round_height.ex` was attached to the message):

> Ok. To better understand what exactly was changed in the file `null_round_height.ex`, here is the delta. Reflect this in the proposed description.

The following prompts were:

> Updates of the documentation are not needed in the description. Make sure that you understand correctly the diff. `@null_rounds_batch_size` is not a new constant. Extend the description of `fetch_neighboring_null_rounds` with the information that it re-uses `neighbors_query`, which is actually renamed to `neighboring_null_rounds_query` to distinguish which exactly neighbors are queried.

> The logic of `fetch_neighboring_existing_blocks`, `process_next_batch`, and `find_first_valid_block` needs to be extended.

After that, the suggested text was edited manually. Finally, Claude was prompted with:

> This is my version of the section "Consistency Support" based on yours. Carefully read through it and check that I didn't miss any important part of the implementation, that all meaningful changes are reflected there.

## Final Code Adjustments

The final adjustments of the code resulting from [this commit](https://github.com/blockscout/blockscout/pull/11633/commits/d844dfdeaa86fc12c8f33ab291a5d977e6dcdffc) were done in two separate Cursor chat sessions with the following prompts:

### Chat to modify `null_round_height.ex`

The lines from 127 to 131 of `apps/explorer/lib/explorer/chain/null_round_height.ex` were attached to the chat.

> Looks like this functionality can be replaced by the call of `fetch_neighboring_null_rounds`. Please confirm.

### Chat to modify `general.ex`

> `repo = if from_api, do: Repo.replica(), else: Repo` is met in the module `@general.ex` two times. At the same time, there is a function `Explorer.Chain.select_repo` in `@chain.ex`.
