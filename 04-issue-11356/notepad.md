## Problem

It was found that a specific combination of configuration parameters in Blockscout, when indexing an Orbit chain using AnyTrust as DA solution, could not allow indexing batches if the AnyTrust committee uses the same data blobs for different batches. The batch information import to DB fails when several batches are inserted in the DB at once since uniqueness of the data blob hash is required.

Another problem occurs when an AnyTrust blob `D1` association with the Arbitrum batch `B1` is inserted in `arbitrum_da_multi_purpose` by one iteration of the batch indexer. When another iteration discovers another batch `B2` with the same data blob `D1`, it will overwrite the previous record, so the linkage of `B1` and `D1` will be lost.

## Proposed solution

The idea of the fix for both these issues is to modify the DB schema so that one data blob could be associated with more than one batch. In order to avoid record duplication for the same data blob but for different batches in the table `arbitrum_da_multi_purpose`, it is suggested to introduce another table `arbitrum_batches_to_da_blobs` which will contain just two columns: the batch ID and the identifier of the record from `arbitrum_da_multi_purpose`.

An alternative solution to add a column into the table `arbitrum_l1_batches` to keep the linkage between the batch and the data blob is considered undesirable due to the following facts:

- The database size would be impacted by an extra column in the table `arbitrum_l1_batches` keeping nulls for Blockscout instances like ArbitrumOne, which has 770k records in this table. This chain has no records in `arbitrum_da_multi_purpose` since it uses eip4844 blobs which are not reflected in this table.
- Since two-way lookups are required ("find a data blob by batch" and "find a batch by data blob"), a new index for the `arbitrum_l1_batches` table must be introduced. Migration actions "add a column" and "introduce new index" on Blockscout instances like ArbitrumOne could take significant time.

## High level scope of changes

### New DB table and the corresponding Ecto schema

1. Add a new migration script that creates a new table. Similar to `Explorer.Repo.Arbitrum.Migrations.AddDaInfo` in `apps/explorer/priv/arbitrum/migrations/20240527212653_add_da_info.exs`.
2. Add a new schema under `Explorer.Chain.Arbitrum` (within the directory `apps/explorer/lib/explorer/chain/arbitrum`) that models records for the new table. Each record builds links between one record in `Explorer.Chain.Arbitrum.L1Batch` (`apps/explorer/lib/explorer/chain/arbitrum/l1_batch.ex`) and one record in `Explorer.Chain.Arbitrum.DaMultiPurposeRecord` (`apps/explorer/lib/explorer/chain/arbitrum/da_multi_purpose_record.ex`).
3. Add a new runner under `Explorer.Chain.Import.Runner.Arbitrum` (within the directory `apps/explorer/lib/explorer/chain/import/runner/arbitrum`) which is responsible for bulk imports of the records defined in the previous step. An existing runner `Explorer.Chain.Import.Runner.Arbitrum.DaMultiPurposeRecords` (`apps/explorer/lib/explorer/chain/import/runner/arbitrum/da_multi_purpose_records.ex`) can be used as an example.
4. Reflect this new runner within `Explorer.Chain.Import.Stage.ChainTypeSpecific` (`apps/explorer/lib/explorer/chain/import/stage/chain_type_specific.ex`).

### Data migration implementation

