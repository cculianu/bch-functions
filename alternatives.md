# Evaluation of Alternatives

Potential alternatives to this proposal are reviewed below.

<details>

<summary><strong>Table of Contents</strong></summary>

- [Evaluation of Alternatives](#evaluation-of-alternatives)
  - [Status Quo: Sidecar Inputs ("Emulated `OP_EVAL`")](#status-quo-sidecar-inputs-emulated-op_eval)
    - [Improved Privacy and Operational Security vs. Status Quo](#improved-privacy-and-operational-security-vs-status-quo)
    - [Improved Efficiency and User Experience vs. Status Quo](#improved-efficiency-and-user-experience-vs-status-quo)
    - [Notes](#notes)
  - [BIP12 `OP_EVAL`](#bip12-op_eval)
  - [Output-Level Function Annex](#output-level-function-annex)
  - [Word Definition Operations (`OP_DEFINE` \& `OP_INVOKE`)](#word-definition-operations-op_define--op_invoke)
  - [OP_EXEC](#op_exec)

</details>

### Status Quo: Sidecar Inputs ("Emulated `OP_EVAL`")

The delegative capabilities offered by `OP_EVAL` have been available on Bitcoin Cash since the [2023 upgrade](https://cashtokens.org/docs/spec/chip), allowing contract systems to delegate authority to arbitrary, user-specified contracts via [identity tokens](https://cashtokens.org/docs/spec/examples#identity-tokens) provided in a "sidecar" input (A.K.A. "emulated `OP_EVAL`").

For example, consider a simple upgradable vault: each UTXO may be spent if an previously-approved condition is revealed and satistfied – much like [Taproot](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki), but without the limitation that conditions must be specified prior to UTXO creation. This configuration allows unused spending paths to remain hidden (via P2SH), improving privacy and/or operational security and reducing the byte length of some transactions<sup>1</sup>.

Today, this kind of upgradable vault may be created by locking funds in UTXOs which inspect a sibling input for authorized tokens (e.g. `<2> OP_UTXOTOKENCATEGORY <EXPECTED_CATEGORY> OP_EQUALVERIFY`), delegating control to the contract(s) which currently hold those tokens<sup>2</sup>. Authorized tokens can then be distributed across multiple contracts, participate in other contract systems, and/or be moved independently from the locked funds.

#### Improved Privacy and Operational Security vs. Status Quo

While the status quo allows unused spending paths for these vaults to remain hidden, the existence of other spending conditions and related metadata (e.g. the total number of conditions, timing and frequency of past activity, the contents of previously-revealed contracts controlling identity tokens, etc.) are revealed by any vault spend<sup>3</sup>. Because this proposal would allow direct delegation within a contract, vaults can be designed to avoid leaking such details, improving overall privacy and operational security.

#### Improved Efficiency and User Experience vs. Status Quo

This proposal also reduces transaction sizes vs. the status quo, especially for spends of [P2S](https://github.com/bitjson/bch-p2s) contracts. As emulating `OP_EVAL` currently requires a sidecar input, this proposal saves at least 41 bytes per use vs. the status quo<sup>4</sup>. Within P2S contracts, this proposal allows contracts to hide unused code paths in the same and similar ways as is currently available via P2SH; for these cases, this proposal also avoids the need for an intermediate transaction (spending an intermediate P2SH contract) to emulate this behavior. In practice, this both reduces the operational complexity of contracts – an atomic transaction avoids the need for additional network monitoring, timing management, and error handling (e.g. partial block inclusion) – and reduces cumulative transaction bandwidth by at least 65 bytes per occurance<sup>5</sup>.

#### Notes

1. Note that identity tokens are also more powerful than Taproot or signature-based delegation in that identity tokens also enable **revocation** of previously-approved spending conditions. E.g. if an employee leaves a company, their keys may be revoked from thousands of different UTXOs without moving (or even revealing the redeem bytecode of) those UTXOs; instead, only the identity token must be moved to a new contract in which the revoked key is not a participant. (The new identity token contract can likewise delegate and/or hide unused spending conditions via sidecar inputs or `OP_EVAL`.)
2. To prevent P2SH UTXOs within such vaults from being associated prior to spending (for privacy and/or operation security), a nonce may be included in the redeem bytecode (e.g. `<NONCE> OP_DROP`).
3. If their redeem bytecode includes a nonce, unspent UTXOs in the same vault may still remain unassociated, hiding the "current balance" of the vault.
4. The additional transaction input adds [a minimum of 41 bytes](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#selection-of-input-length-formula). This proposal may enable additional efficiencies in specific cases, but for simplest comparison, we assume both strategies would authenticate the evaluated bytecode in the same way, e.g. comparison against a 32-byte hash, validation of a 65-byte signature, etc.
5. The minimum byte length of the intermediate transaction.

### BIP12 `OP_EVAL`

[BIP12 `OP_EVAL`](https://github.com/bitcoin/bips/blob/master/bip-0012.mediawiki) was proposed in 2011, and shared the basic behavior of this proposal: pop and evaluate the top stack item. BIP12 differred from this proposal in several ways:

- **Didn't preserve stack frames in the control stack**:
  - BIP12 didn't specify interation with other control flow behavior (at the time, `OP_IF/OP_NOTIF/OP_ELSE/OP_ENDIF`); depending on the implementation, this would allow control flow structures to extend across multiple evaluations.
  - This proposal instead enforces consistent boundaries around existing and future control flow structures (e.g. [loops](https://github.com/bitjson/bch-loops)), simplifying both consensus implementations and developer tooling.
- **Limited evaluation depth to 2**:
  - BIP12's nesting limit would hamper the overall usefullness of functions by preventing deeper composition, requiring additional duplication and/or stack manipulation of code in many cases.
  - While the [existing control stack depth limit](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#retention-of-control-stack-limit) of still 100 applies to this proposal, relatively few computations employ deeper nesting for optimal factoring.
- **Prevented use of `OP_CODESEPARATOR`**:
  - BIP12 required implementations to specifically scan each evaluated script for instances of `OP_CODESEPARATOR`, rejecting the transaction if found. This needlessly increases implementation complexity and eliminates some potential value: [`OP_CODESEPARATOR` remains a useful feature](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#ongoing-value-of-op_codeseparator-operation) for certain use cases and has been preserved by past upgrades (e.g. [`OP_ACTIVEBYTECODE`'s support for `OP_CODESEPARATOR`](https://github.com/bitjson/bch-2022/blob/master/CHIP-2021-02-Add-Native-Introspection-Opcodes.md#op_activebytecode-support-for-op_codeseparator)).
  - This proposal avoids the additional implementation complexity and preserves the usefulness of `OP_CODESEPARATOR` within evaluations.
- **Cleared the alternate stack**:
  - Though not specified, a BIP12 implementation also [cleared the alternate stack](https://github.com/bitcoin/bitcoin/issues/729#issuecomment-3294453) at the end of each evaluation. At best, this behavior is useless: because each alternate stack item requires one byte to push (via `OP_TOALTSTACK`), any program which employed the stack clearing behavior could just as easily have replaced these bytes with equally or more efficient dropping operations (`OP_DROP`, `OP_2DROP`, or wider [stack clearing](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#increased-usability-of-multisig-stack-clearing)). Further, preventing functions from accepting or returning values in the alternate stack would limit their byte efficiency by requiring less efficient stack juggling strategies vs. the higher semantic bandwidth of passing both stacks.
  - This proposal does not implicitly clear the alternate stack after each evaluation.

Overall, this proposal benefits from many years of additional information, research, and clarification of constraints vs. BIP12, which had little visibility into yet-to-be-developed use cases, today's wider contract ecosystem, future introspection operations (especially `OP_ACTIVEBYTECODE`), [CashTokens](https://cashtokens.org/), or the [modern VM limits](https://github.com/bitjson/bch-vm-limits).

### Output-Level Function Annex

Another potential alternative to this proposal is the addition of a per-output function annex, perhaps using a "`PREFIX_FUNCTIONS`" encoding and deployment strategy similar to the [token-encoding `PREFIX_TOKEN`](https://cashtokens.org/docs/spec/chip#token-prefix) codepoint, coupled with an "`OP_CALL`" operation (this could share the `PREFIX_FUNCTIONS` codepoint). This approach would:

- **Disallow hiding of unused spending paths** – By forcing the contents of all function definitions to be exposed at spending time, this approach would limit the proposal's [privacy and operational security benefits](#improved-privacy-and-operational-security-vs-status-quo).

- **Prevent some cross-input deduplication** – By forcing P2SH inputs to duplicate every function definition, this approach would prevent some contract systems from minimizing transaction sizes, e.g. by evaluating bytecode from a [sidecar input](#status-quo-sidecar-inputs-emulated-op_eval) (authenticated via a hash, signature, or token category and/or commitment).

- **Increase implementation complexity** – By requiring a new data structure and assiciated validation rules for specifying function code, this approach would increase the complexity of validating implementations. (What are the length and function count limits? Must all functions be called over the course of evaluation, or can functions be defined and never called? May functions call themselves recursively? Does all VM state carry into function calls? How do `OP_ACTIVEBYTECODE` or `OP_CODESEPARATOR` behave? Etc.)

- **Slightly increase some contract lengths** – By limiting the ability for transaction compilers to optimize function selection bytecode, this approach would increase the byte-length of contracts, especially for the last call of every function. In practice, this might add one additional byte per function call for contracts with more than 18 functions (assuming `OP_1NEGATE` and `OP_0`-`OP_16` are valid selectors), and one byte for the last call of some functions (e.g. `OP_7 OP_CALL` rather than a single `OP_EVAL`).

Overall, this proposal considers `OP_EVAL` to be more capable, efficient, and useful in the long term, minimizing protocol debt.

### Word Definition Operations (`OP_DEFINE` & `OP_INVOKE`)

Another potential alternative to this proposal is a [set of (Forth) word definition operations](https://bitcoincashresearch.org/t/chip-2024-12-op-eval-function-evaluation/1450/15?u=bitjson), e.g. `OP_DEFINE` and `OP_INVOKE`, where `OP_DEFINE` accepts a numeric identifier and function body, and `OP_INVOKE` accepts a numeric function identifier to execute. The `OP_DEFINE/OP_INVOKE` approach would be functionally equivalent to this proposals `OP_EVAL`, but with [increased protocol complexity and increased contract lengths](https://bitcoincashresearch.org/t/chip-2024-12-op-eval-function-evaluation/1450/15?u=bitjson).

### OP_EXEC

Another potential alternative to this proposal is [`OP_EXEC`](https://web.archive.org/web/20250117180758/https://www.nextchain.cash/op_exec.md), a variation on `OP_EVAL` which attempts to improve contract security by defining additional limitations around stack usage ("stack isolation") and recursion (to a depth of 3).

The `OP_EXEC` proposal misunderstands the relevance of "stack isolation" to contract security<sup>1</sup>, has no known or theoretical advantages vs. `OP_EVAL`, and even if optimized to be less wastful per-invocation, would significantly increase the byte length of all contracts vs. `OP_EVAL`<sup>2</sup>.

<small>

1. See [OP_EXEC misunderstands contract security and usage](https://bitcoincashresearch.org/t/chip-2024-12-op-eval-function-evaluation/1450/20?u=bitjson).
2. See [Comparison of OP_EVAL and OP_EXEC](https://bitcoincashresearch.org/t/chip-2024-12-op-eval-function-evaluation/1450/14?u=bitjson).

</small>
