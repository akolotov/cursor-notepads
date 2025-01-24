## GitHub changes

The following commits were prepared with the help of the notepad.

- https://github.com/blockscout/blockscout/pull/11731/commits/73a0f257e7bad1ef06212b276884fcad0c514568
- https://github.com/blockscout/blockscout/pull/11731/commits/e877208e23f170422f10a2e8033976e88c890b6d
- https://github.com/blockscout/blockscout/pull/11731/commits/154d06731c0e8bb79db7d4f3e74ccfc2b0a0ff36

## Notes

### Phase 1

**The Composer mode:** agent
**The Compose model:** `claude-3-5-sonnet-20241022`

The dialog started with the prompt:

> Take a look at proposed changes within the notepad, which additional information do you need before able to implement the changes? Don't modify any files yet, just answer the question.

The next prompt:

> The specification of the `SequencerInbox` contract's function are avalable there @https://github.com/OffchainLabs/nitro-contracts/blob/94999b3e2d3b4b7f8e771cc458b9eb229620dd8f/src/bridge/ISequencerInbox.sol . 
>
> The implementation of `EthereumJSONRPC.Arbitrum.Constants.Contracts` is @contracts.ex 
> 
> The implementation of `Indexer.Fetcher.Arbitrum.Workers.NewBatches` is @new_batches.ex 
> 
> Double check and inform if you need additional information. Don't implement any changes, just confirm if you need additional information.

On this stage the composer informed that it is ready to proceed with the code generation, but it was un-clear
how exactly it will produce non-obvious part of the code. That is why, the following prompt was done:

> Before you change anything answer on the following question:
> - what the structure of `DelayProof`?
> - what way do you use to get selectors for `addSequencerL2BatchFromOriginDelayProof` and `addSequencerL2BatchDelayProof`

So, the composer identifed the gaps in his assumptions and asked for additional context. It was provided by the prompt (a sreenshot of the Read contract page on Blockscout UI was provided):

> - the structure of `DelayProof` is declared here @https://github.com/OffchainLabs/nitro-contracts/blob/94999b3e2d3b4b7f8e771cc458b9eb229620dd8f/src/bridge/DelayBufferTypes.sol 
> - as you will see it uses `Messages.Message` type which is declared here: @https://github.com/OffchainLabs/nitro-contracts/blob/94999b3e2d3b4b7f8e771cc458b9eb229620dd8f/src/bridge/Messages.sol 
> 
> The descriptors of the functions can be found on the screenshot which I made on the `SequencerInbox` contract page on Blockscout UI.

Looks like, the composer now had everything:

> Implement the changes

The produced changes were committed as [73a0f25](https://github.com/blockscout/blockscout/pull/11731/commits/73a0f257e7bad1ef06212b276884fcad0c514568)

### Phase 2

It was found that ABIs for the functions were generated on the previous phase incorrectly. The same composer session was used to prompt:

> New ABIs are constructed in @contracts.ex incorrectly. Take a look at @proven.ex for reference.

It still struggled, that is why was navigated explicetly. 

> But you missed the fact that `bytes32` must be constructed in a different way.

The produced changes were committed as [e877208](https://github.com/blockscout/blockscout/pull/11731/commits/e877208e23f170422f10a2e8033976e88c890b6d)

### Phase 3

Small documentation improvements were done in the same composer session:

> Consider to improve the documentation for the functions:
> - add_sequencer_l2_batch_from_blobs_selector_with_abi
> - add_sequencer_l2_batch_from_blobs_delay_proof_selector_with_abi
> - add_sequencer_l2_batch_from_origin_delay_proof_selector_with_abi
> - add_sequencer_l2_batch_delay_proof_selector_with_abi
> 
> by specifying full set of parameters of the solidity function like it is done for `add_sequencer_l2_batch_from_origin_6f12b0c9_selector_with_abi` and `add_sequencer_l2_batch_from_origin_8f111f3c_selector_with_abi`

The final changes were committed as [154d067](https://github.com/blockscout/blockscout/pull/11731/commits/154d06731c0e8bb79db7d4f3e74ccfc2b0a0ff36)
