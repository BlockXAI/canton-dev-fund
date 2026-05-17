## Development Fund Proposal: GrowStreams — Privacy-Native Payment Streaming for Canton

- **Author:** Satyam Singhal, BlockX AI Ltd
- **Champion:** srikanth-bitdynamics (BitDynamics)
- **Label:** financial-workflows-composability
- **Contact:** info@blockxai.xyz
- **Status:** Submitted
- **Created:** 2026-05-16
- **Repository:** https://github.com/BlockX-AI/Canton_Streams_RewardApp
- **Live product:** https://growstreams.xyz
- **Canton campaign:** https://www.cctools.network/earn/growstreams-x-cctools

---

## Abstract

This proposal requests funding for GrowStreams, an open-source payment streaming primitive for Canton that makes continuous financial obligations flow the way they actually accrue, instead of settling in monthly batches.

Payment streaming has proven its value on other networks. Superfluid has processed over a billion dollars. Sablier has crossed a billion in cumulative volume. But both operate on public chains where every stream rate, balance, and counterparty is visible to anyone. That makes them structurally unusable for the institutional workflows Canton is designed for.

GrowStreams is the same primitive built natively for Canton's privacy model. Stream terms, rates, and balances are visible only to authorized parties. Settlement is enforced on-ledger through Daml templates. The protocol targets CIP-56 token interfaces for asset-agnostic settlement, with Canton Coin and USDCx as the initial reference paths.

This proposal is not speculative. GrowStreams is live, tested, and has demonstrated real demand from the Canton community. Before filing, we engaged directly with Melvis (Executive Director, Canton Foundation) and W. Eric Saranlecki (Canton co-founder). Their specific guidance is reflected in the milestone structure below.

The strongest Canton-native use cases we are targeting:

- LP incentive programs and market-maker retainers
- node operator and validator service billing
- vesting, grants, and private-round unlock schedules
- consortium revenue distribution
- USDCx repo interest and custody fee accrual
- AI agent task payments on Canton

---

## Motivation

Canton teams that want time-based payment flows today have no shared primitive to build from. They either write their own streaming logic from scratch, batch payouts periodically, or rely on manual transfers. Every team solving this independently creates duplicated effort around the same correctness concerns: deterministic accrual over time, withdrawal and settlement correctness, privacy of payment terms, cancellation and refund handling, and reusable SDK surfaces.

We started building GrowStreams on Vara Network in July 2025 to prove the model worked. We ported it to Canton because the participants we are building for are here. Goldman Sachs, DTCC, Euroclear, Broadridge — these organizations are already on Canton. The streaming primitive that serves their continuous obligations does not exist yet.

---

## Objective

The objective is to make payment streaming available as reusable open-source infrastructure on Canton, so Canton teams do not need to build the core on-ledger logic, SDK, and reference integration surfaces from scratch.

The intended outcome is that a Canton developer or application operator can:

- run a fixed-duration LP incentive campaign with on-ledger enforcement and privacy
- stream validator or infrastructure service payments continuously between parties
- vest token allocations for grants, launch participants, or contributors with cliff enforcement
- create treasury-managed recurring payment schedules in CC or USDCx
- distribute consortium revenue proportionally to multiple parties in real time
- enable AI agents to receive and send payments on Canton without human authorization

And the system will:

- enforce all streaming rules on-ledger through Daml templates
- preserve privacy of stream terms, rates, and balances between parties
- support both prefunded guaranteed-settlement streams and non-prefunded rolling top-up streams
- expose clean SDK and CIP-103 dApp API surfaces for integrators
- provide a complete audit trail to authorized parties without public state exposure
- target CIP-56 V1 and V2 token interfaces for asset-agnostic settlement

---

## Current Demand and Design-Partner Validation

This proposal is not based only on theoretical demand.

Current named design-partner signals include:

- **CCTools** — LP incentive streaming, contributor reward distribution, and recurring campaign payout flows. Joint Canton launch campaign generated 4,300+ participants in under 24 hours with 3,661 verified on-chain task completions. CCTools publicly confirmed: "Yesterday's GrowStreams Canton launch brought 4,300+ participants. This is what real discovery looks like."

