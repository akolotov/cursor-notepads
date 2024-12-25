## Problem

The module `Explorer.Chain.Arbitrum.Reader.API` (in `apps/explorer/lib/explorer/chain/arbitrum/reader/api.ex`) is becoming too large. It contains more than 700 lines of code, which makes it difficult to maintain (see `apps/explorer/lib/explorer/chain/arbitrum/reader/README.md`).

## Proposed Solution

Divide the module into three new modules:
- `Explorer.Chain.Arbitrum.Reader.API.Messages` in `apps/explorer/lib/explorer/chain/arbitrum/reader/api/messages.ex`; it will contain all the public and private functions that are responsible for querying DB data related to the cross-chain Arbitrum messages.
- `Explorer.Chain.Arbitrum.Reader.API.Settlement` in `apps/explorer/lib/explorer/chain/arbitrum/reader/api/settlement.ex`; it will contain all the public and private functions that are responsible for querying DB data related to the batches, DA blob data, rollup blocks and transactions included in batches, block confirmations on the parent chain and so on.
- `Explorer.Chain.Arbitrum.Reader.API.General` in `apps/explorer/lib/explorer/chain/arbitrum/reader/api/general.ex`; it will include `transaction_to_logs_by_topic0`.

## High-level Scope of Changes

1. After creating modules as described in "Proposed Solution", the following modules need to be refactored since they use functionality from `Explorer.Chain.Arbitrum.Reader.API`:
- apps/block_scout_web/lib/block_scout_web/controllers/api/v2/arbitrum_controller.ex
- apps/block_scout_web/lib/block_scout_web/controllers/api/v2/block_controller.ex
- apps/block_scout_web/lib/block_scout_web/controllers/api/v2/transaction_controller.ex
- apps/block_scout_web/lib/block_scout_web/views/api/v2/arbitrum_view.ex
- apps/explorer/lib/explorer/arbitrum/claim_rollup_message.ex

2. The documentation in the following files needs to be adjusted as well since they refer to either the file `apps/explorer/lib/explorer/chain/arbitrum/reader/api.ex` or the module `Explorer.Chain.Arbitrum.Reader.API`:
- apps/block_scout_web/lib/guides/arbitrum.md
- apps/explorer/lib/explorer/chain/arbitrum/reader/common.ex
- apps/explorer/lib/explorer/chain/arbitrum/reader/README.md

### Consistency Preserving

1. Any private or public function must have the specification and documentation comment.
