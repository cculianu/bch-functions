# CHIP-2025-08 Functions Take 2: Function Definition and Invocation Operations, Take II

        Title: Function Definition and Invocation Operations, Take II
        Type: Standards
        Layer: Consensus
        Maintainers: Jason Dreyzehner, Calin Culianu
        Status: Draft
        Initial Publication Date: 2024-12-12
        Latest Revision Date: 2025-08-11
        Version: 2.2.0

## Executive Summary

This proposal is a fork of, and pretty much indentical to: [CHIP-2025-05 Functions: Function Definition and Invocation Operations](https://github.com/bitjson/bch-functions). The only difference is that this proposal adds an additional flag `fCanDefineFunctions` which restricts `OP_DEFINE`. In this proposal, `OP_DEFINE` is restricted to only be allowed to execute when this flag is `true`. The rules governing this flag prohibit "code that writes code". A description of this flag and how it operarates [follows in the subsequent proposal below](#can-define-functions-flag-fcandefinefunctions), which is more-or-less the same as the original proposal (with additions that describe `fCanDefineFunctions` and how it restricts `OP_DEFINE`).  What follows is the amended original proposal:

---


## Summary

This proposal introduces the `OP_DEFINE` and `OP_INVOKE` opcodes, enabling Bitcoin Cash contract bytecode to be factored into reusable functions.

## Motivation & Benefits

- **Reduced transaction sizes** – By eliminating duplicated bytecode, contract lengths can be optimized for a wider variety of use cases: finite field arithmetic, pairing-based cryptography, zero-knowledge proof systems, homomorphic encryption, post-quantum cryptography, and other important applications for the future security and competitiveness of Bitcoin Cash.

- **Stronger privacy and operational security** – By enabling contract designs which leak less information about security measures, current assets, and transaction history, this proposal [strengthens privacy and operational security](./alternatives.md#improved-privacy-and-operational-security-vs-status-quo) for a variety of use cases.

- **Improved auditability** – Without reusable functions, contracts are forced to unnecessarily duplicate significant logic. This proposal enables contracts to be written using more succinct, auditable patterns.

## Deployment

Deployment of this specification is proposed for the May 2026 upgrade.

- Activation is proposed for `1763208000` MTP, (`2025-11-15T12:00:00.000Z`) on `chipnet`.
- Activation is proposed for `1778846400` MTP, (`2026-05-15T12:00:00.000Z`) on the BCH network (`mainnet`), `testnet3`, `testnet4`, and `scalenet`.

## Technical Specification

The virtual machine is modified to add a [Function Table](#function-table) of immutable functions, the `OP_DEFINE` opcode is defined at codepoint `0x89` (`137`), and the `OP_INVOKE` opcode is defined at `0x8a` (`138`).

### Function Table

The virtual machine (VM) is modified to add a new data structure: the `Function Table` is a data structure (such as a map or sparse array) that holds immutable byte vectors; it maps each defined function identifier to an immutable function body (the byte vector).

Due to VM limits, the function table does not exceed [Maximum Cumulative Depth](#maximum-cumulative-depth-max_cumulative_depth) in length/count, and the function table's held byte vectors do not exceed the [Stack Element Length Limit](https://github.com/bitjson/bch-vm-limits?#increased-stack-element-length-limit).

#### Defined Function Count

A `Defined Function Count` counter is added to track the count of defined functions.

#### Function Identifier

A function identifier – the function's index in the [function table](#function-table) – is an integer greater than or equal to `0` and less than [Maximum Cumulative Depth](#maximum-cumulative-depth-max_cumulative_depth) (currently, `1000`).

#### Maximum Cumulative Depth (`MAX_CUMULATIVE_DEPTH`)

The existing cumulative stack and altstack depth limit (A.K.A. `MAX_STACK_SIZE`; 1000 items) is modified to incorporate `Defined Function Count`: the sum of stack depth, alternate stack depth, and `Defined Function Count` must be less than `1000`.

#### Can Define Functions Flag (`fCanDefineFunctions`)

A flag is introduced, `fCanDefineFunctions`, which is initially set to `true` at the start of an evaluation.  It remains `true` so long as the code being executed is one of the following opcodes:

- A data push opcode: `OP_0`, `OP_1` through `OP_16`, `OP_1NEGATE`, direct data push <= 75 bytes, `OP_PUSHDATA1`, `OP_PUSHDATA2`, or `OP_PUSHDATA4`
- An `OP_DEFINE` opcode
- A stack manipulation opcode in the following inclusive range: `OP_TOALTSTACK` (`0x6b`) through `OP_TUCK` (`0x7d`), **except** for `OP_DEPTH` (`0x74`)

For any other opcode (such as `OP_CAT`, `OP_SPLIT`, `OP_ADD`, `OP_IF`, `OP_HASH160`, `OP_DEPTH`, etc), the flag gets latched to `false` for the remainder of the evaluation. If this flag is `false`, subsequent `OP_DEFINE` opcodes encountered in the script will fail the script immediately.

The intent of this new flag is to prevent "code that writes code" situations. This flag forces scripts to essentially push all the code they intend to use as functions up-front, and disallows on-the-fly construction of stack data items intended to be used as code, such that all the code a script will execute as functions is immediately apparent and forced to exist "at the top" of the script bytecode, (more or less). In essence, this flag restricts precisely when and how `OP_DEFINE` can be used.

It is intended that this flag, and thus the restriction it introduces, will constrain scripts that make use of functions to be more easily amenable to static analysis.

This flag (and thus the restriction on how `OP_DEFINE` can be used) may be removed in a future Bitcoin Cash network upgrade in order to explicitly allow "code that writes code".

##### Notice of Possible Future Expansion

While unusual, it is possible to design pre-signed transactions, contract systems, and protocols which rely on the rejection of otherwise-valid transactions made invalid only by specifically exceeding one or more current VM limits. This proposal interprets such failure-reliant constructions as intentional – the constructions are designed to fail unless/until a possible future network upgrade in which such limits are increased, e.g. upgrade-activation futures contracts. Contract authors are advised that future upgrades may raise VM limits by increasing [Maximum Cumulative Depth](#maximum-cumulative-depth-max_cumulative_depth), or otherwise. See [Limits CHIP Rationale: Inclusion of "Notice of Possible Future Expansion"](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#inclusion-of-notice-of-possible-future-expansion).

#### Reset Before Each Bytecode Evaluation

Prior to each phase of evaluation (unlocking bytecode, locking bytecode, and redeem bytecode), the [`Function Table`](#function-table), [`Defined Function Count`](#defined-function-count), and [`fCanDefineFunctions` flag](#can-define-functions-flag-fcandefinefunctions) must be reset to an empty table, `0`, and `true` respectively.

### `OP_DEFINE`

```CashAssembly
<function_body> <function_identifier> OP_DEFINE
```

The `OP_DEFINE` opcode is defined at codepoint `0x89` (`137`) with the following behavior:

1. If the [`fCanDefineFunctions`](#can-define-functions-flag-fcandefinefunctions) flag is `false`, fail execution immediately, otherwise:
2. Pop the top item from the stack to interpret as a function identifier (a VM Number between `0` and `999`, inclusive).<sup>1</sup>
3. Pop the next item from the stack to interpret as the function body<sup>2</sup>, and copy it to the [function table](#function-table) at the index equal to the function identifier. If that function index is out of range or already defined, error.<sup>3</sup>
4. Increment `Defined Function Count` by one.<sup>4</sup>

<small>

#### `OP_DEFINE` Clarifications

1. If the stack is empty (no `function_identifier`), error. If the popped item is not a VM Number between `0` and `999` (inclusive), error. See [Rationale: Use of Stack-Based Parameters](./rationale.md#use-of-stack-based-parameters) and [Rationale: Use of Positive Integer-Based Function Identifiers](./rationale.md#use-of-positive-integer-based-function-identifiers).
2. if the stack is empty (no `function_body`), error. Note that any stack item is a valid function body (including empty stack items); implementations must not attempt to parse the function body until invoked by `OP_INVOKE`. See [Rationale: Deferred Parsing of Function Bodies](./rationale.md#deferred-parsing-of-function-bodies).
3. See [Rationale: Immutability of Function Bodies](./rationale.md#immutability-of-function-bodies). Note also that function identifiers/table indexes may be defined in any order. See [Rationale: Support for Skipping Function Identifiers](./rationale.md#support-for-skipping-function-identifiers).
4. Note that the [Operation Cost](http://github.com/bitjson/bch-vm-limits#operation-cost-limit) of `OP_DEFINE` is equal to the [Base Instruction Cost](https://github.com/bitjson/bch-vm-limits?#base-instruction-cost).

</small>

### `OP_INVOKE`

```CashAssembly
<function_identifier> OP_INVOKE
```

The `OP_INVOKE` opcode is defined at codepoint `0x8a` (`138`) with the following behavior:

1. Pop the top item from the stack to interpret as a function identifier.<sup>1</sup>
2. Preserve the active bytecode (A.K.A. `script`), instruction pointer (A.K.A. program counter or `pc`), and index of the last executed code separator (A.K.A. `pbegincodehash`) by pushing them to the top of the control stack.<sup>2</sup> (Note that this subjects function invocations to the existing control stack depth limit of `100`.)
3. Reset the instruction pointer and last executed code separator, then execute the function body as if it were the active bytecode.<sup>3</sup> If the bytecode is malformed (i.e. a push operation requires more bytes than are available in the remaining segment of bytecode to be parsed), error.
4. When the evaluation is complete<sup>4</sup>, restore the original bytecode, instruction pointer, and last executed code separator, then continue evaluation after the OP_INVOKE instruction.<sup>5</sup>

<small>

#### OP_INVOKE Clarifications

1. If the stack is empty (no `function_identifier`), error. If the popped item is not a VM Number representing a previously-defined function in the [function table](#function-table), error. If the referenced function body has a length of zero: error if the control stack depth limit of `100` would be exceeded by a non-empty evaluation; otherwise, continue after the `OP_INVOKE` instruction.
2. Note that this requires the control stack to be capable of holding a new **stack frame** data type to preserve the state of the parent evaluation.
3. `OP_INVOKE`ed bytecode otherwise shares the context of the parent evaluation:
   1. Invoked functions may modify the stack and alternate stack without limitation, and control stack usage (e.g. `OP_IF/OP_END_IF` or further `OP_INVOKE`s) within the evaluation remains restricted by the 100-item depth limit.
   2. Executed `OP_CODESEPARATOR` operations record the index of the current instruction pointer (A.K.A. `pc`) within the `OP_INVOKE`ed bytecode as the last executed code separator (A.K.A. `pbegincodehash`).
   3. The `OP_ACTIVEBYTECODE` operation produces the serialization of the active bytecode beginning from the last executed code separator.
   4. In signature operations, the covered bytecode includes only the active bytecode beginning from the last executed code separator (or if none have been executed in the current evaluation, the full active bytecode).
4. For an `OP_INVOKE` evaluation to complete successfully, the top element remaining on the control stack must be a stack frame to resume after the evaluation. (E.g. Validation fails if an `OP_IF` within the evaluation is not resolved by a matching `OP_ENDIF` within the same evaluation.)
5. After an active bytecode has been fully evaluated, the next stack frame is resumed (restoring its bytecode, instruction pointer, and last executed code separator values) until all stack frames are complete (the control stack is empty) or the evaluation has produced some error. Note that two or more `OP_INVOKE` operations may be resolved within the same virtual machine step (e.g. if a child `OP_INVOKE` is the final instruction of a parent `OP_INVOKE` evaluation). Note that the [Operation Cost](http://github.com/bitjson/bch-vm-limits#operation-cost-limit) of `OP_INVOKE` is equal to the [Base Instruction Cost](https://github.com/bitjson/bch-vm-limits?#base-instruction-cost).

</small>

## Rationale

- [Appendix: Rationale &rarr;](rationale.md#rationale)
  - [Simple, Statically-Applicable Contract Length Optimizations](rationale.md#simple-statically-applicable-contract-length-optimizations)
  - [Immutability of Function Bodies](rationale.md#immutability-of-function-bodies)
  - [Deferred Parsing of Function Bodies](rationale.md#deferred-parsing-of-function-bodies)
  - [Use of Stack-Based Parameters](rationale.md#use-of-stack-based-parameters)
  - [Use of Positive Integer-Based Function Identifiers](rationale.md#use-of-positive-integer-based-function-identifiers)
  - [Support for Skipping Function Identifiers](rationale.md#support-for-skipping-function-identifiers)
  - [Opcode Naming: `OP_DEFINE` and `OP_INVOKE`](rationale.md#opcode-naming-op_define-and-op_invoke)
  - [Preservation of Alternate Stack](rationale.md#preservation-of-alternate-stack)
  - [Preservation of `OP_CODESEPARATOR` Support](rationale.md#preservation-of-op_codeseparator-support)
  - [Non-Impact on Performance or Static Analysis](rationale.md#non-impact-on-performance-or-static-analysis)
    - [Estimation of Contract Validation Cost](rationale.md#estimation-of-contract-validation-cost)

## Evaluations of Alternatives

- [Appendix: Evaluations of Alternatives &rarr;](alternatives.md#evaluation-of-alternatives)
  - [Status Quo: Duplicated Logic](alternatives.md#status-quo-duplicated-logic)
  - [Status Quo: Sidecar Inputs ("Emulated `OP_EVAL`")](alternatives.md#status-quo-sidecar-inputs-emulated-op_eval)
    - [Improved Privacy and Operational Security vs. Status Quo](alternatives.md#improved-privacy-and-operational-security-vs-status-quo)
    - [Improved Efficiency and User Experience vs. Status Quo](alternatives.md#improved-efficiency-and-user-experience-vs-status-quo)
  - [Alternative: Multi-Byte `OP_DEFINE` and `OP_INVOKE` Opcodes](alternatives.md#alternative-multi-byte-op_define-and-op_invoke-opcodes)
  - [Alternative: BIP12 `OP_EVAL`](alternatives.md#alternative-bip12-op_eval)
  - [Alternative: CHIP 2024-12 OP_EVAL: Function Evaluation](alternatives.md#alternative-chip-2024-12-op_eval-function-evaluation)
  - [Alternative: Output-Level Function Annex](alternatives.md#alternative-output-level-function-annex)
  - [Alternative: OP_EXEC](alternatives.md#alternative-op_exec)

## Risk Assessment

- [Appendix: Risk Assessment &rarr;](risk-assessment.md#risk-assessment)
  - [Risks \& Security Considerations](risk-assessment.md#risks--security-considerations)
    - [User Impact Risks](risk-assessment.md#user-impact-risks)
      - [Reduced or Equivalent Node Validation Costs](risk-assessment.md#reduced-or-equivalent-node-validation-costs)
      - [Increased or Equivalent Contract Capabilities](risk-assessment.md#increased-or-equivalent-contract-capabilities)
    - [Consensus Risks](risk-assessment.md#consensus-risks)
      - [Full-Transaction Test Vectors](risk-assessment.md#full-transaction-test-vectors)
      - [Additional Performance Benchmarks](risk-assessment.md#additional-performance-benchmarks)
      - [`Chipnet` Preview Activation](risk-assessment.md#chipnet-preview-activation)
    - [Denial-of-Service (DoS) Risks](risk-assessment.md#denial-of-service-dos-risks)
      - [Node Performance Safety Margin](risk-assessment.md#node-performance-safety-margin)
    - [Protocol Complexity Risks](risk-assessment.md#protocol-complexity-risks)
      - [Support for Post-Activation Simplification](risk-assessment.md#support-for-post-activation-simplification)
      - [Evaluation of Alternatives](risk-assessment.md#evaluation-of-alternatives)
  - [Upgrade Costs](risk-assessment.md#upgrade-costs)
    - [Node Upgrade Costs](risk-assessment.md#node-upgrade-costs)
    - [Ecosystem Upgrade Costs](risk-assessment.md#ecosystem-upgrade-costs)
  - [Maintenance Costs](risk-assessment.md#maintenance-costs)
    - [Node Maintenance Costs](risk-assessment.md#node-maintenance-costs)
    - [Ecosystem Maintenance Costs](risk-assessment.md#ecosystem-maintenance-costs)

## Test Vectors

This proposal includes [a suite of functional tests and benchmarks](./vmb_tests/) to verify the performance of all operations within virtual machine implementations.

## Implementations

Please see the following implementations for examples and additional test vectors:

- C++:
  - [Bitcoin Cash Node (BCHN)](https://bitcoincashnode.org/) – A professional, miner-friendly node that solves practical problems for Bitcoin Cash. [Merge Request !1937](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/-/merge_requests/1937).
- **JavaScript/TypeScript**:
  - [Libauth](https://github.com/bitauth/libauth) – An ultra-lightweight, zero-dependency JavaScript library for Bitcoin Cash. [Branch `next`](https://github.com/bitauth/libauth/tree/next).
  - [Bitauth IDE](https://github.com/bitauth/bitauth-ide) – An online IDE for bitcoin (cash) contracts. [Branch `next`](https://github.com/bitauth/bitauth-ide/tree/next).

## Feedback & Reviews

- [Functions CHIP Issues](https://github.com/bitjson/bch-functions/issues)
- [`CHIP-2025-05 Functions: Function Definition and Invocation Operations` - Bitcoin Cash Research](https://bitcoincashresearch.org/t/chip-2025-05-functions-function-definition-and-invocation-operations/1576)

## Changelog

This section summarizes the evolution of this document.

- **v2.1.0 - 2025-08-11**
  - Come up with the `fCanDefineFunctions` scheme to prohibit "code that writes code". 
- **v2.0.0 – 2025-05-27**
  - Split `OP_EVAL` into `OP_DEFINE` and `OP_INVOKE`
- **v1.0.1 – 2025-05-02**
  - Clarify description ([#1](https://github.com/bitjson/bch-functions/pull/1))
  - Add [Rationale](./rationale.md)
  - Add [Evaluation of Alternatives](./alternatives.md)
  - Add [Risk Assessment](./risk-assessment.md)
  - Commit latest test vectors
  - Link to BCHN implementation
- **v1.0.0 – 2024-12-12**
  - Initial publication

## Copyright

This document is placed in the public domain.