1. Define a new key in `Explorer.Chain.Cache.BackgroundMigrations` (`apps/explorer/lib/explorer/chain/cache/background_migrations.ex`) to reflect the migration status when a migrator task (described i[...]
2. Add a migration sub-module (`Explorer.Migrator.AddressCurrentTokenBalanceTokenType` in `apps/explorer/lib/explorer/migrator/address_current_token_balance_token_type.ex` could be used as an example)[...]
    - find records in `arbitrum_da_multi_purpose` which have no corresponding records in `arbitrum_batches_to_da_blobs`.
    - take `data_key` and `batch_number` from `arbitrum_da_multi_purpose` to insert a record into `arbitrum_batches_to_da_blobs`.
    - define `update_cache` to set the corresponding key in `Explorer.Chain.Cache.BackgroundMigrations` to `true`.
3. Update the script `config/runtime.exs` to enable the new migration process:
    - For the module `Explorer.Migrator.ArbitrumDaRecordsNormalization` under the application `:explorer`, set the config parameter `enabled` to depend on the current chain type being `:arbitrum`.
    - Parse two environment variables to apply to the config parameters `concurrency` and `batch_size`, which are used in the module `Explorer.Migrator.ArbitrumDaRecordsNormalization` to build a batch of data for migration in one run of the migrator.
4. Update `Explorer.Application.configurable_children` in `apps/explorer/lib/explorer/application.ex` to add the new migration process to the list of children processes. The migration process should be enabled only for the `:indexer` app using the function `configure_mode_dependent_process`.

### Changes in batches indexing process

1. Update `Indexer.Fetcher.Arbitrum.DA.Anytrust.prepare_for_import` (`apps/indexer/lib/indexer/fetcher/arbitrum/da/anytrust.ex`):
    - The first parameter (which works actually as an accumulator) of the function must be a tuple which consists of two lists. The first list will be responsible to keep data of `Arbitrum.DaMultiPurposeRecord.to_import()` the second list is responsible to keep data of `Arbitrum.BatchToDaBlob.to_import()`.
    - The returning tuple must be extended with the list of of records for `arbitrum_batches_to_da_blobs` (`Arbitrum.BatchToDaBlob.to_import()`), so the returning tuple instead of one list will contain another tuple with the list of `Arbitrum.DaMultiPurposeRecord.to_import()` and the list of `Arbitrum.BatchToDaBlob.to_import()`.
    - The elements of `Arbitrum.DaMultiPurposeRecord.to_import()` must still keep `batch_number` but it must be `nil` since this field is not deleted yet from `arbitrum_da_multi_purpose`.
2. The function `Indexer.Fetcher.Arbitrum.DA.Celestia..prepare_for_import` of the module `Indexer.Fetcher.Arbitrum.DA.Celestia` (`apps/indexer/lib/indexer/fetcher/arbitrum/da/celestia.ex`) must be adapted similarly to the function `Indexer.Fetcher.Arbitrum.DA.Anytrust.prepare_for_import`.
3. Update`Indexer.Fetcher.Arbitrum.DA.Common.prepare_for_import` (`apps/indexer/lib/indexer/fetcher/arbitrum/da/common.ex`) to be able to work with new`Indexer.Fetcher.Arbitrum.DA.Anytrust.prepare_for_import`and`Indexer.Fetcher.Arbitrum.DA.Celestia.prepare_for_import`.
4. Update `Indexer.Fetcher.Arbitrum.Workers.NewBatches.handle_batches_from_logs` (`apps/indexer/lib/indexer/fetcher/arbitrum/workers/new_batches.ex`) to be able to work with new `Indexer.Fetcher.Arbitrum.DA.Common.prepare_for_import` and then update `Indexer.Fetcher.Arbitrum.Workers.NewBatches.do_discover` to use proper data structure for import.

### Support of new DB schema in API

In order to allow the new database structure to co-exist with the existing structure for the operation to receive the indexed data through API before the migration to the new structure is performed completely, it is necessary to update`apps/explorer/lib/explorer/chain/arbitrum/reader.ex`:

1. `Explorer.Chain.Arbitrum.Reader.get_da_info_by_batch_number` is used to reflect DA data in the batch view (see `BlockScoutWeb.API.V2.ArbitrumView.generate_anytrust_certificate` and `BlockScoutWeb.API.V2.ArbitrumView.generate_celestia_da_info` in `apps/block_scout_web/lib/block_scout_web/views/api/v2/arbitrum_view.ex`), so the following logic needs to be implemented:
    - if the corresponding key in the `Explorer.Chain.Cache.BackgroundMigrations` is set to `true`, get the data blob from `arbitrum_batches_to_da_blobs`.
    - Otherwise, if the query to `Explorer.Chain.Arbitrum.DaMultiPurposeRecord` for the specific batch number returns, use `arbitrum_batches_to_da_blobs` as a fallback to check if the data blob for the batch is there.
2. `Explorer.Chain.Arbitrum.Reader.get_da_record_by_data_key` is used to identify which batch needs to be displayed when the DA-related data is provided through API (see `BlockScoutWeb.API.V2.ArbitrumController.batch_by_data_availability_info` in `apps/block_scout_web/lib/block_scout_web/controllers/api/v2/arbitrum_controller.ex`), so the following logic needs to be implemented:
    - get the data blob information by the same query which exists already (requests data from `Explorer.Chain.Arbitrum.DaMultiPurposeRecord`)
    - if the corresponding key in the `Explorer.Chain.Cache.BackgroundMigrations` is set to `true`, get the batch number from `arbitrum_batches_to_da_blobs`.
    - Otherwise, use the batch number from the result returned by the query requesting data from `Explorer.Chain.Arbitrum.DaMultiPurposeRecord`.

### DB integrity recovery

The implementation below is needed to recover the database integrity for the instances where the second problem described above occurred.

[NEEDS BREAKDOWN] Create a new process in the fetcher by using BackgroundTask to check if the batch exists in the database and its data type is AnyTrust whereas there is no corresponding data record. If so, re-read the L1 transactions that submitted the batch and either recover the linkage between the batch and data blob (if the blob exists in the DB) or insert missing blob data.  This process will start only when the corresponding key in the`Explorer.Chain.Cache.BackgroundMigrations`is set to`true`.

### After migration cleanup

[NEEDS ADDITIONAL RESEARCH] Investigate how to implement removal of `batch_number` the field from the table `arbitrum_da_multi_purpose` when migration is completed.
