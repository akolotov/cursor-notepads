## Problem

In Arbitrum, the same AnyTrust data blob can be associated with different batches when these batches consist only of deposits from the parent chain. These deposits rely on data already present in the parent chain, and the batches merely advance the `delayedMessages` count (via the `addSequencerL2BatchFromOrigin` event) without introducing unique rollup-specific data. Consequently, the data blob hash remains identical across such batches, as it lacks new or time-dependent information. 

The database schema to support this Arbitrum behavior was extended in [PR #11485](https://github.com/blockscout/blockscout/pull/11485). However, changes to Blockscout’s API endpoint `/api/v2/arbitrum/batches/da/:data_hash` to accommodate this behavior—enabling the retrieval of multiple batches associated with a single data hash - were postponed and will be addressed in a separate PR.

## Proposed solution

The handler of the corresponding API endpoint will be changed to support two ways to be called:

`/api/v2/arbitrum/batches/da/anytrust/{data_hash}` - to gets most recent batch that corresponds to given data hash (default behavior)
`/api/v2/arbitrum/batches/da/anytrust/{data_hash}?type=all` - gets all batches that  corresponds to given data hash

The reason to not change the behaviour of the existing route `/api/v2/arbitrum/batches/da/anytrust/{data_hash}` is to not break the current functionality in Blockscout UI which assumes currently that this API endpoint returns the information for one batch similar to the endpoint `/api/v2/arbitrum/batches/:batch_number`. 

Calling the endpoint with `type=all` will return a list of batches with a limited amount of details, similar to how they are returned by `/api/v2/arbitrum/batches`.

## High level scope of changes

### Changes in the reader functionality (`apps/explorer/lib/explorer/chain/arbitrum/reader/api.ex`)

1. The query from `Explorer.Chain.Arbitrum.Reader.API.get_da_record_by_data_key_old_schema` is moved into a separate private function `build_da_records_by_data_key_query` since it could be reused. No other changes in `get_da_record_by_data_key_old_schema` since the situation when several batches could be referred by the same DA data blob is not supported in the old schema.

2. A private function `build_batch_numbers_by_data_key_query` to return a query that requests all the batch numbers for the given DA blob hash from `BatchToDaBlob` is created. The batches are sorted by numbers from highest to lowest.

3. The function `Explorer.Chain.Arbitrum.Reader.API.get_da_record_by_data_key` itself is not changed since it will continue to serve to get the batch number and the DA record corresponding to this batch. However, the function `Explorer.Chain.Arbitrum.Reader.API.get_da_record_by_data_key_new_schema` is changed to:
  - Get the DA blob description by executing the query returned by the function `build_da_records_by_data_key_query`
  - Extend the query returned by the function `build_batch_numbers_by_data_key_query` to return only the first element (the highest batch number)
  - Return the received batch number and DA blob descriptor from `get_da_record_by_data_key_new_schema`

4. The new public function `Explorer.Chain.Arbitrum.Reader.API.get_all_da_records_by_data_key` is created. It executes the queries from `build_da_records_by_data_key_query` and `build_batch_numbers_by_data_key_query` to receive the DA blob description and numbers of all batches associated with this DA blob. The function returns the list of batch numbers and the DA blob description. If execution of any of the queries returns `nil`, `get_all_da_records_by_data_key` must return `:error`. If `MigrationStatuses.get_arbitrum_da_records_normalization_finished()` returns `false`, the function must call `get_da_record_by_data_key` to get the DA blob description and batch number. In this case, the function will return a single-element list containing that batch number. If `get_da_record_by_data_key_new_schema` returns `:error`, `get_all_da_records_by_data_key` must return `:error` as well.

5. The function `Explorer.Chain.Arbitrum.Reader.API.batches` is extended to accept a list of batch numbers via the `batch_numbers` option in the keyword list. When this option is provided and its value is not `nil` or the empty list, the function returns the batches with the specified numbers. The same pagination operations that are applicable to the batches list received without the `batch_numbers` option will still apply.

### Changes in the controller (`apps/block_scout_web/lib/block_scout_web/controllers/api/v2/arbitrum_controller.ex`)

1. The function `BlockScoutWeb.API.V2.ArbitrumController.batches` must be extended to allow accept the list of the batches in the `batch_numbers` element of the `params` map. If the list exists it must be passed to the function `Explorer.Chain.Arbitrum.Reader.API.batches` as an option in the keyword list.

2. The function `BlockScoutWeb.API.V2.ArbitrumController.batch_by_data_availability_info` (it clause matching with the case when batch for AnyTrust DA blob is requested) calls one of the following new functions depending on the `type` element of the `params` map (there could be no such element in the map): `one_batch_by_data_availability_info` or `all_batches_by_data_availability_info`. Both function accepts the DA blob hash and the `params` map as well to forward it to the next function in the callflow if it is necessary.

3. The private function `one_batch_by_data_availability_info` receives the batch number by `Explorer.Chain.Arbitrum.Reader.API.get_da_record_by_data_key` and passes it to `BlockScoutWeb.API.V2.ArbitrumController.batch` in the element `batch_number` of the `params` map.

4. The private function `all_batches_by_data_availability_info` receives list of the batch numbers by calling `Explorer.Chain.Arbitrum.Reader.API.get_all_da_records_by_data_key` and passes it to `BlockScoutWeb.API.V2.ArbitrumController.batches` in the element `batch_numbers` of the `params` map.

### Changes in the view (`apps/block_scout_web/lib/block_scout_web/views/api/v2/arbitrum_view.ex`)

No changes are needed at this level since either `BlockScoutWeb.API.V2.ArbitrumController.batch` or `BlockScoutWeb.API.V2.ArbitrumController.batches` is called as the final step of the call flow initiated by `BlockScoutWeb.API.V2.ArbitrumController.batch_by_data_availability_info`. The final step of both `batch` and `batches` is to render JSONs: "arbitrum_batch.json" or "arbitrum_batches.json".

### Consistency preserving

1. If an existing private or public function is changed, its documentation comment and specification must be updated as well.

2. If a new private or public function is added, both the documentation comment and the specification must be always added.