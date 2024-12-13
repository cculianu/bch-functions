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
2. Preserve the active bytecode (A.K.A. `script`), instruction pointer (A.K.A. program counter), and index of the last executed code separator (A.K.A. `pbegincodehash`) at the top of the control stack.<sup>2</sup> (Note that this subjects `OP_EVAL` evaluations to the existing control stack depth limit of `100`.)
3. Evaluate the stack-provided bytecode as if it were the active bytecode, without modifying other evaluation context.<sup>3</sup> If the bytecode is malformed (i.e. a push operation requires more bytes than are available in the remaining segment of bytecode to be parsed), error.
4. When the evaluation is complete, restore the original bytecode and continue evaluation after the OP_EVAL instruction.<sup>4</sup>

#### Clarifications

1. If the stack is empty, error. If the top stack item has a length of zero, continue as with any other bytecode. (Note that like all other evaluations, an `OP_EVAL` of zero-length bytecode will still produce an error at a control stack depth of `100`.)
2. Note that this requires the control stack to be capable of holding a new **stack frame** data type to preserve the current state of the evaluation.
3. Other than the popped bytecode and newly-preserved stack frame, `OP_EVAL` does not interfere with the stack, alternate stack, control stack, or other evaluation context:
   1. Evaluations performed by `OP_EVAL` can be understood as efficient function calls: the evaluation may modify the stack and alternate stack without limitation, and the `OP_EVAL` evaluation's control stack usage remains restricted by the existing 100-item depth limit ().
   2. The `OP_CODESEPARATOR` operation records the index of the current instruction pointer (A.K.A. `pc`) within the `OP_EVAL`-ed bytecode.
   3. The `OP_ACTIVEBYTECODE` operation produces the serialization of the active bytecode beginning from the last executed code separator (A.K.A. `pbegincodehash`).
   4. In signature operations, the covered bytecode includes only the active bytecode beginning from the last executed code separator (A.K.A. `pbegincodehash`).
4. After an active bytecode has been fully evaluated, the next stack frame is resumed until all stack frames are complete (the control stack is empty) or the evaluation has produced some error. Note that two or more `OP_EVAL` operations may be resolved within the same virtual machine step (e.g. if a child `OP_EVAL` is the final instruction of a parent `OP_EVAL` evaluation).

## Implementations

Please see the following implementations for additional examples and test vectors:

- **JavaScript/TypeScript**:
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash. [Branch `next`](https://github.com/bitauth/libauth/tree/next).
  - [Bitauth IDE](https://github.com/bitauth/bitauth-ide) – An online IDE for bitcoin (cash) contracts. [Branch `next`](https://github.com/bitauth/bitauth-ide/tree/next).

## Feedback & Reviews

- [OP_EVAL CHIP Issues](https://github.com/bitjson/bch-eval/issues)
- [`CHIP 2024-12 OP_EVAL` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/) // TODO

## Changelog

This section summarizes the evolution of this document.

- **v1.0.0 – 2024-12-12**
  - Initial publication

## Copyright

This document is placed in the public domain.