- **Ginie (BlockX AI)** — usage-based billing between Canton parties for AI-powered smart contract generation. Metered API access flows requiring continuous settlement as contracts are generated and deployed.

- **Web3Cash** — continuous reward streaming to contributors as they earn, replacing batch payouts at period end. Campaign owners depositing real USDC need streaming rails for real-time contributor reward distribution on Canton.

We are actively reaching out to additional Canton ecosystem projects to collect on-PR validation comments from teams that want to use GrowStreams for specific workflows. We welcome any Canton project with a concrete use case to comment on this PR with their workflow needs.

These are not abstract examples. They map directly to live workflow needs in the current Canton ecosystem, and the revised milestones are tied to partner validation and external usage rather than only implementation completeness.

---

## Canton-Native Use Cases

Per guidance from the Canton Foundation, we lead with hourly and daily streams as the realistic improvement over the status quo for institutions currently on Canton. The per-second precision is the protocol's capability ceiling, not the primary institutional pitch.

**LP incentives and market-maker retainers**

Canton DeFi protocols run periodic batch reward distributions to liquidity providers. With GrowStreams, LP rewards flow continuously proportional to position size. Treasury-funded, on-ledger enforced, private. No manual distribution at period end.

**Node operator and validator billing**

Super Validators charge participants for infrastructure that runs continuously. Today this is a monthly invoice with 30 days of credit risk on both sides. With GrowStreams it becomes a daily stream with a credit cap. Auto-pause when the cap is reached. No invoice. No reconciliation.

**Vesting, grants, and unlock schedules**

Token distributions for contributors, grants, launch participants, and private rounds with cliff enforcement on-ledger. The VestingWithdraw choice assertFails before the cliff date. No admin can override this. The Daml type system enforces it.

**Consortium revenue distribution**

Multiple institutions sharing a Canton application today split revenue quarterly via spreadsheet. With the Split Router, proportional distribution happens continuously. On-chain math is the agreement. No back-office dispute is possible.

**USDCx repo interest and custody fee accrual**

USDCx launched on Canton in December 2025. Repo interest accrues every second. Settlement is currently scheduled and periodic. GrowStreams makes those interest flows continuous between counterparties with full privacy on negotiated rates.

**AI agent payments**

Canton's Director of Institutional Sales posted publicly in April 2026: "The agent economy has officially started on Canton. With privacy." AI agents cannot hold bank accounts. They cannot wait for invoices. A streaming primitive that triggers on task completion and settles in USDCx privately on Canton is the natural payment layer for what Canton is building toward.

---

## Standards Alignment

- **CIP-56 V1 and V2:** the SettlementAdapter sits behind a narrow token boundary targeting CIP-56 token interfaces. Stream engine logic stays asset-agnostic. CIP-56 V2 support delivered alongside or shortly after V2 ratification.
- **CIP-103 dApp API:** all lifecycle operations (Create, Withdraw, Pause, Resume, Cancel, MutualCancel, Renew, TopUp, Clip, Complete) exposed through CIP-103 JSON-RPC surface. Any CIP-103 compliant wallet can authorize stream operations without bespoke integration.
- **Wallet SDK:** integrates with @canton-network/wallet-sdk as preferred ledger backend.
- **Mainnet commitment:** active streams on Canton Mainnet are required acceptance criteria for M3.

---

## Implementation

### How accrual state is stored and why there is no settlement risk

This question deserves a direct answer because it is the most important technical concern about streaming primitives.

GrowStreams does not write to the ledger every second. Instead, the StreamAgreement contract stores the sender, receiver, rate, total deposited, total withdrawn, and the timestamp of the last settlement. These fields are always on-chain. Nothing is stored off-chain.

The accrual formula is:

```
Accrued = (Ledger Time - Last Settled) x Rate
```

When a party calls ObservationView, the contract computes the accrued amount deterministically from on-chain state using Canton's Ledger Time. This is a non-consuming choice — it reads but does not write, so there is no transaction fee. The result is deterministic: any party with access can compute the same number independently from the same on-chain state.

When the receiver calls Withdraw, one transaction runs the formula at that exact Ledger Time moment, transfers the exact accrued amount, and updates the Last Settled timestamp on-chain. The contract enforces:

```
totalWithdrawn <= accrued(now) <= totalDeposited
```

