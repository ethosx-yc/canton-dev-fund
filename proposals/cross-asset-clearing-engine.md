# Institutional Cross-Asset Clearing Engine for Canton

**Contributor:** EthosX, Inc. (Delaware C-corp; Y Combinator Summer 2022 batch; backed by Y Combinator, Franklin Templeton, and strategic investors)
**Status:** External contributor; champion to be confirmed via SIG review
**Primary SIG:** Financial Workflows & Composability
**Secondary SIGs:** Token Standards / Asset Standards; DeFi Protocols & Liquidity
**Date:** June, 2026
**License (proposal text):** CC0-1.0
**License (software deliverables):** Apache-2.0

---

## 1. Summary

This proposal contributes an open-source cross-asset central counterparty (CCP) smart-contract layer to daml-finance: legal novation, multi-layer default waterfall, on-chain margin engine, and lifecycle and liquidation state machines suitable for institutional derivatives clearing on Canton.

The engine is asset-class-neutral by design. The first product instantiation delivered under this grant is option instruments (European and American, including digital and knock-out/knock-in variants). The second instantiation is interest-rate swap lifecycle delivered alongside an ISDA CDM adapter in the final milestone. The engine is architected for mapping to BIS/IOSCO Principles for Financial Market Infrastructures (PFMI Principles 4 through 14) so that regulated CCPs and institutional participants can adopt it under existing supervisory frameworks.

EthosX requests **$200,000 USD (approximately 1,290,000 CC at submission spot)** disbursed across four milestones over a nine-month period from July 15, 2026 to April 21, 2027.

---

## 2. Motivation

### What Canton has built

Over the last fifteen months Canton has built out collateral, risk, and settlement rails for institutional derivatives:

- QCP × Digital Asset bilateral CSA margining (January 2025)
- Nasdaq Calypso × QCP × Primrose × Digital Asset risk and margining integration (June 2025)
- DTCC UST Collateral Network pilot (September 2024)
- DTCC × Digital Asset partnership to tokenize DTC-custodied USTs via ComposerX (announced December 2025, MVP targeted for 2026)
- Live 24/7 on-chain UST financing executed on Tradeweb with BofA, Circle, Citadel Securities, Cumberland DRW, DTCC, Hidden Road, Société Générale, Tradeweb, and Virtu (August 2025)
- LayerZero went live on Canton as the first interoperability protocol (March 2026), enabling cross-chain asset and message transport with institutional testing involving Goldman Sachs, BNP Paribas, DRW, QCP, and Tradeweb
- JSCC, Mizuho, Nomura, and Digital Asset launched a JFSA-backed proof-of-concept on April 20, 2026 to bring JGB collateral mobility on-chain via Canton (running through approximately September 2026 under Japan's Payment Innovation Project), testing whether JGB rights transfers under the Book-Entry Transfer Act preserve legal status on-chain and whether collateral posting and substitution can move from business-hours to 24/7
- Tokenized collateral live today: USDC, USYC, tokenized USTs (with native JPMD issuance announced for phased 2026 rollout)
- ISDA CSA bilateral workflows in DAML and DAML Finance vanilla option templates

### What does not yet exist on Canton

A central counterparty smart-contract layer. Specifically: no on-chain legal novation, no segregated default fund, no multi-layer waterfall, no on-chain liquidation engine, no on-chain portfolio margining, and no on-chain margin-call state machine. QCP's work is bilateral CSA margining; a CCP is structurally different and sits above it. The JSCC PoC validates that an incumbent clearing house can bring its own collateral mobility on-chain via Canton, but the PoC does not deliver the smart-contract clearing logic that operates on posted collateral. EthosX's contribution is that complementary layer: novation, margin engine, default waterfall, liquidation state machine, the engine that runs once collateral is posted. The two layers compose. JGB-collateralized instruments cleared on Canton in the future could be margined and defaulted through the open-source primitives proposed here.

### Why this matters

Central counterparty clearing is the layer that processes more value than any other in capital markets. Globally, CCPs clear over $1 quadrillion of notional annually across rates, equities, FX, and credit (BIS, CCP Global); OTC derivatives outstanding stand at approximately $846 trillion notional with $21.8 trillion of gross market value (BIS, June 2025); and clearing members post over $420 billion of initial margin to major CCPs for cleared IRD and CDS alone (ISDA Margin Survey, year-end 2025).

No PFMI-grade CCP exists on any public blockchain today. Some on-chain venues run automated liquidation and insurance-pool loss absorption (perpetual DEXes such as Hyperliquid, dYdX, GMX, Bluefin), but none operate as a CCP under the BIS/IOSCO Principles for Financial Market Infrastructures: no legal novation, no segregated default fund, no multi-layer waterfall, no clearing-member structure, and no DCO-equivalent regulated status.

### Why Canton specifically

Canton is the only public chain with the architecture to host an institutional CCP:

- **Sub-transaction privacy.** During any transaction on Canton, each participant sees only the parts of the transaction tree where they are a stakeholder (signatory, observer, or controller). During a CCP-orchestrated liquidation, the CCP and the defaulting member observe the full close-out workflow; non-defaulting clearing members observe only the variation-margin updates that affect their own portfolios. Public-ledger-by-default chains expose every clearing member's position to every other member, which is incompatible with PFMI Principle 14 (segregation and portability) as institutional clearing operationalizes it.
- **DAML's authorization model** uses signatories, observers, and controllers as first-class contract primitives that map directly onto CCP legal roles: a clearing-member posting collateral is a signatory requirement, the CCP declaring default is a controller right, and a settlement bank's or regulator's read access is an observer relationship. The legal contract and the on-chain contract share the same authorization graph. Other smart-contract languages encode authorization through `msg.sender` or account-key checks, requiring application-layer code to reconstruct multi-party legal relationships with no enforced linkage between on-chain authorization and the underlying legal contract.
- **Deterministic transaction finality on the Global Synchronizer.** The Canton Global Synchronizer runs Byzantine fault-tolerant consensus with greater-than-two-thirds Super Validator quorum; once a transaction is committed it cannot be reverted. This is a structural prerequisite for any margin cycle or liquidation state machine, because probabilistic-finality chains require confirmation waits (minutes for economic finality on most L1s) that are operationally incompatible with intra-day margin computation and same-day default management.
- **Atomic multi-domain composability via the Global Synchronizer.** The Global Synchronizer uses two-phase commit to coordinate atomic multi-party, multi-domain transactions. A margin-call workflow that touches the CCP's clearing domain, a tokenized-payment domain (USDC), and a tokenized-collateral domain (USYC, tokenized USTs) executes as a single atomic transaction: either all sub-steps commit, or all sub-steps abort. There is no half-completed state where collateral has moved but instrument exercise has not, and no bridge hop where assets are exposed to counterparty risk during settlement. The engine's lifecycle and liquidation state machines can therefore treat collateral, payment, and instrument changes as a unified atomic operation rather than a sequence requiring application-layer retry or rollback logic.

The remaining structural gap is the smart-contract layer that novates trades, runs the margin cycle, and processes defaults under PFMI. This proposal closes that gap.

---

## 3. Alignment with Canton Architecture and Priorities

### Q2 priority areas (per Development Fund Proposal Review Process)

- **Scaling the Network:** cleared derivatives are the largest-by-notional category in capital markets; bringing them onto Canton is a primary throughput pathway.
- **App Building and Developer Experience:** the engine is asset-class-neutral and contributed as a reference implementation; other Canton apps can inherit clearing as a composable primitive rather than rebuilding it.
- **Security and Resilience:** the proposal includes property-based testing of margin-engine invariants and scenario-based testing of the default-waterfall in Milestone 2. Default-waterfall correctness is a security property of the clearing layer.

### CIP alignment

- **CIP-0082:** this proposal is submitted under the Protocol Development Fund established by CIP-0082.
- **CIP-0100 (Governance of the CIP-0082 Development Fund, approved January 2026):** this proposal conforms to CIP-0100 in form (CC-denominated milestones, quarterly milestone cadence, public PR submission, named SIG alignment) and substance (common-good contribution, milestone-based payouts).
- **CIP-0056 (Token Standard):** all collateral and instrument tokens conform to CIP-0056 where applicable, ensuring composability with the broader Canton token ecosystem.
- **CIP-0103 (dApp Standard, approved January 2026):** the engine's external interfaces (party onboarding, collateral deposit and withdrawal, margin notification, default declaration) are designed for compatibility with the CIP-0103 dApp Standard, so that clearing members and downstream applications can integrate with the engine through the same patterns used elsewhere in the Canton ecosystem.
- **Wallet Gateway interfaces:** clearing-member collateral deposit and withdrawal flows are designed to integrate with the Wallet Gateway reference implementation pattern currently under review in the Canton dev fund (PR #109), so that institutional key-management workflows are supported through the ratified pattern as the specification finalizes.

### Architectural alignment

The engine is built on the DAML Finance Contingent Claims framework, extending it with novation, margining, and default-management primitives. The Global Synchronizer's atomic two-phase commit is leveraged for cross-domain workflows during margin calls and liquidations (a single transaction can touch the CCP clearing domain, a tokenized-payment domain, and a tokenized-collateral domain, with all sub-steps either committing or aborting atomically).

---

## 4. Technical Approach

### Existing CCP engine (Ethereum prototype, internal)

Over the last two years EthosX has built a CCP-architected clearing prototype on Ethereum. The engine is asset-class-neutral by design, with eight modules:

1. **Novation:** implemented
2. **Real-time initial and variation margin math:** implemented (pricing consumes any supplied volatility surface; oracle-agnostic)
3. **Trading lifecycle:** implemented for European and American, and options including digital and knockout/knock-in variants
4. **Factory contracts:** implemented for on-chain option deployment; RFQ-and-orderbook-based matching with on-chain clearing
5. **Default-waterfall structural framework:** implemented with one layer (single default fund) populated
6. **Portfolio margining math:** implemented (currently runs off-chain)
7. **Margin-call triggering:** math implemented; on-chain state machine partial; orchestration currently off-chain
8. **Liquidation:** math done; on-chain state machine partial

### What the DAML port will deliver

The DAML port completes and extends:

- On-chain margin-call state machine with maintenance-margin triggers
- On-chain liquidation state machine
- Full multi-layer waterfall: defaulter margin, defaulter fund, CCP skin-in-the-game, mutualized fund, assessment powers
- On-chain portfolio margining integrated with clearing-member accounts
- ISDA CDM adapter for cross-asset extensibility
- IRS lifecycle module as the second asset-class instantiation, drawing on the proposer's direct experience at JP Morgan Global Derivatives Clearing

### External interfaces

- **Oracles:** Chainlink data standard (Data Streams, SmartData NAV and AUM feeds, Proof of Reserve) is live on Canton as of February 2026, with Chainlink Labs operating as a Canton Super Validator. The engine consumes Chainlink price feeds for margin calculation, NAV and AUM data for collateral valuation, and Proof of Reserve attestations for collateral solvency checks. Chainlink data is cryptographically signed by the Decentralized Oracle Network and verified on-chain by consuming contracts. Credit-event observables for future asset-class extensions (CDS, CDX in Phase 2) will be sourced from Chainlink or equivalent oracle feeds when those asset classes are added.
- **Settlement:** USDC, USYC, and tokenized USTs are live on Canton today. Tokenized Gilts have been demonstrated in cross-border intraday repo transactions on Canton (February 2026, with LSEG, Euroclear, DTCC, Citadel Securities, Cumberland DRW, and others). Société Générale's SG-FORGE EUR CoinVertible (EURCV) and USD CoinVertible (USDCV) are announced for Canton deployment (May 2026), positioned for collateral and repo workflows. Native JPMD issuance on Canton is announced for phased 2026 rollout (JPMD currently lives on Coinbase's Base; native Canton issuance is under development). The engine's settlement abstraction is collateral-token-agnostic and will integrate additional collateral tokens (CoinVertibles, native JPMD, tokenized Gilts and other sovereign bonds) as they reach production on Canton.
- **Risk feed:** Calypso-compatible interface, following the precedent established by the Nasdaq Calypso × QCP × Primrose × Digital Asset integration in June 2025
- **Wallet integration:** clearing-member collateral deposit and withdrawal flows are designed to integrate with the Wallet Gateway reference implementation pattern currently under review in the Canton dev fund (PR #109). The engine's wallet-facing interfaces will track the Wallet Gateway specification as it is ratified.

### PFMI mapping

The engine is architected for mapping to BIS/IOSCO Principles for Financial Market Infrastructures (PFMI), specifically:

- Principle 4 (credit risk)
- Principle 5 (collateral)
- Principle 6 (margin)
- Principle 7 (liquidity risk)
- Principle 13 (participant default rules and procedures)
- Principle 14 (segregation and portability)

This mapping is published as part of the architecture documentation in Milestone 1, so that regulated CCPs and institutional participants can adopt the engine under existing supervisory frameworks.

### Backward compatibility

The work delivers new modules within daml-finance. It introduces no breaking changes to existing daml-finance instrument templates or to the Canton Protocol. Existing DAML Finance vanilla option templates remain unchanged; the engine extends them with clearing capability.

### Why this cannot be delivered by extending existing daml-finance components

DAML Finance today provides: instrument templates (European and American vanilla options, barrier options, FpML-based interest rate and currency swap interfaces, dividend instruments, credit-default templates), the Contingent Claims framework for representing payoff structures, and account and holding primitives for managing asset ownership. These templates describe individual trades and their payoff logic, and the holding primitives let parties record asset positions. They do not provide a central counterparty: there is no novation primitive that legally substitutes the CCP between two clearing-member counterparties; no margin engine that calculates initial and variation margin across a portfolio and triggers margin calls; no default waterfall that absorbs losses across defaulter margin, defaulter fund, CCP skin-in-the-game, mutualized fund, and assessment powers; no liquidation state machine that orchestrates a defaulter's portfolio close-out; and no clearing-member account structure that segregates positions and collateral per PFMI Principle 14.

The components proposed here are net-new on top of existing daml-finance work, not a re-implementation of it. The engine consumes existing DAML Finance instrument templates as inputs (it does not replace them) and produces the CCP-specific layer that converts a collection of bilateral trades into a centrally-cleared portfolio. Execution venues and reference patterns being built on Canton (CLOB-based and RFQ-based exchanges, settlement reference patterns, DEX reference implementations maintained by Digital Asset and ecosystem contributors) are likewise complementary: they handle execution, order matching, and asset transfer; the CCP layer proposed here handles the risk-management lifecycle that sits between execution and final settlement (novation of executed trades, margin computation across portfolios, default-management workflows). An on-chain derivatives stack on Canton requires both execution and clearing layers; this proposal addresses the latter.

### Security approach

During Phase 1 the engine is primarily testnet-deployed. There is no mainnet collateral, no live trading, and no production user exposure during the grant period. A best-effort mainnet reference instance may be hosted if a Canton mainnet operator wishes to support it (see Section 8 Adoption Plan), but any such mainnet reference is a public-demonstration deployment without production user exposure; mainnet deployment of the engine for production use is deferred to Phase 2 and gated on the external audit. Security work in Phase 1 is calibrated accordingly:

- **Property-based testing of margin and waterfall invariants** delivered as a Milestone 2 acceptance criterion (no spurious margin calls under unchanged portfolio; waterfall total preservation under default; idempotency of liquidation re-runs)
- **Published threat model** delivered alongside Milestone 2, identifying attack surfaces (oracle manipulation, time-of-check-to-time-of-use, partial-execution states, governance capture of the CCP party) and the engine's mitigations
- **Internal code review** by the EthosX engineering team for each milestone before submission for milestone acceptance

A full external security audit by a recognized DAML auditor is committed as a precondition to any mainnet deployment of the engine, scoped as a Phase 2 deliverable with dedicated audit budget. The audit is sequenced ahead of any public bug bounty: a bug bounty against unaudited code creates poor incentives (community hunters surface findings that a systematic professional audit would catch more cheaply, and without an audit baseline it is difficult to distinguish novel findings from known issues). The Phase 2 sequencing is therefore: external audit, audit-finding remediation, public bug bounty against the audited codebase, then mainnet deployment. A substantial external audit is most valuable on the feature-complete codebase that goes to mainnet, not on the testnet codebase that is still being extended during Phase 1.

---

## 5. Scope, Milestones, and Acceptance Criteria

Nine months: July 15, 2026 to April 21, 2027. Funding is denominated in CC, fixed at submission spot. Per Canton fund template, milestone tranches paid after the six-month mark from M0 acceptance (approximately January 15, 2027), specifically M2 and M3, are subject to CC re-evaluation at the six-month mark to account for USD/CC price volatility.

### M0: Proposal acceptance

**Target date:** July 15, 2026
**Amount:** $35,000 USD / approximately 225,000 CC (17.5% of total)

**Deliverables:**

- Signed grant agreement
- Kickoff call with Canton Foundation Tech & Ops representatives
- Public announcement of EthosX as a Canton ecosystem contributor
- Tracking issue opened in canton-foundation/canton-dev-fund

**Acceptance criteria:**

- Pull request merged
- Tracking issue live and referenced from the proposal file
- Public blog or announcement post published

---

### M1: Foundation layer

**Target date:** October 21, 2026
**Amount:** $55,000 USD / approximately 355,000 CC (27.5% of total)

**Deliverables:**

- DAML Finance scaffold for clearing-member, CCP, and settlement-bank parties
- On-chain novation primitive
- Initial- and variation-margin engine consuming an external volatility surface
- Option instrument templates: European and American, including digital and knock-out/knock-in variants
- Factory contracts for on-chain option deployment
- Architecture documentation including PFMI mapping
- Ecosystem engagement: attendance at Financial Workflows & Composability SIG sessions

**Acceptance criteria:**

- DAR package published to a public Canton package registry under Apache-2.0
- All instrument templates compile and pass unit tests
- Margin engine produces deterministic IM and VM values against a published test suite of trade scenarios
- Integration test against Canton testnet passing for at least one full novation and margin cycle (clearing-member onboarding, trade novation, margin calculation, collateral deposit)
- Test coverage greater than or equal to 80% across delivered modules (line coverage, measured via standard DAML coverage tooling)
- Architecture documentation published in the same repository
- Documented attendance at at least two Financial Workflows & Composability SIG sessions during the M0-to-M1 window, verifiable through SIG meeting minutes or chair attestation

---

### M2: Risk and lifecycle layer

**Target date:** January 20, 2027
**Amount:** $55,000 USD / approximately 355,000 CC (27.5% of total)

**Deliverables:**

- On-chain margin-call state machine with maintenance-margin triggers
- On-chain liquidation state machine
- Multi-layer default waterfall: defaulter margin, defaulter fund, CCP skin-in-the-game, mutualized fund, assessment powers
- On-chain portfolio margining integrated with clearing-member accounts
- Published threat model identifying attack surfaces (oracle manipulation, time-of-check-to-time-of-use, partial-execution states, governance capture) and the engine's mitigations
- Ecosystem engagement: at least one public office-hours session demonstrating the engine architecture and the M1 + M2 deliverables on Canton testnet

**Acceptance criteria:**

- Margin-call and liquidation state machines pass scenario tests covering core default scenarios: single-member default, multi-member default, partial-recovery liquidation, and full-waterfall exhaustion
- Waterfall layers correctly absorb losses per PFMI Principle 13 in the tested scenarios
- Property-based tests in place for key margin-engine invariants (waterfall total preservation under default; idempotency of liquidation re-runs)
- Portfolio margining produces measurably lower IM than the gross sum of position-level IM for correlated positions, against a published test portfolio
- Threat model document published alongside code, covering oracle, timing, partial-execution, and governance-capture attack surfaces
- At least one public office-hours session held during the M1-to-M2 window with published recording or transcript, demonstrating the engine architecture and a live walkthrough of the M1 + M2 deliverables on Canton testnet
- Test coverage greater than or equal to 80% across delivered modules

---

### M3: Multi-asset extension

**Target date:** April 21, 2027
**Amount:** $55,000 USD / approximately 355,000 CC (27.5% of total)

**Deliverables:**

- ISDA CDM adapter (subset covering rates trade representations sufficient for IRS clearing)
- IRS lifecycle module: fixing, payment-date generation, compounding, payment netting
- End-to-end integration test on Canton testnet
- Ecosystem engagement: public AMA or technical walkthrough on the completed engine; presentation at one Canton ecosystem forum (SIG or community channel)

**Acceptance criteria:**

- ISDA CDM adapter parses a published reference set of CDM JSON trade representations for rates instruments without error
- IRS lifecycle correctly generates and pays fixings against a published test calendar and rate-fixing source for at least one currency (USD)
- End-to-end integration test on Canton testnet: at least one IRS trade cleared from novation through fixing and final payment, with all collateral and payment legs settled atomically via the Global Synchronizer
- Test coverage greater than or equal to 80% across delivered modules
- Public AMA or technical walkthrough delivered on the completed engine with published recording or transcript; engine presented at at least one Canton ecosystem forum during the M2-to-M3 window

---

## 6. Phase 2 Roadmap (not in scope of this proposal)

Following acceptance of M3, EthosX intends to propose a Phase 2 grant (Q2 2027 onward; indicative scope $500K to $1M) covering additional asset classes and engine extensions:

- FX-forward and FX-option lifecycle (delivery and cash-settled variants; multi-currency collateral)
- Credit-event-triggered products (single-name CDS, index CDX)
- Futures lifecycle module (cash-settled, physically-settled, daily mark-to-market)
- Digital-payoff instruments at scale (M&A-contingent payouts, corporate-action contracts, regulatory regime hedges)
- Portfolio margining at clearing-member scale with cross-product netting
- Multi-currency collateral with haircuts and FX-correlation-aware margining
- Formal verification of additional margin and waterfall invariants beyond the property-based tests delivered in Phase 1
- Security-to-mainnet sequence: external security audit by a recognized DAML auditor, audit-finding remediation, public bug bounty against the audited codebase, and then mainnet deployment of the engine. Each step is scoped as a Phase 2 deliverable with dedicated budget.

Phase 2 is referenced for ecosystem context only and is not part of this proposal's funding request.

---

## 7. Long-term Maintenance Plan

All code delivered under this grant is Apache-2.0 licensed and contributed upstream to daml-finance, or to a successor open-source repository as directed by the DAML Finance maintainers and the Canton Foundation Tech & Ops Committee.

**Maintainer-of-record:** EthosX engineering team for a minimum of 12 months from final-milestone acceptance, covering bug fixes, security patches, and integration support for ecosystem builders.

**Release compatibility SLA:**

- Compatibility with each new Canton Protocol release within approximately 30 days of release
- Compatibility with each new Splice release within approximately 60 days of release
- Engine deprecation notices given no less than two minor releases in advance for any breaking interface change

**Security SLA:**

- CVE-classified Critical or High severity issues: patch within approximately 7 days of disclosure
- CVE-classified Medium severity issues: patch within approximately 30 days of disclosure
- Coordinated disclosure window for any security finding: 30 days before public disclosure, or as agreed with the reporter

**Issue and PR review SLA:**

- 5 business days for critical security issues
- 15 business days for general issues and feature pull requests

**Knowledge transfer:** documentation, architecture decision records, and at least one technical workshop or office-hours session delivered to Canton ecosystem developers during the grant period (target: end of M2).

**Continuity:** if EthosX is unable to continue as maintainer-of-record at any point, code ownership transfers to the DAML Finance core maintainers. EthosX commits to a six-month transition support window in that scenario, including documentation handover, architecture walkthroughs with successor maintainers, and a final security review pass.

---

## 8. Adoption Plan

The clearing engine is shared infrastructure delivered as a public good, not a commercial product. The engine is published under Apache-2.0 and contributed upstream to daml-finance so any Canton participant can adopt it without commercial engagement with EthosX.

### Target users

The engine targets four categories of Canton ecosystem participants:

1. **Canton-native execution venues building cleared derivatives infrastructure.** Exchanges and execution platforms on Canton (CLOB-based, RFQ-based, or hybrid) that need a clearing layer behind their execution stack. These venues can integrate the engine as a clearing primitive rather than rebuilding novation, margin, and default-management logic from scratch.
2. **Institutional CCP operators evaluating Canton.** Regulated CCPs evaluating on-chain infrastructure can use the engine as a reference architecture for what a PFMI-mapped on-chain CCP looks like, accelerating their internal proof-of-concept timelines and reducing the engineering cost of any subsequent integration.
3. **Tier-1 banks running on-chain collateral, repo, and tokenized-derivatives workflows on Canton.** Existing Canton participants (banks, broker-dealers, market-makers, settlement venues) extending their on-chain workflows from bilateral CSA margining to centrally-cleared derivatives gain the missing layer their current workflows lack.
4. **Tokenized-derivative applications on Canton.** Application teams building cleared on-chain derivatives products (options, futures, swaps) for institutional or accredited customers can adopt the engine to deliver PFMI-mapped clearing without rebuilding CCP infrastructure themselves.

### Discovery mechanism

Three discovery surfaces, in priority order:

- **daml-finance package registry (primary).** Any Canton developer importing daml-finance encounters the clearing primitives as a standard library extension. This is the lowest-friction discovery path: the engine is found by the same registry lookups developers already use for instrument templates and Contingent Claims.
- **Public ecosystem engagement during the grant period.** SIG meeting participation, the M2 office-hours session, the M3 AMA and ecosystem-forum presentation. These are explicitly scoped as discovery surfaces, not only delivery activities.
- **Architecture documentation and PFMI mapping (published as ADRs in M1).** Reviewers and integrators can self-evaluate fit without commercial engagement. Institutional buyers in particular need to see PFMI mapping before any internal evaluation begins; publishing this in M1 makes the engine evaluable from day one.

### What initial adoption looks like

The kinds of signals that indicate early traction for a Phase-1 testnet engine (qualitatively described; specific counts depend on third-party choices outside EthosX's control):

- **Clearing-member parties instantiated** against the engine on Canton testnet by third parties (not just EthosX's reference deployment). Even a single non-EthosX testnet party signals integration intent.
- **Canton applications importing the daml-finance clearing extension package**, verifiable through public package-registry data. Each external import is a discovery and evaluation signal.
- **Ecosystem participant attestations or public commentary** on the engine, posted on the PR thread, in SIG meetings, or in ecosystem forums. These are the closest leading indicators of pre-production interest.
- **Publicly-documented architecture conversations** between EthosX and institutional participants (testnet integration discussions, technical walkthroughs, design reviews) during the engagement-ramp activities in M1-M3.

EthosX will publish these signals in real time as they occur. Production adoption metrics (cleared notional volume in production, recognized DCO or CCP status, commercial integration agreements) are explicitly Phase 2 objectives gated on the external audit and mainnet deployment, and outside the scope of this Phase 1 grant.

### Mechanisms

**Open contribution to daml-finance.** The engine is published as a daml-finance extension package under Apache-2.0. This is the primary mechanism through which the four target user categories above discover and integrate the engine.

**Ecosystem engagement during the grant period.** EthosX commits to a three-step engagement ramp across the milestones, calibrated to the maturity of the engine at each stage:

- **During M0 to M1 (foundation):** publish design documentation and architecture decision records in the open-source repository; attend Financial Workflows & Composability SIG sessions and engage with other relevant SIGs to gather technical feedback on the engine design.
- **During M1 to M2 (risk and lifecycle):** host at least one public office-hours session demonstrating the engine architecture and a live walkthrough of the M1 and M2 deliverables on Canton testnet, once the engine has enough working surface area to support a substantive demonstration.
- **During M2 to M3 (multi-asset extension):** host a public AMA or technical walkthrough on the completed engine, and present at at least one Canton ecosystem forum.

These activities surface design feedback from Canton-active institutional participants and ecosystem developers, and serve as discovery surfaces for the four target user categories. They are commitments to engagement and discovery, not commitments to third-party integration agreements.

**Reference deployment by EthosX.** EthosX will deploy a reference instance of the engine on Canton testnet throughout Phase 1, as a public, observable demonstration that the engine functions end-to-end. The reference deployment is a continuous-integration surface for the engine and a discovery surface for prospective integrators, not a commercial venue. A Canton mainnet reference instance is a best-effort, no-guarantee deliverable conditional on a Canton mainnet operator (validator, Super Validator, or institutional participant operating a participant node) willing to host the engine and accept the unaudited-code risk in writing during Phase 1. Full mainnet deployment of the engine for any production use is deferred to Phase 2 and is gated on completion of a full external security audit. Whether institutional participants choose to use the engine for actual trading is a separate commercial activity outside the scope of this grant.

### Adoption metrics tracked publicly

- Number of clearing-member parties instantiated against the engine on Canton testnet, and on Canton mainnet if a best-effort mainnet reference is hosted
- Number of distinct Canton applications importing the daml-finance clearing extension package
- Cumulative notional novated through the reference deployment on Canton testnet, and on Canton mainnet if a best-effort mainnet reference is hosted
- Number of ecosystem participants who have publicly evaluated or commented on the engine through Canton Foundation channels (PR thread, SIG meetings, ecosystem forums)

---

## 9. Team

**Deepanshu, Co-founder & CEO.** 12+ years across global financial markets and on-chain derivatives. Ex-Vice President, Global Derivatives Clearing (interest rate swaps), JP Morgan Chase. Buy-side and sell-side trading expertise. Alumnus of IIT Kharagpur and IIM Calcutta.

**Smit Patoliya, Co-founder & CTO.** International buy-side experience; quant trader at Two Roads (options trading). Algorithmically traded multiple asset classes of derivatives. Alumnus of IIT Madras.

**Amit Kumar, Head of Research.** Ex-Quantitative Strategist at Quadeye Securities (top-3 Indian HFT). HFT and MFT trading expertise across US, European, and LATAM markets. Alumnus of IIT Delhi.

**Advisors:**

- **Bharat Gupta.** Senior Principal, Financial Services & Insurance at Infosys Consulting. 20+ years across capital markets, digital assets, and digital transformation for tier-1 financial institutions. Alumnus of IIT Delhi.
- **Jean-Luc Savignac.** 30+ years in global capital markets. Ex-Head of Equity and Index Global Sales at Eurex. Ex-Global COO Fixed Income Currencies and Commodities / COO Americas at Newedge Group (Société Générale).
- **Jose (Pepe) Ruiz.** 25+ years building and running derivative trading teams for major global financial institutions. Led North America trading desk at ED&F Man Capital. Led Emerging Markets trading desk at Crédit Agricole CIB.
- **Nikhil Talwar.** 15+ years of traditional derivatives trading, risk management, and portfolio management. Led volatility trading, OTC trading, and prop trading at LedgerPrime. Built and led the Digital Assets desk at Chicago Trading Company.

### Shipping capability evidence

EthosX has shipped two on-chain derivatives products to date: a perpetual options protocol deployed on Arbitrum and BNB Chain, and a creator-led prediction market platform on Aptos (cumulative volume $40M+, 70,000+ traders, 120 active creators, $0 customer acquisition cost). Both serve as evidence of EthosX's ability to ship complex on-chain financial primitives at production scale. Neither is in scope for this proposal.

---

## 10. License

This proposal document is dedicated to the public domain under **CC0-1.0** (Creative Commons CC0 1.0 Universal).

All software deliverables produced under this grant are licensed under **Apache-2.0** and contributed to daml-finance, or to a successor open-source repository as directed by the DAML Finance maintainers and the Canton Foundation Tech & Ops Committee.

---

## 11. References

- Canton Foundation Protocol Development Fund: https://github.com/canton-foundation/canton-dev-fund
- CIP-0082 (Protocol Development Fund): https://github.com/canton-foundation/cips/blob/main/cip-0082/cip-0082.md
- CIP-0100 (Development Fund Governance): https://github.com/canton-foundation/cips/blob/main/cip-0100/cip-0100.md
- CIP-0103 (dApp Standard): https://github.com/canton-foundation/cips/blob/main/cip-0103/cip-0103.md
- JSCC × Mizuho × Nomura × Digital Asset JGB collateral PoC on Canton (April 2026, JFSA Payment Innovation Project): https://www.canton.network and corresponding JSCC and Mizuho press releases
- LayerZero × Canton interoperability protocol launch (March 2026): https://layerzero.network and Canton press release
- DAML Finance documentation: Contingent Claims framework, Option Instrument templates, FpML swap interfaces
- BIS/IOSCO Principles for Financial Market Infrastructures (PFMI), April 2012
- ISDA Margin Survey, Year-End 2025
- BIS OTC Derivatives Statistics, June 2025
- ISDA Common Domain Model (CDM)
- EthosX: https://www.ycombinator.com/companies/ethosx
