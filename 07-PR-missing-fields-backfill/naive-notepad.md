This functionality describes a module `Indexer.Fetcher.Arbitrum.Workers.Backfill` which must be in the file `apps/indexer/lib/indexer/fetcher/arbitrum/workers/backfill.ex`

1. There should be a private function `fetch_receipts` to get the receipts by `fetch_by_block_numbers` from @receipts.ex.

This is the function definition:
```elixir
EthereumJSONRPC.Receipts.fetch_by_block_numbers(
  block_numbers,
  json_rpc_named_arguments
)
```

The receipts must be requested in batches by chunking the list block numbers. Similar functionlity is implemented in `Indexer.Block.Fetcher.Receipts.fetch` of @receipts.ex. But making concurrent requests is not needed to not overload RPC node. The function could return receipts only since information about logs is not required. 

2. There should be a public function `discover_blocks`. Its functionality is described below in the step #3-6

3. To get all number of blocks with missinging fields by `blocks_with_missing_fields` from @db.ex. There is no need to query transactions with missing fields from the database since it is assumed that operations to update the block information and transactions information are atomic as per the nature of @fetcher.ex. Note, that this requirement to have the atomic update for blocks and transactions must be preserved in the new implemented functionality (below)

The function definition:
```elixir
Indexer.Fetcher.Arbitrum.Utils.Db.blocks_with_missing_fields(
  start_block_number,
  end_block_number
)
```

4. Getting all these blocks by `fetch_blocks_by_numbers` from @ethereum_jsonrpc.ex. No need to get the transactions bodies that is why `with_transactions` is `false`. 

The function definition:
```elixir
EthereumJSONRPC.fetch_blocks_by_numbers(
  block_numbers,
  json_rpc_named_arguments,
  with_transactions? \\ true
)
```

The blocks data be received here in batches by chunking the list of block numbers.

5. Getting all the receipts for these blocks by the function from the step #1.

6. As soon as the blocks and receipt are received the function must update the data base by making the DB update atomic through `Ecto.Multi` functionality.

Here is the example how Multi.Ecto is used already in the Blockscout code:
```elixir
    multi =
      inserts
      |> Map.get(:insert_scroll_batch_bundles, [])
      |> Enum.reduce(Multi.new(), fn bundle, multi_acc ->
        start_batch_number = start_by_final_batch_number[bundle.final_batch_number]

        Multi.update_all(
          multi_acc,
          bundle.id,
          from(b in Batch, where: b.number >= ^start_batch_number and b.number <= ^bundle.final_batch_number),
          set: [bundle_id: bundle.id]
        )
      end)

    Repo.transaction(multi)
```
The example was taken from `Indexer.Fetcher.Scroll.Batch.import_items` (@batch.ex)

For every block received in the step #4, the fields `send_count`, `send_root` and `l1_block_number` must be updated in `Explorer.Chain.Block` (@block.ex).

For every receipt received on the step #5, the field `gas_used_for_l1` must be updated in `Explorer.Chain.Transaction` (@transaction.ex).