This invariant is enforced on-ledger by the Daml template. A receiver can never withdraw more than has actually accrued. A sender's deposit is never at risk beyond what they committed. There is no off-chain settlement engine, no oracle, and no trusted third party. The math and the state are both on Canton.

A 30-day stream generates approximately three to four on-chain transactions: one to open, one or more withdrawals, and one to close. The precision is in the deterministic formula, not the transaction count.

### Supported assets

Today the protocol is implemented and tested with the GROW token (a CIP-56 compatible fungible token). The SettlementAdapter is designed to be asset-agnostic by targeting CIP-56 token interfaces. Canton Coin and USDCx are the M1 and M2 integration paths respectively. Any CIP-56 compliant token can be supported without changes to the core streaming logic.

### Two streaming models

**Prefunded streams**

The sender locks a balance into escrow at creation. The contract tracks deposited, withdrawn, and accrued amounts. Each Withdraw transfers only what has accrued and not yet been withdrawn. Each Cancel or Complete settles the accrued amount and returns the remainder. No stream can promise more than the funded balance.

This removes credit risk, liquidation mechanics, and insolvency handling from the design. It is suited to LP incentive campaigns, vesting schedules, grants, milestone escrow, and fixed-term retainers.

**Non-prefunded rolling top-up streams**

For open-ended recurring flows, the sender maintains funding through top-ups without locking the full lifetime amount. Withdrawals remain bounded by what has been funded and accrued. This improves capital efficiency for subscriptions, recurring infrastructure billing, and ongoing service retainers.

### Core contracts

**StreamAgreement** — core Daml template. Sender, receiver, optional observers, token reference, deposited amount, withdrawn amount, last settled timestamp, rate, stream mode, status.

**StreamGroup** — groups related streams for batch workflows such as LP incentive campaigns or multi-recipient payroll distributions.

### Stream types

- Linear
- CliffLinear
- Stepped
- RenewableTerm

### Choice model

- Create
- Withdraw
- Pause
- Resume
- Cancel
- MutualCancel
- Renew
- TopUp
- Clip
- Complete

### Privacy and authorization

Sender creates and funds the stream and can cancel where permitted. Receiver can withdraw accrued amounts. Observers can view status for treasury, compliance, or audit purposes.

Stream terms, rates, and balances are visible only to signatories and observers. No global public state. Full auditability available to authorized parties without exposure to external participants.

### Core invariants

- 0 <= totalWithdrawn <= accrued(now) <= totalFunded
- alreadyWithdrawn + withdrawable + refundable = totalFunded
- accrued amount is monotonic in Ledger Time
- no negative remaining balances
- once Cancelled or Completed, no further withdrawal is possible
- non-prefunded flows cannot pay more than the actually funded balance

### SDK, dashboard, and wallet integration

- TypeScript SDK published to npm as @growstreams/sdk
- reference React dashboard
- CIP-103 dApp API bindings for wallet-driven authorization
- integration examples and documentation

---

## Proven technical state

66 consecutive test passes across 6 use cases with zero errors in 104 seconds of continuous operation on Canton 3.4.11 LocalNet. Real Daml contracts. Real GROW token movements between four parties.

**Payroll streaming** — 1 GROW per second, Alice to Bob. Bob withdrew 120.634 GROW in cycle 1, approximately 10 GROW in each subsequent 10-second cycle. Mathematically exact across 11 cycles.

**LP reward distribution** — 10 GROW per second pool, 70% to Alice and 30% to Carol. Ratio exact across all 11 cycles.

**Institutional billing** — 0.5 GROW per second per session, pause and resume per session. GrowToken created atomically in the same transaction as the Pause choice.

**Token vesting** — 12,000 GROW with a cliff on May 4, 2026. VestingWithdraw assertFails before the cliff across all 11 test cycles.

**SaaS subscription** — 1 GROW per second, delta approximately 10 GROW per 10-second interval across 11 cycles. Zero rounding errors.

**Milestone escrow** — Admin confirms deliverable, GROW transfers and GrowToken is created in the same Canton transaction. DVP. Atomic. Private.

Terminal output from this test run is in the /evidence/ folder of the repository. Any committee member can reproduce these results.

---

## Validation from comparable ecosystems

