## Solution text

[solution-text.md](solution-text.md) describes the changes made under https://github.com/blockscout/blockscout/pull/11633/commits/5923b2bdccfd5383d71ceabbb3e29180353ecabf.

Although no Cursor's notepad was used to prepare these code changes, a draft for the items 1, 2, and 3 of the section "Consistency Support" in the solution text was prepared with Claude (OpenAI's model 01 was not able to follow the instructions of such a prompt adequately).

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
