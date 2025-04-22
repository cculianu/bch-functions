# CHIP-2024-12 OP_EVAL: Function Evaluation

        Title: Function Evaluation
        Type: Standards
        Layer: Consensus
        Maintainer: Jason Dreyzehner
        Status: Draft
        Initial Publication Date: 2024-12-12
        Latest Revision Date: 2024-12-12
        Version: 1.0.0

## Summary

This proposal introduces the `OP_EVAL` operation, enabling Bitcoin Cash contract bytecode to be factored into reusable functions.

## Motivation & Benefits

- **Improved auditability** – Without reusable functions, existing contracts are forced to unnecessarily duplicate code segments, often with additional stack management operations that can easily obfuscate unintended behavior. By enabling function evaluation, this proposal enables contracts to be written using more succinct, auditable patterns.

- **Reduced transaction sizes** – By eliminating duplicated bytecode, contract lengths can be optimized for a wide variety of use cases: finite field arithmetic, pairing-based cryptography, zero-knowledge proof systems, homomorphic encryption, post-quantum cryptography, and other important applications for the future security and competitiveness of Bitcoin Cash.

## Deployment

Deployment of this specification is proposed for the May 2026 upgrade.

- Activation is proposed for `1763208000` MTP, (`2025-11-15T12:00:00.000Z`) on `chipnet`.
- Activation is proposed for `1778846400` MTP, (`2026-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, `testnet4`, and `scalenet`.

## Technical Specification

The `OP_EVAL` opcode is defined at codepoint `0x62` (`98`) with the following behavior:

1. Pop the top item from the stack to interpret as bytecode.<sup>1</sup>
2. Preserve the active bytecode (A.K.A. `script`), instruction pointer (A.K.A. program counter, or `pc`), and index of the last executed code separator (A.K.A. `pbegincodehash`) by pushing them to the top of the control stack.<sup>2</sup> (Note that this subjects `OP_EVAL` evaluations to the existing control stack depth limit of `100`.)
3. Reset the `pc` and `pbegincodehash` to the beginning of the new bytecode and proceed to evaluate the stack-provided bytecode as if it were the active bytecode, without modifying other evaluation context.<sup>3</sup> If the bytecode is malformed (i.e. a push operation requires more bytes than are available in the remaining segment of bytecode to be parsed), error.
4. When the evaluation is complete, restore the original bytecode, along with the preserved `pc` and `pbegincodehash`, and continue evaluation after the OP_EVAL instruction.<sup>4</sup>

#### Clarifications

1. If the stack is empty, error. If the top stack item has a length of zero, implementations may choose to evaluate the empty bytecode (which will complete immediately) or to continue as with any other bytecode. Implementations are advised that in this latter case, the VM should behave "as if" it had executed an empty script, and as such an `OP_EVAL` of a zero-length script should still be considered as having temporarily increased the control stack depth, such that the implementation must produce an error should the control stack depth exceed `100`.
2. Note that this requires the control stack to be capable of holding a new **stack frame** data type to preserve the state of the preserved evaluation.
3. `OP_EVAL`-executed bytecode shares the same stack and alternate stack as the parent bytecode that created it (i.e. modifications to these stacks are be visible to the parent bytecode):
   1. Evaluations performed by `OP_EVAL` can be understood as efficient function calls: the evaluation may modify the stack and alternate stack without limitation, and the `OP_EVAL` evaluation's control stack usage remains restricted by the existing 100-item depth limit, and any additional items added to the control stack during the `OP_EVAL` evaluation contribute towards this limit.
   2. The `OP_CODESEPARATOR` operation records the index of the current instruction pointer (A.K.A. `pc`) within the `OP_EVAL`-ed bytecode (and saves it to `pbegincodehash`).
   3. The `OP_ACTIVEBYTECODE` operation produces the serialization of the active bytecode beginning from the last executed code separator (A.K.A. `pbegincodehash`), or from the beginning of the bytecode should no such `OP_CODESEPARATOR` instruction have been previously executed during the `OP_EVAL` evaluation.
   4. Similarly, in signature operations, the covered bytecode includes only the active bytecode beginning from the last executed code separator (A.K.A. `pbegincodehash`), or from the bytecode's beginning if no code separator has been seen.
4. After an active bytecode has been fully evaluated, the previous stack frame is resumed (and `pc` and `pbegincodehash` are restored to their preserved values) until all stack frames are complete (the control stack is empty) or the evaluation has produced some error. Note that two or more `OP_EVAL` operations may be resolved within the same virtual machine step (e.g. if a child `OP_EVAL` is the final instruction of a parent `OP_EVAL` evaluation).

## Implementations

Please see the following implementations for additional examples and test vectors:

- **JavaScript/TypeScript**:
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash. [Branch `next`](https://github.com/bitauth/libauth/tree/next).
  - [Bitauth IDE](https://github.com/bitauth/bitauth-ide) – An online IDE for bitcoin (cash) contracts. [Branch `next`](https://github.com/bitauth/bitauth-ide/tree/next).

## Feedback & Reviews

- [OP_EVAL CHIP Issues](https://github.com/bitjson/bch-eval/issues)
- [`CHIP 2024-12 OP_EVAL: Function Evaluation` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2024-12-op-eval-function-evaluation/1450)

## Changelog

This section summarizes the evolution of this document.

- **v1.0.0 – 2024-12-12**
  - Initial publication

## Copyright

This document is placed in the public domain.