Sablier demonstrates that prefunded streaming can reach product-market fit. Over 1 billion dollars in cumulative volume, strong adoption in vesting and committed payment programs.

Superfluid demonstrates the non-prefunded model at scale. Over 1.25 billion dollars streamed to more than one million recipients across 850 projects.

Neither protocol could serve Canton's institutional use cases because their stream rates are public. Canton's sub-transaction privacy is what makes streaming viable for the participants already on this network.

---

## Live demand evidence

**CCTools campaign — May 2026**

- 4,300+ participants in under 24 hours
- 2,500 invite codes claimed same day, round 2 launched immediately
- 12,950 campaign views
- 3,661 verified participants with completed tasks

**GrowStreams ecosystem directory on CCTools**

- 7,233 upvotes
- 8,905 views
- Score 66/100

**growstreams.xyz platform**

- 1,317 registered users
- 3,078 quest completions
- 398,940 XP minted on-chain
- 24 active quests, 5 active campaigns

**X / Twitter**

- 4,519 followers
- +2,900 gained in a single day from the Canton community response

---

## Milestones

Per guidance from the Canton Foundation: demand is demonstrated upfront and the bulk of funding is tied to Mainnet adoption metrics. 15% of development funding is released at M1. 85% requires active streams on Canton Mainnet.

---

### Milestone 1 — TestNet deployment, SDK, and CCTools integration

**60,000 CC**

Deliverables:

- Canton TestNet deployment with public contract ID verifiable on Canton explorer
- daml test output showing all 6 use cases passing, stored in /evidence/ folder
- TypeScript SDK v0.1.0 published to npm as @growstreams/sdk
- CIP-56 V1 token standard conformance — Canton Coin reference flow
- CIP-103 dApp API bindings for all lifecycle operations
- developer quickstart documentation at growstreams.xyz/docs
- CCTools integration live with verifiable on-chain activity

Acceptance criteria:

- Canton TestNet contract address publicly accessible and verifiable
- npm install @growstreams/sdk resolves and connects to TestNet
- daml test output in /evidence/ showing all tests passing with timestamp
- on-chain activity from CCTools integration verifiable by template ID

---

### Milestone 2 — Mainnet deployment and security audit

**80,000 CC**

Deliverables:

- independent security audit by a qualified Daml and Canton reviewer, report published publicly
- all Critical and High findings remediated before Mainnet deployment
- Canton Mainnet deployment with contract ID verifiable on Canton explorer
- SDK v1.0 with CIP-56 V2 support and USDCx reference integration
- React reference dashboard and proxy
- non-prefunded rolling top-up path implemented and documented
- 3 external Canton dApp integrations live on TestNet

Acceptance criteria:

- audit report published with zero unresolved Critical or High findings
- Mainnet contract address verifiable on Canton block explorer
- 3 named Canton projects with verifiable on-chain activity by template ID
- SDK v1.0 installable from npm and functional against Mainnet

---

### Milestone 3 — Mainnet adoption

**230,000 CC**

Nothing in this milestone is paid for code delivery. Everything is paid for demonstrated usage by external parties on Mainnet.

Deliverables:

- 10 or more Canton dApp integrations active on Mainnet, verifiable by template ID
- 100 or more active streams on Mainnet over a rolling 30-day window
- 500,000 CC burned in transaction fees through GrowStreams template IDs, grantee excluded
- one institutional pilot with documented on-chain evidence
- Apache 2.0 release of full Daml package, TypeScript SDK, reference dashboard, and documentation
- final adoption report to the committee

Acceptance criteria:

- 10 or more integrations with on-chain transaction history verifiable by template ID
- 100 or more streams active over a rolling 30-day window verifiable on Canton explorer
- 500,000 CC burn threshold crossed, grantee excluded, verifiable on Canton explorer
- institutional pilot documented with on-chain evidence
- full Apache 2.0 release published on GitHub

---

### Milestone 4 — Performance-based adoption bonus

**Up to 500,000 CC**

Disbursed in 100,000 CC increments as on-chain CC burn through GrowStreams template IDs crosses each 200,000 CC threshold. Grantee excluded. 12-month window from M3 delivery. Capped at 500,000 CC total.

Acceptance criteria per tranche:

