## Phase #1

The way to implement the changes described in the notepad (named `query-optimization`) was not linear. The main part of the changes was made in the Cursor Chat in Agent mode. However, not all prompts were successful on the first attempt. “Restore Checkpoint” was used, the prompt was adjusted, and the changes were re-generated. Some changes were made in short separate chats to avoid muddling the context of the main chat.

----

> Take a look at the section "Proposed Solution" in the notepad and analyze if the solution breaks the current logic implemented in `Indexer.Fetcher.Arbitrum.Utils.Db.Settlement.l1_blocks_to_expect_rollup_blocks_confirmation` through the call of `Explorer.Chain.Arbitrum.Reader.Indexer.Settlement.l1_blocks_of_confirmations_bounding_first_unconfirmed_rollup_blocks_gap`.
>
> If the solution equal to the logic implemented to this moment consider to check the section "High Level Scope of Changes" to ensure that the described changes there completely implement logic suggested in "Proposed Solution".

----

> Let's implement the step 1 of the section "Extending Reader Functionality"

----

> Implement the step 2 of the section "Extending Reader Functionality"

----

> The function does not need to return the rollup block - only the parent chain block is required.

----

> Implement the step 3 of the section "Extending Reader Functionality"

----

> Implement the changes described in the section "Altering Wrapper Function"

----

> Explain why in the case of "All blocks are confirmed, get the L1 block of the lowest confirmed block" usage of `Reader.closest_confirmed_block_below` is correct choice?

----

> For me, it looks incorrectly to assume that the lowes confirmed block is always 1. Take a look at @first-block-confirmations as an example why this assumption is not correct. From my perspecive, in the scenario it is necessary to receive the earliest block in the table `arbitrum_batch_l2_blocks` to get L1 block of its confirmation then. Do you agree? If yes, what could the best ways be to implement this?

----

> In fact `earliest_rollup_block` return the earliest rollup block discovered by the batch fetcher. It could be different from the actual earliest block of the chain and could be different from the earliest rollup block discovered by the block fetcher (due to async nature of fetchers work). Adjust the comment and probably the function name correspondingly.

----

> Does it make sense in the comments clarify what exactly each case mean?

----

> Will you be able to create new index as it is described in the `New Index Introduction` section? What will the steps be? Instruct with @update-db-schema.mdc . Make sure that you use proper chain type. The migration name is "add_unconfirmed_blocks_index".

----

> Pay attention on details described in "New Index Introduction":
> - the order for `block_number`
> - proper condition for the index
> - requirement for the flag `concurrently`.
> 
> @20240201125730_create_arbitrum_tables.exs contains the schema of the table `arbitrum_batch_l2_blocks`.
> @20240318154323_create_blob_transactions_index.exs contains an example how `concurrently` flag is used.

----

> Implement the changes described in the "New Index Usage" section. 

----

> Consider to change the function names `closest_confirmed_block_above` and `closest_confirmed_block_below` to something that reflect that they return L1 block numbers. 

----

> Analyze how it is effective to make ascending ordering in `l1_block_of_closest_confirmed_block_above` while the index is optimized for the descending ordering. Keep in mind that we need to have just one element as the result. 

----

> Consider to extend the comment of `l1_blocks_to_expect_rollup_blocks_confirmation` to reflect why group of consecutive queries is preferred to one comprehensive query.

----

