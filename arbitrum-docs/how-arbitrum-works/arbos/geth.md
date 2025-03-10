---
author: dzgoldman
---

# Geth

Nitro makes minimal modifications to Geth in hopes of not violating its assumptions. This document will explore the relationship between Geth and ArbOS, which consists of a series of hooks, interface implementations, and strategic re-appropriations of Geth's basic types.

We store ArbOS's state at an address inside a Geth `statedb`. In doing so, ArbOS inherits the `statedb`'s statefulness and lifetime properties. For example, a transaction's direct state changes to ArbOS are discarded upon a revert.

**0xA4B05FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF**<br/>
The fictional account representing ArbOS

:::info

Please note any links on this page may be referencing old releases of Nitro or our fork of Geth. While we try to keep this up to date and most of this should be stable, please check against latest releases for [Nitro](https://github.com/OffchainLabs/nitro/releases) and [Geth](https://github.com/OffchainLabs/go-ethereum/releases) for most recent changes.

:::

## Hooks

Arbitrum uses various hooks to modify Geth's behavior when processing transactions. Each provides an opportunity for ArbOS to update its state and make decisions about the transaction during its lifetime. Transactions are applied using Geth's [`ApplyTransaction`][applytransaction_link] function.

Below is [`ApplyTransaction`][applytransaction_link]'s callgraph, with additional info on where the various Arbitrum-specific hooks are inserted. Click on any to go to their section. By default, these hooks do nothing so as to leave Geth's default behavior unchanged, but for chains configured with [`EnableArbOS`](#EnableArbOS) set to true, [`ReadyEVMForL2`](#ReadyEVMForL2) installs the alternative L2 hooks.

- `core.ApplyTransaction` ⮕ `core.applyTransaction` ⮕ `core.ApplyMessage`
  - `core.NewStateTransition`
    - [`ReadyEVMForL2`](#ReadyEVMForL2)
  - `core.TransitionDb`
    - [`StartTxHook`](#StartTxHook)
    - `core.transitionDbImpl`
      - if `IsArbitrum()` remove tip
      - [`GasChargingHook`](#GasChargingHook)
      - `evm.Call`
        - `core.vm.EVMInterpreter.Run`
          - [`PushCaller`](#PushCaller)
          - [`PopCaller`](#PopCaller)
      - `core.StateTransition.refundGas`
        - [`ForceRefundGas`](#ForceRefundGas)
        - [`NonrefundableGas`](#NonrefundableGas)
    - [`EndTxHook`](#EndTxHook)
  - added return parameter: `transactionResult`

What follows is an overview of each hook, in chronological order.

### [`ReadyEVMForL2`][readyevmforl2_link]{#ReadyEVMForL2}

A call to [`ReadyEVMForL2`][readyevmforl2_link] installs the other transaction-specific hooks into each Geth [`EVM`][evm_link] right before it performs a state transition. Without this call, the state transition will instead use the default [`DefaultTxProcessor`][defaulttxprocessor_link] and get exactly the same results as vanilla Geth. A [`TxProcessor`][txprocessor_link] object is what carries these hooks and the associated Arbitrum-specific state during the transaction's lifetime.

### [`StartTxHook`][starttxhook_link]{#StartTxHook}

The [`StartTxHook`][starttxhook_link] is called by Geth before a transaction starts executing. This allows ArbOS to handle two Arbitrum-specific transaction types.

If the transaction is `ArbitrumDepositTx`, ArbOS adds balance to the destination account. This is safe because the L1 bridge submits such a transaction only after collecting the same amount of funds on L1.

If the transaction is an `ArbitrumSubmitRetryableTx`, ArbOS creates a retryable based on the transaction's fields. If the transaction includes sufficient gas, ArbOS schedules a retry of the new retryable.

The hook returns `true` for both of these transaction types, signifying that the state transition is complete.

### [`GasChargingHook`][gascharginghook_link]{#GasChargingHook}

This fallible hook ensures the user has enough funds to pay their poster's L1 calldata costs. If not, the transaction is reverted and the [`EVM`][evm_link] does not start. In the common case that the user can pay, the amount paid for calldata is set aside for later reimbursement of the poster. All other fees go to the network account, as they represent the transaction's burden on validators and nodes more generally.

If the user attempts to purchase compute gas in excess of ArbOS's per-block gas limit, the difference is [set aside][difference_set_aside_link] and [refunded later][refunded_later_link] via [`ForceRefundGas`](#ForceRefundGas) so that only the gas limit is used. Note that the limit observed may not be the same as that seen [at the start of the block][that_seen_link] if ArbOS's larger gas pool falls below the [`MaxPerBlockGasLimit`][max_perblock_limit_link] while processing the block's previous transactions.

[difference_set_aside_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L407
[refunded_later_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/state_transition.go#L419
[that_seen_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/block_processor.go#L176
[max_perblock_limit_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/l2pricing/l2pricing.go#L86

### [`PushCaller`][pushcaller_link]{#PushCaller}

These hooks track the callers within the EVM callstack, pushing and popping as calls are made and complete. This provides [`ArbSys`](/build-decentralized-apps/precompiles/02-reference.mdx#arbsys) with info about the callstack, which it uses to implement the methods `WasMyCallersAddressAliased` and `MyCallersAddressWithoutAliasing`.

### [`L1BlockHash`][l1blockhash_link]

In Arbitrum, the BlockHash and Number operations return data that relies on underlying L1 blocks instead of L2 blocks, to accommodate the normal use-case of these opcodes, which often assume Ethereum-like time passes between different blocks. The L1BlockHash and L1BlockNumber hooks have the required data for these operations.

### [`ForceRefundGas`][forcerefundgas_link]{#ForceRefundGas}

This hook allows ArbOS to add additional refunds to the user's tx. This is currently only used to refund any compute gas purchased in excess of ArbOS's per-block gas limit during the [`GasChargingHook`](#GasChargingHook).

### [`NonrefundableGas`][nonrefundablegas_link]{#NonrefundableGas}

Because poster costs come at the expense of L1 aggregators and not the network more broadly, the amounts paid for L1 calldata should not be refunded. This hook provides Geth access to the equivalent amount of L2 gas the poster's cost equals, ensuring this amount is not reimbursed for network-incentivized behaviors like freeing storage slots.

### [`EndTxHook`][endtxhook_link]{#EndTxHook}

The [`EndTxHook`][endtxhook_link] is called after the [`EVM`][evm_link] has returned a transaction's result, allowing one last opportunity for ArbOS to intervene before the state transition is finalized. Final gas amounts are known at this point, enabling ArbOS to credit the network and poster each's share of the user's gas expenditures as well as adjust the pools. The hook returns from the [`TxProcessor`][txprocessor_link] a final time, in effect discarding its state as the system moves on to the next transaction where the hook's contents will be set afresh.

[applytransaction_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/state_processor.go#L152
[evm_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/vm/evm.go#L101
[defaulttxprocessor_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/vm/evm_arbitrum.go#L42
[txprocessor_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L38
[starttxhook_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L100
[readyevmforl2_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbstate/geth-hook.go#L47
[gascharginghook_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L354
[pushcaller_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L76
[popcaller_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L80
[forcerefundgas_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L425
[nonrefundablegas_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L418
[endtxhook_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L429
[l1blockhash_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L617
[l1blocknumber_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/tx_processor.go#L600

## Interfaces and components

### [`APIBackend`][apibackend_link]

APIBackend implements the [`ethapi.Backend`][ethapi.backend_link] interface, which allows simple integration of the Arbitrum chain to existing Geth API. Most calls are answered using the Backend member.

[apibackend_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/arbitrum/apibackend.go#L34
[ethapi.backend_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/internal/ethapi/backend.go#L42

### [`Backend`][backend_link]

This struct was created as an Arbitrum equivalent to the [`Ethereum`][ethereum_link] struct. It is mostly glue logic, including a pointer to the ArbInterface interface.

[backend_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/arbitrum/backend.go#L15
[ethereum_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/eth/backend.go#L68

### [`ArbInterface`][arbinterface_link]

This interface is the main interaction-point between geth-standard APIs and the Arbitrum chain. Geth APIs mostly either check status by working on the Blockchain struct retrieved from the [`Blockchain`][blockchain_link] call, or send transactions to Arbitrum using the [`PublishTransactions`][publishtransactions_link] call.

[arbinterface_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/arbitrum/arbos_interface.go#L10
[blockchain_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/arbitrum/arbos_interface.go#L12
[publishtransactions_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/arbitrum/arbos_interface.go#L11

### [`RecordingKV`][recordingkv_link]

RecordingKV is a read-only key-value store, which retrieves values from an internal trie database. All values accessed by a RecordingKV are also recorded internally. This is used to record all preimages accessed during block creation, which will be needed to prove execution of this particular block.
A [`RecordingChainContext`][recordingchaincontext_link] should also be used, to record which block headers the block execution reads (another option would be to always assume the last 256 block headers were accessed).
The process is simplified using two functions: [`PrepareRecording`][preparerecording_link] creates a stateDB and chaincontext objects, running block creation process using these objects records the required preimages, and [`PreimagesFromRecording`][preimagesfromrecording_link] function extracts the preimages recorded.

[recordingkv_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/arbitrum/recordingdb.go#L22
[recordingchaincontext_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/arbitrum/recordingdb.go#L123
[preparerecording_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/arbitrum/recordingdb.go#L152
[preimagesfromrecording_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/arbitrum/recordingdb.go#L174

## Transaction Types

Nitro Geth includes a few L2-specific transaction types. Click on any to jump to their section.

| Tx Type                                           | Represents                           | Last Hook Reached &nbsp;   | Source |
| :------------------------------------------------ | :----------------------------------- | :------------------------- | ------ |
| [`ArbitrumUnsignedTx`][arbtxunsigned]             | An L1 to L2 message                  | [`EndTxHook`][he]          | Bridge |
| [`ArbitrumContractTx`][arbtxcontract]             | A nonce-less L1 to L2 message &nbsp; | [`EndTxHook`][he]          | Bridge |
| [`ArbitrumDepositTx`][arbtxdeposit]               | A user deposit                       | [`StartTxHook`][hs]        | Bridge |
| [`ArbitrumSubmitRetryableTx`][arbtxsubmit] &nbsp; | Creating a retryable                 | [`StartTxHook`][hs] &nbsp; | Bridge |
| [`ArbitrumRetryTx`][arbtxretry]                   | A retryable redeem attempt           | [`EndTxHook`][he]          | L2     |
| [`ArbitrumInternalTx`][arbtxinternal]             | ArbOS state update                   | [`StartTxHook`][hs]        | ArbOS  |

[arbtxunsigned]: #ArbitrumUnsignedTx
[arbtxcontract]: #ArbitrumContractTx
[arbtxsubmit]: #ArbitrumSubmitRetryableTx
[arbtxretry]: #ArbitrumRetryTx
[arbtxdeposit]: #ArbitrumDepositTx
[arbtxinternal]: #ArbitrumInternalTx
[hs]: #StartTxHook
[he]: #EndTxHook

The following reference documents each type.

### [`ArbitrumUnsignedTx`][arbitrumunsignedtx_link]{#ArbitrumUnsignedTx}

Provides a mechanism for a user on L1 to message a contract on L2. This uses the bridge for authentication rather than requiring the user's signature. Note, the user's acting address will be remapped on L2 to distinguish them from a normal L2 caller.

### [`ArbitrumContractTx`][arbitrumcontracttx_link]{#ArbitrumContractTx}

These are like an [`ArbitrumUnsignedTx`][arbitrumunsignedtx_link] but are intended for smart contracts. These use the bridge's unique, sequential nonce rather than requiring the caller specify their own. An L1 contract may still use an [`ArbitrumUnsignedTx`][arbitrumunsignedtx_link], but doing so may necessitate tracking the nonce in L1 state.

### [`ArbitrumDepositTx`][arbitrumdeposittx_link]{#ArbitrumDepositTx}

Represents a user deposit from L1 to L2. This increases the user's balance by the amount deposited on L1.

### [`ArbitrumSubmitRetryableTx`][arbitrumsubmitretryabletx_link]{#ArbitrumSubmitRetryableTx}

Represents a retryable submission and may schedule an [`ArbitrumRetryTx`](#ArbitrumRetryTx) if provided enough gas. Please see the [retryables documentation](/how-arbitrum-works/arbos/introduction.md#Retryables) for more info.

### [`ArbitrumRetryTx`][arbitrumretrytx_link]{#ArbitrumRetryTx}

These are scheduled by calls to the `redeem` method of the [ArbRetryableTx](/build-decentralized-apps/precompiles/02-reference.mdx#arbretryabletx) precompile and via retryable auto-redemption. Please see the [retryables documentation](/how-arbitrum-works/arbos/introduction.md#Retryables) for more info.

### [`ArbitrumInternalTx`][arbitruminternaltx_link]{#ArbitrumInternalTx}

Because tracing support requires ArbOS's state-changes happen inside a transaction, ArbOS may create a transaction of this type to update its state in-between user-generated transactions. Such a transaction has a [`Type`][internaltype_link] field signifying the state it will update, though currently this is just future-proofing as there's only one value it may have. Below are the internal transaction types.

#### [`InternalTxStartBlock`][arbinternaltxstartblock_link]

Updates the L1 block number and L1 base fee. This transaction [is generated][block_generated_link] whenever a new block is created. They are [guaranteed to be the first][block_first_link] in their L2 block.

[arbitrumunsignedtx_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/arb_types.go#L43
[arbitrumcontracttx_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/arb_types.go#L104
[arbitrumsubmitretryabletx_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/arb_types.go#L232
[arbitrumretrytx_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/arb_types.go#L161
[arbitrumdeposittx_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/arb_types.go#L338
[arbitruminternaltx_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/internal_tx.go
[internaltype_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/arb_types.go#L387
[arbinternaltxstartblock_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/internal_tx.go#L22
[block_generated_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/block_processor.go#L181
[block_first_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/block_processor.go#L182

## Transaction Run Modes and Underlying Transactions

A [geth message][geth_message_link] may be processed for various purposes. For example, a message may be used to estimate the gas of a contract call, whereas another may perform the corresponding state transition. Nitro Geth denotes the intent behind a message by means of its [`TxRunMode`][txrunmode_link], [which it sets][set_run_mode_link] before processing it. ArbOS uses this info to make decisions about the transaction the message ultimately constructs.

A message [derived from a transaction][asmessage_link] will carry that transaction in a field accessible via its [`UnderlyingTransaction`][underlying_link] method. While this is related to the way a given message is used, they are not one-to-one. The table below shows the various run modes and whether each could have an underlying transaction.

| Run Mode                                 | Scope                   | Carries an Underlying Tx?                                                                                    |
| :--------------------------------------- | :---------------------- | :----------------------------------------------------------------------------------------------------------- |
| [`MessageCommitMode`][mc0]               | state transition &nbsp; | always                                                                                                       |
| [`MessageGasEstimationMode`][mc1] &nbsp; | gas estimation          | when created via [NodeInterface](/build-decentralized-apps/nodeinterface/02-reference.mdx) or when scheduled |
| [`MessageEthcallMode`][mc2]              | eth_calls               | never                                                                                                        |

[mc0]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/transaction.go#L654
[mc1]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/transaction.go#L655
[mc2]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/transaction.go#L656
[geth_message_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/transaction.go#L634
[txrunmode_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/transaction.go#L701
[set_run_mode_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/internal/ethapi/api.go#L955
[asmessage_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/transaction.go#L676
[underlying_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/transaction.go#L700

## Arbitrum Chain Parameters

Nitro's Geth may be configured with the following [l2-specific chain parameters][chain_params_link]. These allow the rollup creator to customize their rollup at genesis.

### `EnableArbos`

Introduces [ArbOS](/how-arbitrum-works/arbos/introduction.md), converting what would otherwise be a vanilla L1 chain into an L2 Arbitrum rollup.

### `AllowDebugPrecompiles`

Allows access to debug precompiles. Not enabled for Arbitrum One. When false, calls to debug precompiles will always revert.

### `DataAvailabilityCommittee`

Currently does nothing besides indicate that the rollup will access a data availability service for preimage resolution in the future. This is not enabled for Arbitrum One, which is a strict state-function of its L1 inbox messages.

[chain_params_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/params/config_arbitrum.go#L25

## Miscellaneous Geth Changes

### ABI Gas Margin

Vanilla Geth's abi library submits txes with the exact estimate the node returns, employing no padding. This means a transaction may revert should another arriving just before even slightly change the transaction's codepath. To account for this, we've added a `GasMargin` field to `bind.TransactOpts` that [pads estimates][pad_estimates_link] by the number of basis points set.

### Conservation of L2 ETH

The total amount of L2 ether in the system should not change except in controlled cases, such as when bridging. As a safety precaution, ArbOS checks Geth's [balance delta][conservation_link] each time a block is created, [alerting or panicking][alert_link] should conservation be violated.

### MixDigest and ExtraData

To aid with [outbox proof construction][proof_link], the root hash and leaf count of ArbOS's [send merkle accumulator][merkle_link] are stored in the `MixDigest` and `ExtraData` fields of each L2 block. The yellow paper specifies that the `ExtraData` field may be no larger than 32 bytes, so we use the first 8 bytes of the `MixDigest`, which has no meaning in a system without miners/stakers, to store the send count.

### Retryable Support

Retryables are mostly implemented in [ArbOS](/how-arbitrum-works/arbos/introduction.md#retryables). Some modifications were required in Geth to support them.

- Added ScheduledTxes field to ExecutionResult. This lists transactions scheduled during the execution. To enable using this field, we also pass the ExecutionResult to callers of ApplyTransaction.
- Added gasEstimation param to DoCall. When enabled, DoCall will also also executing any retryable activated by the original call. This allows estimating gas to enable retryables.

### Added accessors

Added [`UnderlyingTransaction`][underlyingtransaction_link] to Message interface
Added [`GetCurrentTxLogs`](https://github.com/OffchainLabs/go-ethereum/tree/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/state/statedb_arbitrum.go) to StateDB
We created the AdvancedPrecompile interface, which executes and charges gas with the same function call. This is used by [Arbitrum's precompiles](/build-decentralized-apps/precompiles/01-overview.mdx), and also wraps Geth's standard precompiles.

### WASM build support

The WASM Arbitrum executable does not support file operations. We created [`fileutil.go`](https://github.com/OffchainLabs/go-ethereum/tree/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/rawdb/fileutil.go) to wrap fileutil calls, stubbing them out when building WASM. [`fake_leveldb.go`](https://github.com/OffchainLabs/go-ethereum/tree/7503143fd13f73e46a966ea2c42a058af96f7fcf/ethdb/leveldb/fake_leveldb.go) is a similar WASM-mock for leveldb. These are not required for the WASM block-replayer.

### Types

Arbitrum introduces a new [`signer`](https://github.com/OffchainLabs/go-ethereum/tree/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/arbitrum_signer.go), and multiple new [`transaction types`](https://github.com/OffchainLabs/go-ethereum/tree/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/types/transaction.go).

### ReorgToOldBlock

Geth natively only allows reorgs to a fork of the currently-known network. In nitro, reorgs can sometimes be detected before computing the forked block. We added the [`ReorgToOldBlock`](https://github.com/OffchainLabs/go-ethereum/tree/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/blockchain_arbitrum.go#L38) function to support reorging to a block that's an ancestor of current head.

### Genesis block creation

Genesis block in nitro is not necessarily block #0. Nitro supports importing blocks that take place before genesis. We split out [`WriteHeadBlock`][writeheadblock_link] from genesis.Commit and use it to commit non-zero genesis blocks.

[pad_estimates_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/accounts/abi/bind/base.go#L355
[conservation_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/state/statedb.go#L42
[alert_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/block_processor.go#L424
[proof_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/system_tests/outbox_test.go#L26
[merkle_link]: https://github.com/OffchainLabs/nitro/blob/8e786ec6d1ac3862be85e0c9b5ac79cbd883791c/arbos/merkleAccumulator/merkleAccumulator.go#L13
[underlyingtransaction_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/state_transition.go#L69
[writeheadblock_link]: https://github.com/OffchainLabs/go-ethereum/blob/7503143fd13f73e46a966ea2c42a058af96f7fcf/core/genesis.go#L415
