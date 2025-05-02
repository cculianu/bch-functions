# Risk Assessment

The following security considerations, potential risks, and costs have been reviewed to verify the safety and advisability of this proposal.

<details>

<summary><strong>Table of Contents</strong></summary>

- [Risk Assessment](#risk-assessment)
  - [Risks \& Security Considerations](#risks--security-considerations)
    - [User Impact Risks](#user-impact-risks)
      - [Reduced or Equivalent Node Validation Costs](#reduced-or-equivalent-node-validation-costs)
      - [Increased or Equivalent Contract Capabilities](#increased-or-equivalent-contract-capabilities)
    - [Consensus Risks](#consensus-risks)
      - [Full-Transaction Test Vectors](#full-transaction-test-vectors)
      - [Additional Performance Benchmarks](#additional-performance-benchmarks)
      - [`Chipnet` Preview Activation](#chipnet-preview-activation)
    - [Denial-of-Service (DoS) Risks](#denial-of-service-dos-risks)
      - [Node Performance Safety Margin](#node-performance-safety-margin)
    - [Protocol Complexity Risks](#protocol-complexity-risks)
      - [Support for Post-Activation Simplification](#support-for-post-activation-simplification)
      - [Evaluation of Alternatives](#evaluation-of-alternatives)
  - [Upgrade Costs](#upgrade-costs)
    - [Node Upgrade Costs](#node-upgrade-costs)
    - [Ecosystem Upgrade Costs](#ecosystem-upgrade-costs)
  - [Maintenance Costs](#maintenance-costs)
    - [Node Maintenance Costs](#node-maintenance-costs)
    - [Ecosystem Maintenance Costs](#ecosystem-maintenance-costs)

</details>

## Risks & Security Considerations

This section reviews the foreseeable security implications of the proposed changes to the Bitcoin Cash network. Key technical considerations include user impact risks, consensus risks, denial-of-service (DoS) risks, and risks to protocol complexity or maintenance burden of newly introduced behavior.

### User Impact Risks

All upgrade proposals must carefully analyze proposed changes for potential impacts to existing Bitcoin Cash users and use cases. Virtual Machine (VM) upgrades can impact node operators and blockchain indexers (and therefore payment processors, exchanges, and other businesses), software development libraries, wallets, decentralized applications, and a wide range of pre-signed transactions, contract systems, and transaction-settled protocols.

This proposal is designed to preserve backwards-compatibility along a variety of dimensions, minimizing user impact risks:

#### Reduced or Equivalent Node Validation Costs

By avoiding both the introduction of new data structures for function definitions and avoiding the addition of other new operations (`OP_EVAL` can only evaluate pre-existing operations), this proposal minimizes the risk of increase to worst-case validation performance, preventing any increase in node operation costs.

Critically, the addition of new control flow operations – like `OP_EVAL` – was accounted for in the development of [CHIP-2021-05 VM Limits](https://github.com/bitjson/bch-vm-limits). From [VM Limits Risk Assesement: Consideration of Possible Future Changes](https://github.com/bitjson/bch-vm-limits/blob/master/risk-assessment.md#consideration-of-possible-future-changes):

> - **Compatibility with additional control flow structures** – By limiting the density of pushed bytes and the density of evaluated operations, this proposal's re-targeted limits remain effective even if future upgrades significantly expand the VMs control flow capabilities: definite or indefinite loops, pushed bytecode evaluation, word/function definition and evaluation, or other structures. Further, the worst-case performance of contracts using such control flow structures can be safely decoupled from the contract length limit, allowing greater flexibility, e.g. loops which safely evaluate more instructions than would otherwise be possible to fit within contract length limits. See [Rationale: Retention of Control Stack Limit](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#retention-of-control-stack-limit), [Rationale: Limitation of Pushed Bytes](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#limitation-of-pushed-bytes), and [Rationale: Selection of Base Instruction Cost](https://github.com/bitjson/bch-vm-limits/blob/master/rationale.md#selection-of-base-instruction-cost).

Further, this risk mitigation strategy has been empirically verified across multiple implementations using a wide range of [cross-implementation functional tests and performance benchmarks](./vmb_tests/).

#### Increased or Equivalent Contract Capabilities

Because this proposal adds the `OP_EVAL` operation without modifying the behavior of any other operations, existing contract systems, decentralized applications, pre-signed transactions, and transaction-settled protocols are not impacted. Upgrades are only required for applications wishing to take advantage of new privacy, auditability, or fee efficiency capabilities.

### Consensus Risks

All network consensus upgrade proposals must account for consensus risks arising from incorrect or inconsistent implementation of consensus-critical changes. For Virtual Machine (VM) upgrades, consensus risks primarily apply to node implementations and other software which performs VM evaluation as part of transaction validation.

This proposal mitigates consensus risks via 1) an extensive set of full-transaction test vectors, 2) a new cross-implementation performance testing methodology, and 3) a 6-month early activation on `chipnet`.

#### Full-Transaction Test Vectors

To minimize the risk of inconsistencies between implementations, this proposal includes [over 30,000 cross-implementation functional tests and performance benchmarks](./vmb_tests/) covering both the contents of the upgrade and of significant portions of pre-existing VM behavior.

In developing these test vectors, a workflow for test vector development has been established between [Libauth](https://github.com/bitauth/libauth) (a debugging-focused JavaScript implementation) and [Bitcoin Cash Node](https://gitlab.com/bitcoin-cash-node/bitcoin-cash-node/) (C++) – the implementation most commonly relied on by Bitcoin Cash miners. In effect, this development brings the added reliability of [N-version programming](https://en.wikipedia.org/wiki/N-version_programming) to Bitcoin Cash's VM test vector development.

#### Additional Performance Benchmarks

This proposal adds [additional performance benchmarks](./vmb_tests/) thoroughly covering `OP_EVAL` usage to help detect performance irregularities and regressions in specific node implementations and VM-evaluating software.

#### `Chipnet` Preview Activation

Finally, like all Bitcoin Cash Improvement Proposals (CHIPs), this proposal schedules activation on `chipnet` 6-months before activation on `mainnet`.

While many development teams and ecosystem stakeholders have reviewed this proposal prior to the November lock-in, only a smaller subset of highly-interested projects and parties (e.g. contract developers, decentralized application development teams, and node implementations) can speculatively allocate development time for complete integration testing with existing software systems.

By scheduling a 6-month early activation on `chipnet`, this proposal minimizes the cost of widest-possible integration testing – in a partially adversarial environment – in which software defects can be discovered and corrected prior to `mainnet` activation.

### Denial-of-Service (DoS) Risks

All network consensus upgrade proposals which alter VM behavior carry risks related to Denial-of-Service (DoS) attacks. In particular, modifications to VM behavior could 1) exacerbate the worst-case performance of transaction or block validation for both expensive-but-valid cases and excessively-invalid cases, and/or 2) decrease the cost or increase the practicality of attempting a particular VM-related DOS attack.

#### Node Performance Safety Margin

This proposal maintains Bitcoin Cash's existing, `10x` to `100x` margin of safety for transaction and block validation performance. See [Limits CHIP Risk Assessment: Expanded Node Performance Safety Margin](https://github.com/bitjson/bch-vm-limits/blob/master/risk-assessment.md#expanded-node-performance-safety-margin).

### Protocol Complexity Risks

All upgrade proposals must carefully analyze proposed changes for both immediate and potential future impacts on overall protocol complexity. This proposal has been reviewed to ensure that all changes are 1) minimal, 2) necessary, and 3) avoid creating technical debt, even if future upgrades were to further expand the VM's capabilities.

#### Support for Post-Activation Simplification

This proposal allows for backwards-compatible implementation. As a result, implementations may choose to remove activation code following activation.

#### Evaluation of Alternatives

This proposal includes a thorough [Evaluation of Alternatives](./alternatives.md) assessing each option's impact on protocol complexity.

## Upgrade Costs

This section reviews the costs of implementing the proposed changes.

### Node Upgrade Costs

This proposal affects consensus – all fully-validating node software must implement these Virtual Machine (VM) changes to remain in consensus. To minimize the cost of validating node implementation upgrades, this proposal includes a wide range of [cross-implementation functional tests and performance benchmarks](./vmb_tests/).

### Ecosystem Upgrade Costs

Upgrade costs for the wider ecosystem – wallets, blockchain indexers, mining, mining-monitoring, and mining-administrative software, etc. are limited:

- This proposal **does not create new user-facing primitives** (e.g. [CashTokens, 2023](https://cashtokens.org/)) which might demand significant changes or additional features in wallets, block explorers, or other ecosystem software.

- This proposal **does not notably impact user or miner-visible metrics** (e.g. [Adaptive BlockSize Limit Algorithm, 2024](https://gitlab.com/0353F40E/ebaa#chip-2023-04-adaptive-blocksize-limit-algorithm-for-bitcoin-cash)) which might demand significant changes in software like block explorers, mining calculators, and proprietary mining software and tooling, as well as user-facing documentation and marketing materials referencing the well-known, Blocksize Limit metric.

While all Bitcoin Cash software systems must inevitably be upgraded to follow all consensus upgrades (e.g. a company must annually upgrade to the latest version of their preferred full node implementation), ecosystem upgrade costs for this proposal are essentially confined to specialized software, tooling, and documentation for contract developers. Further, as the fundamental aim of this proposal is to expand the power and flexibility of Bitcoin Cash's VM for these stakeholders, significant portions of these costs were already speculatively paid, in hopes of later activation, during the creation and review of this proposal.

## Maintenance Costs

All network upgrade proposals must evaluate their foreseeable impact on maintenance costs. Virtual Machine (VM) upgrades can increase the risks of [consensus divergence](#consensus-risks), [performance issues](#denial-of-service-dos-risks), and increased [implementation complexity](#protocol-complexity-risks). This section reviews foreseeable ongoing costs following activation of the proposed changes.

### Node Maintenance Costs

This proposal minimizes any negative impact on node maintenance costs:

- **Equivalent worst-case validation cost** – By design, this proposal avoids any increase in worst-case validation cost. This reduces ongoing node maintenance costs by reducing the potential impact of implementation-specific performance issues.

- **New functional tests and performance benchmarks** – This proposal improves the long-term maintainability of implementations by contributing thorough [cross-implementation functional tests and performance benchmarks](./vmb_tests/) to help root out existing or future performance issues in specific implementations.

- **Potential increase in project maintenance resources** – By increasing the power and flexibility of Bitcoin Cash's VM for contract development, this proposal has the potential to attract further resources to node and VM implementations, both to 1) enhance performance of implementations primarily aimed at mining and/or business infrastructure, and 2) enhance the feature sets of alternative implementations aimed at contract development, decentralized application design, and/or specialized wallet management.

### Ecosystem Maintenance Costs

As with [ecosystem upgrade costs](#ecosystem-upgrade-costs), maintenance costs for the wider ecosystem – wallets, blockchain indexers, mining, mining-monitoring, and mining-administrative software, etc. are limited: this proposal **does not create new user-facing primitives**, and it **does not notably impact user or miner-visible metrics**. See [Ecosystem Upgrade Costs](#ecosystem-upgrade-costs).

Features of this proposal which are relevant to [node maintenance costs](#node-maintenance-costs) are also relevant to the rest of the ecosystem: 1) decreased or equivalent worst-case validation cost, 2) new functional tests and performance benchmarks, and a 3) potential increase in project maintenance resources.
