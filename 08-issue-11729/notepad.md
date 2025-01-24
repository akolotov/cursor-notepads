## Issue

On the Blockscout instance "https://arbitrum-sepolia.blockscout.com/" the batches indexing stuck with the following record in the logs:

```plaintext
GenServer Indexer.Fetcher.Arbitrum.TrackingBatchesStatuses terminating
** (CaseClauseError) no case clause matching: \"0x917cf8ac000000000000. . .eca8b3fa125c70d5c0e0\"
    (indexer 6.10.1) lib/indexer/fetcher/arbitrum/workers/new_batches.ex:1154: Indexer.Fetcher.Arbitrum.Workers.NewBatches.add_sequencer_l2_batch_from_origin_calldata_parse/1
    (indexer 6.10.1) lib/indexer/fetcher/arbitrum/workers/new_batches.ex:1091: anonymous fn/6 in Indexer.Fetcher.Arbitrum.Workers.NewBatches.execute_transaction_requests_parse_transactions_calldata/6
    (elixir 1.17.3) lib/enum.ex:2531: Enum.\"-reduce/3-lists^foldl/2-0-\"/3
    (indexer 6.10.1) lib/indexer/fetcher/arbitrum/workers/new_batches.ex:762: Indexer.Fetcher.Arbitrum.Workers.NewBatches.handle_batches_from_logs/6
    (indexer 6.10.1) lib/indexer/fetcher/arbitrum/workers/new_batches.ex:615: anonymous fn/7 in Indexer.Fetcher.Arbitrum.Workers.NewBatches.do_discover/8
    (elixir 1.17.3) lib/enum.ex:987: Enum.\"-each/2-lists^foreach/1-0-\"/2
    (indexer 6.10.1) lib/indexer/fetcher/arbitrum/utils/helper.ex:197: anonymous fn/7 in Indexer.Fetcher.Arbitrum.Utils.Helper.execute_for_block_range_in_chunks/5
    (elixir 1.17.3) lib/range.ex:538: Enumerable.Range.reduce/5
Last message: :check_new_batches
State: %{. . .}
```

## Analysis

Starting from last week, some batches of Arbitrum Sepolia have been submitted to the `SequencerInbox` contract using the method `addSequencerL2BatchFromBlobsDelayProof`.

In fact, there are a family of calls, including [`addSequencerL2BatchFromBlobsDelayProof`](https://github.com/OffchainLabs/nitro-contracts/blob/94999b3e2d3b4b7f8e771cc458b9eb229620dd8f/src/bridge/ISequencerInbox.sol#L239-L248), [`addSequencerL2BatchFromOriginDelayProof`](https://github.com/OffchainLabs/nitro-contracts/blob/94999b3e2d3b4b7f8e771cc458b9eb229620dd8f/src/bridge/ISequencerInbox.sol#L250-L261), and [`addSequencerL2BatchDelayProof`](https://github.com/OffchainLabs/nitro-contracts/blob/94999b3e2d3b4b7f8e771cc458b9eb229620dd8f/src/bridge/ISequencerInbox.sol#L263-L273), that were introduced by the `SequencerInbox` contract. However, their support was not added to the Blockscout backend responsible for indexing new batches.

## Proposed changes

1. The ABI of new functions of the `SequencerInbox` contract must be added to the module [`EthereumJSONRPC.Arbitrum.Constants.Contracts`](https://github.com/blockscout/blockscout/blob/b6d8f01a1d3edaf55f8f662a4acb110d33ac23b0/apps/ethereum_jsonrpc/lib/ethereum_jsonrpc/arbitrum/constants/contracts.ex). 
2. Parsing input parameters of the new functions must be supported in the function [`add_sequencer_l2_batch_from_origin_calldata_parse`](https://github.com/blockscout/blockscout/blob/4be733ce8c1620a7519c4b2f18ca74aea9ddd1d8/apps/indexer/lib/indexer/fetcher/arbitrum/workers/new_batches.ex#L1149-L1185) of the module `Indexer.Fetcher.Arbitrum.Workers.NewBatches`.
