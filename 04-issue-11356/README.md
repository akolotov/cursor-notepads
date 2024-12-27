## GitHub changes

The following commit was prepared with the help of the notepad.

- https://github.com/blockscout/blockscout/pull/11485/commits/7aba4adf4610aa43ba4492a5c6b8219213dee5c2
- https://github.com/blockscout/blockscout/pull/11485/commits/887dfeae4875b77526fef67add8ed94af156eac7
- https://github.com/blockscout/blockscout/pull/11485/commits/38f08fa085c4a2fcee1288d472a35422c9378608
- https://github.com/blockscout/blockscout/pull/11485/commits/7c1aeb1ab41d923a66c5a075235cd0fdde34998f

## Notes

**The Composer mode:** agent

The dialog started with:

> Take a look at the scections "New DB table and the corresponding Ecto schema", "Data migration implementation", "Changes in batches indexing process" and "Support of new DB schema in API" within "High level scope of changes" and check if you can access to source code of the files referred there?

The consequent prompts:

> Great! Let's start from the step 1 in the "New DB table and the corresponding Ecto schema". The content for the new migration script must be added to @20241217155103_add_data_blobs_to_batches_table.exs .

> 1. `data_type` must not be added.
> 2. rename `data_key` into `data_blob_id`. It must be the primary key.

> 1. I was wrong. `batch_number` must be the primary key.
> 2. Do you think that index should be created for `batch_number` even if it is the primary key?

> Let's move to the step 2 of "New DB table and the corresponding Ecto schema". The new schema must be placed into @batch_to_da_blob.ex

> 1. What is `define_field: false` for in `belongs_to` constructions?
> 2. Is `field(:data_blob_id, Hash.Full)` required if it is defined in `belongs_to(:da_record, ...`?

> Let's move to the step 3 under "New DB table and the corresponding Ecto schema". The file for the new runner is @batch_to_da_blobs.ex

> Let's move to the step 4 of "New DB table and the corresponding Ecto schema"

At this point the changes were committed: https://github.com/blockscout/blockscout/pull/11485/commits/7aba4adf4610aa43ba4492a5c6b8219213dee5c2

> Let's implement both steps from "Data migration implementation"

> Rename 
> 1.  the new module  to `ArbitrumDaRecordsDenormalization`
> 2. the new file to `arbitrum_da_records_denormalization.ex`
> 3. the key in `BackgroundMigrations` to `arbitrum_da_records_denormalization_finished`

> I was wrong. This migration is not denormalization definitely. Is it correct to call it normalization instead? What do you think?

> Please explain me what exactly the query in `Explorer.Migrator.ArbitrumDaRecordsNormalization.unprocessed_data_query` does

> Write a short comment for `unprocessed_data_query` that covers the idea of the query.

Before the next prompt additional two items was added in the notepad.

> Perform the steps 3 and 4 from the "Data migration implementation"

At this point the changes were committed: https://github.com/blockscout/blockscout/pull/11485/commits/887dfeae4875b77526fef67add8ed94af156eac7

> Implement the step 1 from "Changes in batches indexing process"

It was discovered that the composer struggle with the step description. The corresponding step in the notepad was re-written, some part of this step was moved to the step 2 of the same section.

> I have updated that step 1 of "Changes in batches indexing process" as following:
>
> Update `Indexer.Fetcher.Arbitrum.DA.Anytrust.prepare_for_import` (`apps/indexer/lib/indexer/fetcher/arbitrum/da/anytrust.ex`) to extend the current returning tuple with the list of of records for `arbitrum_batches_to_da_blobs` (`Arbitrum.BatchToDaBlob.to_import()`), so the returning tuple will contain the list of `Arbitrum.DaMultiPurposeRecord.to_import()`, the list of `Arbitrum.BatchToDaBlob.to_import()` and `MapSet.t()`. The elements of `Arbitrum.DaMultiPurposeRecord.to_import()` must still keep `batch_number` but it must be `nil` since this field is not deleted yet from `arbitrum_da_multi_purpose`. The module `Indexer.Fetcher.Arbitrum.DA.Celestia` (`apps/indexer/lib/indexer/fetcher/arbitrum/da/celestia.ex`) must be adapted correspondingly.

> The first parameter (which works actually as an accumulator) of both `Indexer.Fetcher.Arbitrum.DA.Anytrust.prepare_for_import` and `Indexer.Fetcher.Arbitrum.DA.Celestia.prepare_for_import` must assume that it is also a tuple which consists of two lists. The first list will be responsible to keep data of `Arbitrum.DaMultiPurposeRecord.to_import()` the second list is responsible to keep data of `Arbitrum.BatchToDaBlob.to_import()`.

> Don't specify `inserted_at` and `updated_at` in `batch_to_blob_record` since they will be inserted automatically during the import operation.

The previous prompts demonstrate that the composer expirienced difficutlties with the proper code generation. That's why lots of adjustments was done manually. The next two prompts were intended to check that the description in the notepad still corresponds to the code that was produced manually. 

> Verify that the current implementation of `Indexer.Fetcher.Arbitrum.DA.Anytrust.prepare_for_import` in @anytrust.ex  inline with requirements defined the step 1 from "Changes in batches indexing process" 

> Verify that the current implementation of `Indexer.Fetcher.Arbitrum.DA.Celestia.prepare_for_import` in @celestia.ex inline with requirements defined the step 2 from "Changes in batches indexing process"

> Implement the step 3 from "Changes in batches indexing process"

> Implement the step 4 from "Changes in batches indexing process"

> No, the step 4 is for updates in  @new_batches.ex 

> There is no any reason to return `da_records` and  `batch_to_blobs` in a tuple from `handle_batches_from_logs`. It makes the further handling of this tuple to pass to `Chain.import` complicated.

At this point the changes were committed: [https://github.com/blockscout/blockscout/pull/11485/commits/887dfeae4875b77526fef67add8ed94af156eac7](https://github.com/blockscout/blockscout/pull/11485/commits/38f08fa085c4a2fcee1288d472a35422c9378608)

Due to interruption the compose lost the context and a new composer section (mode: agent) was created:

> Take a look at @issue-11356. Do you have access to the files mentioned in the section "Support of new DB schema in API" of "High level scope of changes"?

> Implement the changes for "Explorer.Chain.Arbitrum.Reader.get_da_info_by_batch_number" described in "Support of new DB schema in API".
> 
> The following files could help you understand the corresponding tables schemas:
> - @da_multi_purpose_record.ex 
> - @batch_to_da_blob.ex 
