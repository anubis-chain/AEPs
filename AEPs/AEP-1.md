<pre>
  AEP: 1
  Title: DAI as the Gas Token before the Aria Hardfork
  Status: Enabled
  Type: Standards
  Created: 2026-04-01
</pre>

# AEP-1: DAI as the Gas Token before the Aria Hardfork

- [AEP-1: DAI as the Gas Token before the Aria Hardfork](#aep-1-dai-as-the-gas-token-before-the-aria-hardfork)
  - [1. Summary](#1-summary)
  - [2. Abstract](#2-abstract)
  - [3. Motivation](#3-motivation)
  - [4. Specification](#4-specification)
    - [4.1 The DAI System Contract](#41-the-dai-system-contract)
    - [4.2 Gas Accounting](#42-gas-accounting)
    - [4.3 Transaction Validity](#43-transaction-validity)
    - [4.4 Fee Distribution](#44-fee-distribution)
    - [4.5 The Aria Transition](#45-the-aria-transition)
  - [5. Compatibility](#5-compatibility)
  - [6. License](#6-license)

## 1. Summary

Anubis launches without a native token. Until the Aria hardfork activates, transaction gas is bought, refunded and distributed in DAI, an ERC-20 system contract deployed in the genesis block. The native balance field is used only for transferring value and plays no role in fee payment.

## 2. Abstract

This AEP specifies how the execution layer charges transaction fees from the DAI system contract instead of the native account balance. Gas purchase, gas refund and fee distribution are implemented as direct reads and writes of the DAI contract's `balanceOf` storage, performed by the protocol outside of EVM execution. The mechanism is active from genesis and is switched off by the Aria hardfork (`ariaTime`), after which the standard native-token gas accounting applies.

## 3. Motivation

Anubis does not issue a native token at launch; validator staking and the native token economy are introduced later by the Aria hardfork. The network still needs a spendable, sybil-resistant fee asset from block one. Using a stablecoin as the gas token:

* gives users fees that are predictable in dollar terms;
* avoids bootstrapping a fee market around a token that does not exist yet;
* keeps the account model and transaction formats fully compatible with go-ethereum/BSC tooling — only the funding source of fees changes.

## 4. Specification

### 4.1 The DAI System Contract

DAI is an ERC-20 contract deployed in the genesis allocation at the fixed address:

```
0x83fd06F0846d9D90B3016bF670Efe2E0B11cDe14
```

Its `balanceOf` mapping resides at storage slot `0`. For a holder `a`, the balance is stored at:

```
key = keccak256(leftPad32(a) ++ leftPad32(0))
```

The protocol manipulates these slots directly through the state database (`GetState`/`SetState`). No EVM call is executed and no `Transfer` event is emitted for protocol-level gas movements.

### 4.2 Gas Accounting

For every transaction executed at a block where the Aria fork is not yet active:

* **Purchase** — `gasLimit x gasPrice` (plus the blob fee `blobGas x blobGasFeeCap` for blob transactions) is deducted from the sender's DAI balance before execution.
* **Refund** — after execution, `remainingGas x gasPrice` is credited back to the sender's DAI balance.
* **Native value** — `msg.value` continues to be paid from and validated against the sender's native balance, independently of fee accounting.

### 4.3 Transaction Validity

The upfront DAI balance check covers `gasLimit x gasFeeCap` (plus the blob fee where applicable). Because a transaction calling `DAI.transfer(to, amount)` spends the same asset that pays for gas, the decoded `amount` is added to the required balance when the transaction's recipient is the DAI contract. The transaction pool applies the same rule (`gas x gasPrice + DAI transfer amount`) when admitting transactions, and `eth_estimateGas` permits transferring the full native balance since none of it is reserved for fees.

### 4.4 Fee Distribution

As in BSC, priority fees are not paid to the coinbase directly. `gasUsed x effectiveTip`, together with the blob fee, is credited — in DAI — to the system address:

```
0xffffFFFfFFffffffffffffffFfFFFfffFFFfFFfE
```

and subsequently distributed to validators through the Parlia system contracts.

### 4.5 The Aria Transition

The switch is governed by the chain configuration:

```
IsAria(num, time) = IsLondon(num) && ariaTime != nil && time >= ariaTime
```

From the first Aria block onward, gas purchase, refund and fee distribution revert to the standard native-token accounting. DAI balances persist unchanged as ordinary ERC-20 state; only their protocol role ends.

## 5. Compatibility

The transaction formats, account model and RPC surface are unchanged. Integrators should note:

* wallets and faucets must check the DAI balance, not the native balance, to determine whether an account can pay fees before Aria;
* protocol-level gas movements do not emit ERC-20 `Transfer` events, so DAI indexers must reconcile balances against transaction receipts;
* `tx.Cost()`-style calculations that add `value` to the fee requirement do not apply before Aria — native value and DAI fees are validated against separate balances.

## 6. License

All the content are licensed under [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
