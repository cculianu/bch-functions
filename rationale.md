# Rationale

This section documents design decisions made in this specification.

<details>

<summary><strong>Table of Contents</strong></summary>

- [Rationale](#rationale)
  - [Preservation of Alternate Stack](#preservation-of-alternate-stack)
  - [Preservation of `OP_CODESEPARATOR` Support](#preservation-of-op_codeseparator-support)
  - [Non-Impact on Performance or Static Analysis](#non-impact-on-performance-or-static-analysis)
    - [Estimation of Contract Validation Performance](#estimation-of-contract-validation-performance)

</details>

## Preservation of Alternate Stack

Though not specified, the BIP12 implementation also [cleared the alternate stack](https://github.com/bitcoin/bitcoin/issues/729#issuecomment-3294453) at the end of each evaluation. At best, this behavior is useless: because each alternate stack item requires one byte to push (via `OP_TOALTSTACK`), any program which employed the stack clearing behavior could just as easily have replaced those bytes with equally or more efficient dropping operations (`OP_DROP`, `OP_2DROP`, or wider [stack clearing](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#increased-usability-of-multisig-stack-clearing)). Further, preventing functions from accepting or returning values in the alternate stack would limit their byte efficiency by requiring less efficient stack juggling strategies vs. the higher semantic bandwidth of passing both stacks.

## Preservation of `OP_CODESEPARATOR` Support

BIP12 required implementations to specifically scan each evaluated script for instances of `OP_CODESEPARATOR`, rejecting the transaction if found. This needlessly increases implementation complexity and eliminates some potential value: [`OP_CODESEPARATOR` remains a useful feature](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#ongoing-value-of-op_codeseparator-operation) for certain use cases and has been preserved by past upgrades (e.g. [`OP_ACTIVEBYTECODE`'s support for `OP_CODESEPARATOR`](https://github.com/bitjson/bch-2022/blob/master/CHIP-2021-02-Add-Native-Introspection-Opcodes.md#op_activebytecode-support-for-op_codeseparator)). This proposal avoids the additional implementation complexity and preserves the usefulness of `OP_CODESEPARATOR` within evaluations.

## Non-Impact on Performance or Static Analysis

The 2011 [BIP12 `OP_EVAL` proposal](https://github.com/bitcoin/bips/blob/master/bip-0012.mediawiki) was ultimately [abandoned](https://web.archive.org/web/20140420092207/https://bitcointalk.org/index.php?topic=46538.0;all#msg2256663) in favor of [BIP16 Pay to Script Hash (P2SH)](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki) due to uncertainty about `OP_EVAL`'s impact on the safety, validation performance, and static analyzability of Bitcoin Cash Virtual Machine (VM) bytecode<sup>1</sup>.

While some of these uncertainties have been addressed by [improvements vs. the BIP12 proposal](./alternatives.md#bip12-op_eval), others have been obviated by additional research and protocol upgrades in the intervening years.

### Estimation of Contract Validation Performance

While the Bitcoin VM initially had few limits, following the [2010 emergency patches](https://github.com/bitjson/bch-bigint/blob/master/rationale.md#scenario-never-increase), it could be argued that the expected runtime of a contract was (in principle) a function of the contract's length, i.e. long contracts take longer to validate than short contracts.

In practice, expensive operations (hashing and signature checking) have always dominated other operations by many orders of magnitude, but [some developers still considered](https://web.archive.org/web/20151210171935/https://github.com/bitcoin/bitcoin/issues/729#issuecomment-3283107) it a potentially useful feature that contracts could "in priciple" be "reviewed" prior to execution, limiting wasted computing resources vs. "fully evaluating" the contract.

As is now highlighted by more than a decade of non-adoption in node implementations and other software: such "review" would necessarily optimize for the uncommon case (an invalid transaction from a misbehaving peer) by penalizing performance in the common case (standard transactions) **leading to worse overall validation performance and network throughput – even if validation cost weren't dominated by the most expensive operations**.

The [2018 restoration of disabled opcodes](https://upgradespecs.bitcoincashnode.org/may-2018-reenabled-opcodes/) further reduced the plausibility of non-evaluated contract analysis by reenabling opcodes that could unpredictably branch or operate on hashes (`OP_SPLIT`, bitwise operations, etc.). For example, the pattern `OP_HASH256 OP_1 OP_SPLIT OP_DROP OP_0 OP_GREATERTHAN OP_IF OP_HASH256 ...` branches based on the result of a hash, obviating any further attempt to inspect without computing the hash. Note that simply "counting" `OP_HASH256` operations here also isn't meaningful: valid contracts can rely on many hashing operations (e.g. Merkle trees), and the more performance-relevant [digest iteration count](https://github.com/bitjson/bch-vm-limits/blob/55b9a446eef4932627d387c622ef0fce38ec9512/readme.md#digest-iteration-count) of any evaluation depends on the precise input lengths of hashed stack items; the existence of hash-based branching implies that such lengths cannot be predicted without a vulnerability in the hashing algorithm.

Later, the [SigChecks (2020)](https://upgradespecs.bitcoincashnode.org/2020-05-15-sigchecks/) resolved [long-standing issues with the SigOps limit](https://upgradespecs.bitcoincashnode.org/2020-05-15-sigchecks/#motivation) by counting signature checks over the course of evaluation rather than by attempting to naively parse VM bytecode. Finally, the [VM limits (2025)](https://github.com/bitjson/bch-vm-limits) upgrade retargeted all VM limits to use similar density-based evaluation limits.

In summary, **fast validation is a fundamental and intentional feature of the VM itself** – critical for the overall throughput of the Bitcoin Cash network. Hypothetical "pre-validation" of contracts never offered improved performance, would have unnecessarily complicated all implementing software, and has been further obviated by multiple protocol upgrades in the intervening years.