The GitHub [commit](https://github.com/blockscout/blockscout/pull/11914/commits/438d80ba436e852e81e0fa067ef4bf1b6efde864) as at the end of the phase 1.

## Phase 2

After receiving this [comment](https://github.com/blockscout/blockscout/pull/11914#discussion_r1998299557) on the PR review, it was necessary to make a research what the new functionality mentioned in the comment is. The Cursor Chat in the 'Ask' mode (with the 'o3-mini-high' model) was used to raise the questions about the code:

----

> Take a look at @heavy_db_index_operation.ex and examples of implementations in @heavy_db_index_operation . Modules @background_migrations.ex, @migration_status.ex , @config.exs and @application.ex  are added for the context.
>
> 1. Explain me what is the idea behind the modules under `Explorer.Migrator.HeavyDbIndexOperation`
> 2. If I need to implement my own havy migration what I need to pay attention to before diciding if I need to implement my own module or not.
> 3. What are main parts must be implemented in my own module.

----

> What is the reason to have two separate callbacks `check_db_index_operation_progress()` and `db_index_operation_status()`. Does `check_db_index_operation_progress()` of one module differ from `check_db_index_operation_progress()` of another module? The same question for `db_index_operation_status`.

----

> Take a look at the query string. Why does it have `((1))` in after the table name in `ON` part?

----

At this moment, another Cursor Chat session in Agent mode was created to implement the changes (see “Phase 3”).

Later, when the changes composed in that chat were ready, this chat was continued, but in Agent mode (using the 'claude-3-5-sonnet-20241022' model) so the changes in README.md file were done automatically.

----

> Based on the discussed things create `apps/explorer/lib/explorer/migrator/heavy_db_index_operation/README.md` file that covers
> 1. What modules with heavy index operations are for
> 2. What is the difference between two `@operation_type` 
> 3. What is the difference between creation of the indexer through `@query_string` and `@table_columns`
> 4. In which cases `update_cache` is empty in which not.
> 5. What is the process to create new module
> 6. What must be updated in the modules @application.ex , @config.exs or @runtime.exs 
> 7. How to create a cache value to track the status of the index creation/deletion?
> 
> These are just areas which must be covered. The order of including the corresponding information into the file could be different.

----

> It is not true. Take a look at @create_arbitrum_batch_l2_blocks_unconfirmed_blocks_index.ex 

----

> What does "automatically implemented" mean?

----

> I am not sure that you are right. Take a look at @create_internal_transactions_block_number_desc_block_index_desc_index.ex and @create_arbitrum_batch_l2_blocks_unconfirmed_blocks_index.ex .
> 
> - table_name, operation_type and `index_name` are implemented even if `@table_name`, `@index_name` and `@operation_type` are defined. 
> 
> - `db_index_operation`, `check_db_index_operation_progress` and `db_index_operation_status` `restart_db_index_operation` and `running_other_heavy_migration_exists` are implemented as well although only use the functionality from `HeavyDbIndexOperationHelper` or `MigrationStatus` - but this functionality is invoked explicitely in the implementations.

----

> The difference between these two approaches must be defined more clearly.
> 
> The index operation should be added through `apps/explorer/config/config.exs` if it must be run on all instances.
> 
> Configuration through `config/runtime.exs` indeed allows to take into account any runtime-specific settings and specifically is used when the index operation depends on the chain type. See an example with `CreateArbitrumBatchL2BlocksUnconfirmedBlocksIndex`

----

> Take a look at @config.exs . In fact, the modules under `HeavyDbIndexOperation` are configured through `for` loop. Finally, the configuration is identical to what you described in README but the actual approach how it is implemented is different.

----

> 1 and 3 is not applicable as we can see from the existing implementation of index operation modules.
> 
> 4-6 is common sense so no needed.
> 
> Consider to add an item to highlight that it is necessary consider blockscout functionality which depends on the index existance (in case of index creation) and if it is necessary this functionality must handle properly the case when the index is not available yet. For example, if without index query could be terminated by timeout or could be too long, or could impact DB performance, the check of index creation process status can be added to these pieces of coded to behave differently - e.g. some functionality can be switched off at all until the index is available.

----

> Take a look at @heavy_db_index_operation.ex. Is my understanding of the state how GenServer switches between task is correct?
> 
> It starts from `:continue`, which cause `:initiate_index_operation`.
> 
> If the index operation was not completed, the next message is `:check_if_db_operation_need_to_be_started` and then `:check_db_index_operation_progress`.
> 
> When `db_index_operation` is called. The next message `:check_db_index_operation_progress` is scheduled in an interval configured by `Application.get_env(:explorer, Explorer.Migrator.HeavyDbIndexOperation)[:check_interval]` or 10 minutes. BTW `Application.get_env(:explorer, Explorer.Migrator.HeavyDbIndexOperation)[:check_interval]` by default is 10 minutes as well.
> 
> It means that in default configuration, the status of index operation will be updated in the cache as completed in 10 minutes even if the operation happens in 1-2 seconds (when the instance is new and DB is empty). This specific of the `HeavyDbIndexOperation` must be explicitely described in README.md file. 

----

After minor manual updates the new README.md file was [committed]([https://github.com/blockscout/blockscout/pull/11914/commits/2794bbfe58489f53078ddb52c7682a1b31bcdd29](https://github.com/blockscout/blockscout/pull/11914/commits/b03aae41f78152a537ee8ee06b18240d20a090d8)) in GitHub.

## Phase 3

The Cursor Chat in Agent mode with the model 'claude-3-5-sonnet-20241022'. The feature "Restore checkpoint" with re-phrasing the previous prompt version was used actively.

----

> Assuming that the production DB can contain millions of records in the `arbitrum_batch_l2_blocks` table, the migration suggested in @20250220230046_add_unconfirmed_blocks_index.exs can be considered as heavy. Therefore, suggest a new module `apps/explorer/lib/explorer/migrator/heavy_db_index_operation/create_addresses_verified_index.ex/create_arbitrum_batch_l2_blocks_unconfirmed_blocks_index.ex` implementing `Explorer.Migrator.HeavyDbIndexOperation` (@heavy_db_index_operation.ex ) behaiour for the new migration.
> 
> What changes in @application.ex and @config.exs  must be done as well?
> 
> Consider as a main example of existing implementation @create_addresses_verified_index.ex .
> 
> Added @helper.ex for the context as well.

----

> `Explorer.Migrator.HeavyDbIndexOperation.CreateArbitrumBatchL2BlocksUnconfirmedBlocksIndex` must not be listed in @config.exs since creation of the index is specific only for the chain type `:arbitrum`. 
> 
> Move the configuration of the migration process (if it is enabled or not) to @runtime.exs similar as the configuration for `Explorer.Migrator.ArbitrumDaRecordsNormalization` is done.

----

> Since it will be required to get the status of migration in the fetcher responsible for indexing confirmations of rollup blocks on the parent chain, it is necessary to introduce the new key in the cache @background_migrations.ex and to add updating the cache in @create_arbitrum_batch_l2_blocks_unconfirmed_blocks_index.ex . 
> 
> An example of similar functionality exists in @create_internal_transactions_block_number_desc_block_index_desc_index.ex .

----

> Update the module doc comment since the cache now keeps status not only for table migration operations but also of heavy index operations. Consider to remove the list of migrations since maintaining it while the list of operations is growing is quite difficult and developers will forget to do this.

----

> Switching to the new approach to create index assumes that the Blockscout server now becomes operational while the index is in progress. It means that the DB queries relying on existance of this index could be ineffective until the index creation is finished. In order to avoid unexpected timeouts it make sense to not run the corresponding queries if the index is not created yet.
> 
> To avoid running tasks discovering confirmations which produce the corresponding queries the following is suggested:
> 
> 1. Create a public function `unconfirmed_blocks_index_ready?` in @helper.ex  which will call `BackgroundMigrations.get_heavy_indexes_create_arbitrum_batch_l2_blocks_unconfirmed_blocks_index_finished()` 
> 
> 2. Create a public function `plan` in @tasks.ex . This function will accepts a function and `state`. If `unconfirmed_blocks_index_ready?` is false it logs a warning (use `log_warning` from @logging.ex ) and returns `{:ok, state}`. Otherwise it call the given function and pass `state` as the input parameter.
> 
> 3. Update 
> - `handle_info(:check_new_confirmations, ...`
> - `handle_info(:check_historical_confirmations, ...`
> of @tracking_batches_statuses.ex 
> They will call `:timer.tc` with `plan` of @tasks.ex where either `check_new` or `check_unprocessed` and `state` are parameters.

----

> With all the changes implemented it is necessary to updated @query-optimization .
> 
> "New Index Introduction" must mention that Heavy Index is created and why it is done rather than usual Ecto.Migration. Here is also important to add about new cache value creation.
> 
> "New Index Usage" must be extended with changes required in `Indexer.Fetcher.Arbitrum.Workers.Confirmations.Tasks`, what is the reason for them, how these changes affect `Indexer.Fetcher.Arbitrum.TrackingBatchesStatuses`
> 
> Suggest the draft of updates for the corresponding section of the notepad only. Output the draft as a markdown snippet.

----

> I adjusted the changes suggested for the section "New Index Usage". Check grammar and spelling.

----

The changes made in the chat session was committed as [2794bbf](https://github.com/blockscout/blockscout/pull/11914/commits/2794bbfe58489f53078ddb52c7682a1b31bcdd29)