- cumulative CC burn through GrowStreams template IDs exceeds the next 200,000 CC threshold
- grantee excluded from qualifying burn count
- 12-month window from M3 delivery has not elapsed
- cap of 500,000 CC total bonus has not been reached

Deliverables:

- canonical template-ID manifest published at M1 as the binding artifact for bonus computation
- public adoption metric report at each tranche claim

---

## Funding summary

| Milestone | CC | What releases it |
|---|---|---|
| M1: TestNet, SDK, CCTools live | 60,000 CC | Code delivery and integration on-chain |
| M2: Mainnet, audit | 80,000 CC | Audit passed, Mainnet live, 3 integrations |
| M3: Mainnet adoption | 230,000 CC | 10 integrations, 100 streams, 500K CC burn |
| M4: Performance bonus | up to 500,000 CC | 200K CC burn per 100K CC tranche |
| **Total** | **370,000 CC + up to 500,000 CC** | |

---

## Acceptance criteria

Every criterion below is verifiable on-chain or via public sources without trusting the team.

- Daml templates implementing all promised stream types and lifecycle choices
- prefunded settlement and refund behavior working for CC and USDCx
- non-prefunded rolling top-up path implemented and documented
- SDK available on npm for all lifecycle flows
- CIP-103 dApp API integration covering all stream lifecycle operations
- CIP-56 V1 and V2 token standard conformance
- dashboard and reference proxy available for stream creation, monitoring, and withdrawal
- privacy and authorization behavior documented and enforced on-ledger
- local demo scripts covering all core flows
- Mainnet deployment with active streams
- 10 or more Canton dApp integrations in production on Mainnet with 100 or more active streams over a rolling 30-day window
- 500,000 CC burned through GrowStreams template IDs by external parties
- independent audit and remediation of Critical and High findings
- maintenance and support window committed
- release artifacts published under Apache 2.0

Project-specific conditions:

- GrowStreams remains a streaming primitive, not a billing platform or DeFi protocol
- stream state and settlement rules remain on-ledger in Daml templates
- stream privacy is preserved, no global public exposure of terms or balances
- prefunded and non-prefunded models are clearly distinguished and documented

---

## Security and maintenance

The security process includes a documented threat model, adversarial and edge-case testing against the core invariants, independent audit by a qualified Daml and Canton reviewer, remediation of all Critical and High findings before Mainnet positioning, and a defined maintenance window.

---

## Team

BlockX AI Ltd is a UK registered company (Companies House: 16254630).

Satyam Singhal — founder. Designed the Obligation-First accrual architecture. Led Vara Network deployment with 7 contracts and 53 tests passing. Built and ran the 21-day live campaign with AI-verified per-second streaming payouts. Leading the Canton build.

Engineering team — smart contract engineers across Rust and Daml, production REST API with dual-mode signing, TypeScript SDK, and AI contribution scoring agent.

Community and content team — built 4,519 X followers organically without paid promotion. Coordinated the CCTools integration that generated 4,300+ participants in a single day.

---

## Sustainability

Protocol revenue model: 2.5% fee on settled streams. All Daml packages, TypeScript SDK, reference dashboard, and documentation released under Apache 2.0.

---

## Links

| Resource | URL |
|---|---|
| Repository | https://github.com/BlockX-AI/Canton_Streams_RewardApp |
| Live product | https://growstreams.xyz |
| Canton campaign | https://www.cctools.network/earn/growstreams-x-cctools |
| X / Twitter | https://x.com/GrowwStreams |
| BlockX AI Ltd | Companies House 16254630 |
| Contact | agrim@growstreams.xyz |

---

## Checklist

- [x] Proposal file added under /proposals/
- [x] Milestones and funding amounts defined
- [x] Acceptance criteria included, all verifiable on-chain without trusting the team
- [x] Alignment with Canton priorities described
- [x] Demand demonstrated with live verifiable numbers
- [x] Bulk of funding tied to Mainnet adoption milestones
- [x] Performance bonus tied to CC burn
- [x] CIP-56 V1 and V2 alignment confirmed
- [x] CIP-103 dApp API integration confirmed
- [x] Mainnet commitment confirmed
- [x] Security audit included in M2
- [x] Maintenance window committed
- [x] Apache 2.0 open source commitment confirmed
