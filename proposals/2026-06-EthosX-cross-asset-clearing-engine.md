## Development Fund Proposal: Institutional Cross-Asset Clearing Engine for Canton

| Field | Value |
| :---- | :---- |
| Author | Deepanshu, Co-founder & CEO, EthosX |
| Org | EthosX, Inc. (Delaware C-corp; Y Combinator Summer 2022; backed by Y Combinator and Franklin Templeton) |
| Status | Submitted |
| Created | 2026-06-24 |
| Label | defi-liquidity |
| Champion | Need Champion (engaging the DeFi Protocols and Liquidity SIG) |
| PR | (to be assigned on open) |

---

## Abstract

This proposal contributes an open-source cross-asset central-counterparty (CCP) clearing engine to daml-finance under Apache-2.0: on-chain novation, a per-pool multi-layer default waterfall, risk and lifecycle state machines, an on-chain default-auction process, and an off-chain margin computation service whose results are attested and verified on-chain. Canton has collateral, risk, and settlement rails for institutional derivatives, but no clearing layer that sits on top of them. This grant builds that layer, options-clearing end-to-end, as a reusable primitive.

Clearing is shared infrastructure by nature: segregated member accounts and a mutualized default waterfall only function across many independent participants, so no single firm should own the primitive. The grant funds the open-source library and its reference implementation only. Any clearing instance EthosX later operates, or any service it offers on top, is a separate, unfunded activity, and the same primitive is equally available for others to operate.

Architecturally, the compute-heavy margin math runs off-chain on observed market data; the on-chain layer verifies cryptographically-signed margin amounts and enforces consequences (margin calls, defaults, waterfall execution). This is the correct shape for clearing on Canton: computation off-chain, attested results and consequence-bearing actions on-chain.

EthosX requests **1,290,000 CC (approximately $200,000 USD at submission spot)** across four milestones over nine months, from July 15, 2026 to April 21, 2027.

---

## Specification

### 1. Objective

**The single objective: a CCP clearing primitive for daml-finance.** Canton today has bilateral CSA margining, tokenized collateral, oracle feeds, and atomic settlement, but no central counterparty: no on-chain novation, no segregated default fund, no multi-layer waterfall, no on-chain liquidation, default-auction, or margin-call state machine, and no margin-attestation contract that consumes off-chain-computed margin under a segregated member-account model. This proposal delivers that one primitive, instantiated end-to-end for options, with the milestones as a dependency chain toward it rather than independent workstreams.

A clarification on the scope of the claim. The engine provides the technical primitives a clearing house deploys (novation, margin, default management, segregation). The legal enforceability of novation and close-out netting in a counterparty insolvency, and PFMI or DCO-equivalent regulatory status, are properties of a deploying institution, its rulebook, governing law, and netting opinions, not of the software. The engine is designed so a regulated, capitalized CCP operator can adopt it under existing supervisory frameworks; it does not by itself confer regulated status.

**Success looks like:** the clearing primitives are importable from a public Canton package registry under Apache-2.0; a party other than EthosX can stand up a pool, novate a trade, run a margin and liquidation cycle, and process a default on testnet from published documentation; and institutional integrators can self-evaluate PFMI fit from published architecture documentation without any commercial engagement with EthosX.

**Out of scope (Phase 2, not funded here):** interest-rate-swap lifecycle and the ISDA CDM adapter; FX, credit, and futures asset classes; digital payoffs at scale; production-scale portfolio margining and alternative risk models (historical VaR, expected shortfall); nested client tiering; cross-pool auctions and designated backstop liquidity providers; attestation hardening (k-of-n, MPC/TEE, redundant attestors); multi-currency collateral; formal verification; and the security-to-mainnet sequence (external audit, remediation, bug bounty, mainnet). Phase 1 is options-only, single-pool, testnet-first, so it delivers a credible end-to-end implementation before adding cross-asset complexity.

### 2. Implementation Mechanics

**Role model: regulated topology by default, separable by design.** The engine does not hard-code a single bundled clearing member. It provides composable, cryptographically-scoped roles around a margin-pool primitive (each pool an isolated mini-CCP) and sorts functions into three groups. Risk-bearing functions are bound together by enforceable pool policy: a Member must hold a contribution in the mutualized default-fund tranche, and the Pool-Operator must hold its own skin-in-the-game tranche. Risk-neutral functions are freely separable (Custodian, KYC and Eligibility Provider, RiskEngine attestor, Keeper, Liquidator, Regulator-observer), which only reduces concentration and improves least privilege. A third-party Default-Fund Contributor is an additive option. The binding and eligibility constraints are enforced on-chain by a `PoolPolicy` contract: every state-changing choice fetches the active policy and asserts conformance through Daml `ensure` clauses, so a violating state cannot be created. The mutualization that is the entire point of central clearing is preserved by construction, and the regulated clearing-member structure is recovered as the engine's default configuration. Client tiering is recovered where a market wants it through a parent-pool member operating a child sub-pool. Full role detail is in Appendix A.

The mutualized tranche is funded by member contributions in the regulated default configuration, which is what makes the auction incentive credible (a member that bids poorly in a default auction has its own attributable contribution moved junior). Other pool configurations fund the mutualized tranche by a per-position add-on or by third-party contributors, as a `PoolPolicy` parameter; that funding flexibility is what lets the same engine also serve non-institutional pools.

**Reuse from an existing prototype (ported, not rebuilt).** Over the last two years EthosX has built a CCP-architected clearing prototype on Ethereum: novation, real-time IM and VM margin math, the full option trading lifecycle (European and American, including digital and knock-out / knock-in variants), factory contracts, a single-layer default fund, off-chain portfolio margining math, and partial on-chain state machines for margin calls and liquidation. The off-ledger SPAN and QuantLib margin engine already exists and is ported, not rebuilt. The on-chain default auction, the multi-layer waterfall, juniorization, and the hedging stage are net-new in the DAML build.

**The DAML port.** The engine will be expressed as new DAML templates and interfaces over daml-finance primitives (Contingent Claims, `Instrument.Generic`, Account, Holding, Lifecycle, Settlement). Option payoffs will be modeled as Contingent Claims trees; the net-new clearing layer (novation, portfolio margin, default management, member-solvency state machines, collateral segregation with the off-ledger attestation boundary) is what is not expressible in the claim algebra and is what the grant funds. In one line: Contingent Claims models what an instrument pays; the engine models what happens when a member cannot pay. The template and interface inventory, the representative claim trees, and the net-new boundary are detailed in Appendix A.

**Off-chain compute, on-chain attestation.** Portfolio margining is not a workload for on-chain computation (vol-surface interpolation, scenario grids, real-time data). The margin service will compute IM by a SPAN-style scenario grid plus VM, with portfolio netting within the pool, and output a cryptographically-signed margin envelope. The methodology is version-pinned and attested: at least 99% single-tailed confidence over a margin period of risk, anti-procyclical, with daily backtesting; the margin period of risk is a configured, backtested parameter rather than a fixed constant, capable of sub-day values given 24/7 operation and permissionless liquidation, with a regulatory floor applied where a recognized-CCP pool requires it. SPAN-style is chosen over historical VaR for scenario transparency, applicability to newly listed instruments with no long return history, and reproducibility against published fixtures. The on-chain `MarginAttestation` contract verifies the attestor's authorization, model version, staleness, replay protection, and adequacy, then exposes the amounts via `IMarginAttestation`. These checks verify authorization and message integrity, not the correctness of the computation itself: in Phase 1 correctness rests on the single trusted attestor plus publicly reproducible test fixtures, and is trust-minimized to a k-of-n or MPC/TEE attestor in Phase 2. This is the same attest-then-verify pattern by which Chainlink Data Streams deliver price feeds to Canton today.

On a stale or absent attestation, the staleness gate blocks risk-increasing actions (opening positions, withdrawing collateral) and holds the pool at its last attested-adequate state. Risk-reducing actions stay live: a flatten-only emergency close (a flat book carries no IM) and permissionless liquidation against the last-good attestation. This addresses operational risk under PFMI Principle 17.

**Default management.** Delivered as a multi-stage process across Phase 1, sequenced to the team's capacity ramp. The M2 resolution spine covers marking, the grace window, continuous liquidation of liquid legs, portability of positions and backing collateral to a non-defaulting member, and the base per-pool waterfall (defaulter margin, defaulter contribution, mutualized fund, operator skin-in-the-game, assessments, recovery). The M3 specialized stages, which depend on the M3 margin engine's attested risk numbers, add the hedging stage, the on-chain sealed-bid commit-reveal default auction with juniorization, and rule-bound recovery tools (variation-margin gains haircutting, capped assessments, optional partial tear-up).

**DAML version strategy.** daml-finance currently targets DAML 2 (SDK 2.10.x LTS, Daml-LF 1.17). The engine's own modules target DAML 3 and the Global Synchronizer from day one. The integration is interface-typed: the engine depends on daml-finance interface packages and the `Claim` type, never implementation modules, and wraps Contingent Claims behind a thin adapter, so it wires against DAML 2 daml-finance today and DAML 3 once that migration completes. Because the integration is interface-typed, the M1 to M3 deliverables run against the current DAML 2 daml-finance through the adapter and do not block on the daml-finance LF-2 build; the LF-2 migration is additive, so the grant deliverables ship regardless of that external timeline.

**Security.** Phase 1 is primarily testnet: no mainnet collateral, no live trading, no production exposure. Security work is property-based testing of margin and waterfall invariants, a published threat model, and internal code review per milestone. A full external audit by a recognized DAML auditor is a precondition to any mainnet deployment and is a Phase 2 deliverable with dedicated budget; the Phase 2 sequence is audit, then remediation, then a public bug bounty, then mainnet.

### 3. Architectural Alignment

**Why Canton.** Canton is the only public chain with the architecture to host an institutional CCP. Sub-transaction privacy means each participant sees only the parts of a transaction tree where it is a stakeholder, so during a liquidation the operator and defaulter observe the full close-out while non-defaulting members observe only the variation-margin updates affecting their own portfolios; public-ledger-by-default chains expose every member's position to every other member, incompatible with how PFMI Principle 14 is operationalized. DAML's signatory, observer, and controller primitives map directly onto clearing legal roles, so risk-neutral functions can be separated while risk-bearing ones are bound, with the legal contract and the on-chain contract sharing one authorization graph. The Global Synchronizer provides deterministic Byzantine-fault-tolerant finality, a prerequisite for any margin or liquidation state machine, and atomic multi-domain composability, so a margin-call workflow touching clearing, payment, and collateral domains commits or aborts as one transaction with no half-settled state.

**Q2 priority areas.** Scaling the Network (cleared derivatives are the largest-by-notional category in capital markets); App Building and Developer Experience (the engine is contributed as a composable primitive other apps inherit rather than rebuild); Security and Resilience (default-management correctness is a security property, tested by property-based and scenario tests).

**CIP alignment.** Submitted under CIP-0082 (Protocol Development Fund) as dev tooling, a reference implementation, and critical infrastructure. Conforms to CIP-0100 (Dev Fund Governance) in form (CC-denominated milestones, public PR, named SIG, six-month re-evaluation) and substance (common-good contribution). Collateral and instrument tokens conform to CIP-0056 (Token Standard) where applicable; external interfaces are designed for CIP-0103 (dApp Standard) compatibility; member deposit and withdrawal flows track the Wallet Gateway reference pattern under review in the fund.

**PFMI alignment by design.** PFMI alignment is a design property delivered across the build, not a document published once. The `PoolPolicy` contract doubles as an on-ledger PFMI parameter contract (cover standard, collateral asset and optional lender-conduct constraints, margin methodology pin, liquidity plan, recovery tools, segregation mode), with templates asserting conformance through `ensure`. The mapping is architectural, not certification: actual PFMI or DCO status is a determination about a licensed legal entity that operates the CCP, and the operator carries the legal and regulatory burden. A principle-by-principle assessment is in Appendix B and is published and refined as the mechanisms land across M1 to M3.

### 4. Backward Compatibility

The work delivers new modules within daml-finance with no breaking changes to existing instrument templates or the Canton Protocol; existing vanilla option templates are unchanged and consumed as inputs. The engine owns clearing logic, not execution or collateral mobility: execution venues and settlement reference patterns handle order matching and asset transfer, and the engine handles the risk-management lifecycle between execution and final settlement. An on-chain derivatives stack needs both layers; this proposal addresses clearing.

---

## Milestones and Deliverables

Nine months, July 15, 2026 to April 21, 2027. Milestone dates are intentionally specific. The M1 to M3 deliverables run against the current daml-finance through the interface adapter and do not block on any external LF-2 build (see Risks and Mitigations).

### Milestone 0: Proposal acceptance and kickoff
- **Estimated Delivery:** 2026-07-15
- **Focus:** establish the workstream in the open.
- **Deliverables / Value Metrics:** signed grant agreement; kickoff with Canton Foundation Tech and Ops; public announcement of EthosX as a contributor; a public tracking issue the ecosystem can follow. Value metric: the clearing workstream is visible and trackable by any Canton participant from day one.

### Milestone 1: Foundation layer (CCP core, roles, option lifecycle)
- **Estimated Delivery:** 2026-10-21
- **Focus:** the asset-class-neutral CCP primitives, the role model with on-chain binding enforcement, and the first product instantiation (options), with margin math as a deterministic library. All on-chain modules target DAML 3 from day one.
- **Deliverables / Value Metrics:** a DAR published to a public Canton package registry under Apache-2.0 so any participant can import the clearing primitives; the role scaffold and `PoolPolicy` enforcing the role-binding and eligibility constraints; the on-chain novation primitive; option instrument templates extending Contingent Claims (European, American, digital, knock-out, knock-in); the deterministic IM and VM margin library; factory contracts; and architecture documentation including the PFMI mapping, the role model, and the DAML 2 to 3 compatibility ADR, written so institutional integrators can self-evaluate fit without engaging EthosX. Value metrics: the primitive is importable and self-instantiable from published docs; at least two DeFi Protocols and Liquidity and Financial Workflows and Composability SIG sessions attended with feedback documented and incorporated.

### Milestone 2: Risk and lifecycle layer (state machines, default-resolution spine)
- **Estimated Delivery:** 2027-01-20
- **Focus:** the on-chain state machines and the default-resolution spine, with interfaces designed to consume attested margin. The specialized default-management stages land in M3 with the margin engine they depend on.
- **Deliverables / Value Metrics:** the on-chain margin-call state machine over the `IMarginAttestation` interface; the liquidation state machine (marking, grace, continuous liquidation); the `Portability` flow; the base per-pool multi-layer waterfall; and a published threat model. Value metrics: a full margin-and-liquidation cycle is demonstrated on testnet by a party other than the operator, with a public office-hours session walking through M1 and M2 on testnet so other teams can reproduce it.

### Milestone 3: Off-chain margin attestation, specialized default management, and production-path readiness (pre-audit)
- **Estimated Delivery:** 2027-04-21
- **Focus:** close the architectural loop and deliver the specialized default-management stages at peak team capacity.
- **Deliverables / Value Metrics:** the off-chain margin computation service (Apache-2.0, including the SPAN engine, signer, and attestor reference, not only the on-chain templates); the `MarginAttestation` contract, `AttestorRegistry`, and `RiskModelCatalog`; the liquidation hedging stage; the `DefaultAuction` (sealed-bid commit-reveal) with juniorization; recovery tools in the waterfall; full lifecycle integration; performance benchmarks; the DAML 3 migration ADR; developer integration guides; and an "operate your own pool" quickstart that lets a third party stand up a pool, attestor, and margin service independently of EthosX. Value metrics: a party other than EthosX stands up a working pool from the quickstart on testnet; a public AMA or technical walkthrough and an ecosystem-forum presentation are delivered; integration guides carry at least three worked examples.

---

## Acceptance Criteria

The Tech and Ops Committee evaluates completion on value to the ecosystem, not delivery of an artifact. Per milestone:

- **M0:** the PR is merged, the tracking issue is live and referenced, and the contributor announcement is published.
- **M1:** the DAR is published under Apache-2.0 to a public registry and is importable by a third party; `PoolPolicy` rejects states that violate the active binding and eligibility constraints (a member without a mutualized contribution and an operator without a skin-in-the-game tranche are both rejected), shown by test; a full novation and margin cycle runs on Canton testnet; the architecture and PFMI documentation is published and is sufficient for an external integrator to self-evaluate; at least two SIG sessions are documented with feedback incorporated.
- **M2:** the margin-call and liquidation state machines pass scenario tests (single-member default, multi-member default, continuous-liquidation close-out, full-waterfall exhaustion); `Portability` transfers a defaulter's positions and collateral atomically to a non-defaulting member; the base waterfall absorbs losses correctly in the tested configuration; property-based invariants hold (waterfall total preservation, idempotent liquidation re-runs, no margin call under attested-sufficient collateral); a party other than the operator completes a margin-and-liquidation cycle on testnet; the threat model and an office-hours recording are published.
- **M3:** the off-chain margin service is deployed publicly with reproducible fixtures and verifiable signatures; the attestation contract rejects the full published negative-case matrix; attestor-key registration and rotation are demonstrated; portfolio margining produces lower IM than the gross sum for correlated positions against a published portfolio, attested on-chain; the `DefaultAuction` passes a single-pool commit-reveal scenario with juniorization of a non-participating member; the hedging stage and recovery tools are demonstrated; the liveness failure mode is demonstrated; a third party stands up a pool from the quickstart on testnet; benchmarks, the migration ADR, integration guides, and the AMA or forum presentation are published.

---

## Funding

**Total Funding Request: 1,290,000 CC (approximately $200,000 USD at submission spot).**

### Payment Breakdown by Milestone
- Milestone 0 (Proposal acceptance and kickoff): 129,000 CC (10%) upon committee acceptance
- Milestone 1 (Foundation layer): 387,000 CC (30%) upon committee acceptance
- Milestone 2 (Risk and lifecycle layer): 387,000 CC (30%) upon committee acceptance
- Milestone 3 (Attestation, specialized default management, production-path readiness): 387,000 CC (30%) upon final release and acceptance

### Budget Rationale
The request funds engineering time across three build roles, with no external-audit cost in Phase 1 (audit is a Phase 2 deliverable with its own budget). Role emphasis tracks the milestone difficulty ramp: M1 is DAML-heavy (the on-chain templates, interfaces, and tests); M2 adds risk-state-machine work; M3 is full-team, with the quant and risk lead on the ported off-ledger margin service and the off-chain and infrastructure work on the attestation system, while the most specialized net-new on-chain work (auction, juniorization, hedging) runs in parallel. The figure is comparable to accepted reference-implementation grants of similar scope.

### Volatility Stipulation
The project duration is greater than six months. The grant is denominated in fixed Canton Coin and will require a re-evaluation at the six-month mark (approximately January 15, 2027), affecting the M2 and M3 tranches. EthosX accepts this term.

---

## Co-Marketing

Upon delivery, EthosX will collaborate with the Foundation on: announcement coordination at M0 (contributor announcement) and M3 (release); a technical blog or case study on on-chain CCP clearing and the off-chain-compute / on-chain-attestation pattern; and developer and ecosystem promotion through the M2 office-hours session, the M3 AMA or technical walkthrough, and an ecosystem-forum presentation. The architecture documentation and PFMI mapping are published openly as discovery surfaces for institutional integrators.

---

## Team and Delivery Capability

The build roles for the nine-month plan are led by the EthosX team.

- **Deepanshu, Co-founder & CEO.** Ex-Vice President, Global Derivatives Clearing (Interest Rate Swaps), JP Morgan Chase. Covers off-chain and infrastructure delivery (the attestation signer, indexer and keeper, Canton integration). Alumnus of IIT Kharagpur and IIM Calcutta.
- **Smit Patoliya, Co-founder & CTO.** International buy-side quant trader at Two Roads (Options Trading). Leads the DAML and Canton build (templates, interfaces, tests). Alumnus of IIT Madras.
- **Amit Kumar, Head of Research.** Ex-Quantitative Strategist at Quadeye Securities. Owns the quant and risk work (the ported SPAN and QuantLib margin service). Alumnus of IIT Delhi.

Internal security review runs at each milestone. The off-ledger margin engine being a port rather than a rebuild, and Phase 1 being scoped to options end-to-end, are what keep this scope realistic.

**Advisors:** Bharat Gupta (Senior Principal at Infosys Consulting, leading Digital Assets within the Financial Services and Insurance practice; advises in an individual capacity); Jean-Luc Savignac (ex-Head of Equity and Index Global Sales at Eurex; ex-Global COO FICC and COO Americas at Newedge Group, Societe Generale); Jose (Pepe) Ruiz (led North America trading at ED&F Man Capital and the Emerging Markets desk at Credit Agricole CIB); Nikhil Talwar (led volatility, OTC, and prop trading at LedgerPrime; built and led the Digital Assets desk at Chicago Trading Company). Infosys is a prospective infrastructure collaborator for the reference deployment, distinct from and out of scope for the open-source library this grant funds.

**Shipping capability evidence.** EthosX has shipped two on-chain derivatives products, a perpetual options protocol on Arbitrum and BNB Chain and a creator-led prediction market on Aptos ($40M+ cumulative volume, 70,000+ traders, 120 active creators, $0 CAC). These establish the capability to design, build, and operate complex on-chain financial primitives at production scale. Neither is in scope for this proposal; they are cited only as delivery evidence.

---

## Risks and Mitigations

- **M3 concentration.** M3 carries the attestation system plus the specialized default-management stages. Mitigation: the M1 and M2 core (roles, novation, option lifecycle, margin library, margin-call and liquidation state machines, portability, base waterfall) is self-contained and ships and is independently useful regardless of M3; within M3, the auction builds on a parallel track from the attestation work.
- **DAML build capacity.** A lean team builds the on-chain modules. Mitigation: the off-ledger margin engine is ported from the existing prototype rather than written from scratch, Phase 1 is scoped to options only, and the milestone ramp concentrates the most specialized work at peak fluency; the prospective Infosys collaboration is an additional capacity path for the reference deployment.
- **daml-finance LF-2 / DAML 3 timeline.** The one external forward dependency is the daml-finance LF-2 build. Mitigation: the interface-typed adapter lets every grant deliverable run against the current DAML 2 daml-finance, so delivery does not block on that timeline; LF-2 migration is additive.
- **Single Phase-1 attestor.** The sole reference attestor is an availability and an integrity single point of failure. Mitigation: the liveness gate degrades safely (block risk-increasing, allow risk-reducing), correctness rests on reproducible public fixtures, and threshold or MPC and TEE hardening plus redundant attestors are the Phase 2 path.

---

## Motivation

Central counterparty clearing processes more value than any other layer in capital markets. Globally, CCPs clear well over a quadrillion dollars of notional each year across rates, equities, FX, and credit: LCH SwapClear alone cleared a record $1,319 trillion of interest-rate-swap notional in 2023, over $1.3 quadrillion in a single asset class at a single clearing house. OTC derivatives outstanding stand at approximately $846 trillion notional with $21.8 trillion of gross market value (BIS, June 2025), and market participants post $423.5 billion of initial margin to major CCPs for cleared IRD and CDS alone (ISDA Margin Survey, year-end 2025).

Over the last fifteen months Canton has built the rails around clearing: QCP and Digital Asset bilateral CSA margining; Nasdaq Calypso, QCP, Primrose, and Digital Asset risk and margining integration; live 24/7 on-chain UST financing on Tradeweb with a roster of tier-1 firms; LayerZero as the first interoperability protocol; a JSCC, Mizuho, Nomura, and Digital Asset JFSA-backed JGB collateral proof-of-concept; tokenized collateral (USDC, USYC, tokenized USTs, with native JPMD announced); and Chainlink data live on Canton. What is missing is the clearing layer on top. QCP's work is bilateral; the JSCC proof-of-concept brings an incumbent's own collateral mobility on-chain but does not deliver the smart-contract clearing logic that operates on posted collateral. The incumbents arriving validate the demand; they will not build an open primitive for everyone else.

**Who is unblocked.** The primitive is for the long tail that will never build bespoke clearing the way a JSCC does: Canton-native execution venues that cannot offer centrally-cleared products because there is no clearing layer behind their order books; institutions stuck at bilateral margining; tokenized-derivative application teams that would each otherwise rebuild novation, margin, and default management from scratch; and institutional operators evaluating Canton who today have no PFMI-mapped on-chain CCP reference to study. None of these benefits require engaging EthosX commercially.

**Why this is a public good, not a single-company product.** A clearing layer is only useful if many independent parties rely on the same primitives: segregated member accounts mean nothing if only one firm's accounts exist, and a default waterfall is only credible if multiple members mutualize risk through it. Clearing is, definitionally, common infrastructure no single participant should own. If it lived in one company's private stack, every other team would rebuild the same regulated-clearing primitives, exactly the duplication a public-good fund exists to prevent. The deliverable is contributed to daml-finance under Apache-2.0, importable by anyone including competitors, and the off-chain margin service implementation is contributed too, not just the on-chain templates, so the commons includes the hard part.

---

## Rationale

**Why extend daml-finance rather than build standalone.** The engine is built on the Contingent Claims framework and composes with daml-finance Account, Holding, Instrument, and Lifecycle primitives. Contingent Claims models the bilateral instrument: contingent cashflows and the conditions under which they occur. It does not, and cannot, model a central counterparty. The net-new layer is novation, portfolio margin (worst-case scenario loss over a netted book, for which Contingent Claims has no margin, scenario, or offset concept), default management (waterfall, juniorization, liquidation, auction), member-solvency state machines, and collateral segregation with the off-ledger attestation boundary. That boundary is precisely why this layer does not already exist by extending what is there, and it is what the grant funds. The detail is in Appendix A.

**Why contributor, not owner.** EthosX contributes as an open-source maintainer and seeks no exclusive rights or licensing control over the primitive. EthosX intends to remain among the maintainers of the upstreamed module under the Foundation's open-source governance, a stewardship role exercised through the community and SIG process rather than internal priority, with continuity transfer to the daml-finance core maintainers if EthosX steps back. Any clearing instance EthosX operates, or any hosting or margin-attestation service it provides, is a separate, unfunded commercial activity equally open to others to provide. Any featured-application reward that a reference deployment might earn is incidental and is not a grant deliverable.

**Why the off-chain-compute / on-chain-attestation shape.** On-chain portfolio margining is not feasible (vol surfaces, scenario grids, real-time data). Computing off-chain and attesting on-chain is the same pattern Canton already uses for oracle data through Chainlink Data Streams, so it fits the existing ecosystem rather than inventing a new trust model.

**Maintenance.** All code is Apache-2.0, contributed upstream. EthosX commits to maintainer-of-record duties for at least twelve months from final acceptance (bug fixes, security patches, integration support), with target compatibility windows for new protocol and Splice releases and best-effort security response times, backstopped by the continuity transfer to the core maintainers.

---

## Appendix A: Daml model detail

The engine is expressed as new DAML templates and interfaces over daml-finance primitives, classified by relationship to existing primitives (composes-with, extends, or net-new).

**Templates the port will add:**

| Template | Purpose | Relationship to daml-finance / Contingent Claims | Milestone |
|---|---|---|---|
| `MemberAccount` | segregated positions and collateral per member (LSOC); member is direct signatory | composes-with Account + Holding | M1 |
| `Novation` | interposes the pool operator as counterparty to each side of a matched trade (operator-faced legs that net within `MemberAccount.positions`) | net-new | M1 |
| `OptionInstrument` | option payoff as a Contingent Claims tree | extends Contingent Claims (implements `Interface.Claims.HasClaims`); composes-with `Instrument.Generic` | M1 |
| `ClearedTrade` | immutable per-trade provenance and audit lineage from novation; not the live position store (`MemberAccount.positions` is authoritative) | composes-with Holding | M1 |
| `PoolPolicy` | enforceable eligibility, role-binding, and PFMI-parameter rules per pool | net-new | M1 |
| `MarginCallState` | state machine that consumes attested margin | net-new (consumes `IMarginAttestation`) | M2 |
| `LiquidationState` | defaulter close-out orchestration: mark, grace, continuous liquidation, and portability in M2; hedging and auction stages in M3 | net-new | M2 (hedging stage M3) |
| `Portability` | atomic transfer of a defaulter's positions and backing collateral to a non-defaulting member or the operator | net-new | M2 |
| `DefaultWaterfall` | per-pool, operator-configured loss-absorption layers (up to six tranches); juniorization and recovery tools added M3 | net-new | M2 base; juniorization and recovery M3 |
| `DefaultAuction` | sealed-bid commit-reveal auction of a defaulter portfolio, with juniorization | net-new | M3 |
| `RiskModelCatalog` | version-pinned approved risk-model hashes per pool | net-new | M3 |
| `AttestorRegistry` | registered attestor identities and keys, with rotation | net-new | M3 |
| `MarginAttestation` | on-chain record of off-ledger margin, authorization-verified | net-new | M3 |

**Interfaces the port will define:**

| Interface | Purpose | Implemented by | Milestone |
|---|---|---|---|
| `IClearingInstrument` | common interface for any cleared instrument; `getClaims` extends daml-finance `HasClaims`, plus `getUnderlying` / `getExpiry` / `getSettlementCcy` / `getMarginClass` | `OptionInstrument` (a second family in Phase 2) | M1 |
| `IMarginAttestation` | abstraction over attested margin so state machines do not depend on the off-chain service shape | `MarginAttestation` | M2 (interface), M3 (implementation) |
| `IDefaultManagement` | uniform hook for waterfall, liquidation, and auction drivers | `LiquidationState`, `DefaultWaterfall` in M2; `DefaultAuction` in M3 | M2 interface, implementers M2 to M3 |

`IClearingInstrument` is real in M1 as a defined interface with a single implementer (options). Asset-class neutrality is aspirational until a second instrument family (IRS or FX, Phase 2) implements it, and is stated that way rather than claimed as proven in M1.

**Instrument modeling (Contingent Claims).** With `S` the spot observation, `K` the strike, `H` the barrier, `Q` the digital payout, and `ccy` the settlement currency, the representative trees are:

```
European call:    when (t == expiry) ( (scale (S - K) (one ccy)) or zero )
European put:     when (t == expiry) ( (scale (K - S) (one ccy)) or zero )
American call:    anytime (t <= expiry) ( (scale (S - K) (one ccy)) or zero )
Digital call:     when (t == expiry) ( cond (K <= S) (scale Q (one ccy)) zero )
Knock-out (DO):   until (S <= H) ( europeanCall )
Knock-in  (DI):   when  (S <= H) ( europeanCall )
```

The `or zero` is the holder's exercise election; `anytime` is the American election; the digital uses the `cond` constructor, a then-or-else branch gated by an inequality, and has no election; KI and KO reconstruct the vanilla. Barrier monitoring frequency and style, rebate, and physical-versus-cash settlement are product choices expressible in the same algebra. The Ethereum prototype runs this full suite today. daml-finance Lifecycle derives settlement effects from the claim tree; the engine's settlement commit applies them with the clearing checks (multi-account atomic collateral adequacy, the margin-attestation interaction, the novated counterparty, default coupling, per-pool isolation) that Lifecycle does not provide.

**The net-new boundary.** daml-finance provides instrument templates, the Contingent Claims framework, and account and holding primitives; these describe individual trades and record positions. They do not provide a central counterparty. The net-new clearing layer (novation, portfolio margin, default management, member-solvency state machines, and collateral segregation with the off-ledger attestation boundary) is not expressible in the claim algebra. Contingent Claims models what an instrument pays; the engine models what happens when a member cannot pay.

**Role detail.** The full role table (Pool-Operator, Member, Default-Fund Contributor, Custodian, KYC and Eligibility Provider, RiskEngine attestor, Keeper, Liquidator, Regulator) and the three-bucket sort (bound by policy, freely separable, additive) underpin Section 2. Collateral and default funds are pool-scoped and not fungible across pools in Phase 1: a member active in two pools posts and is margined separately in each; cross-pool netting and cross-pool auctions are Phase 2.

---

## Appendix B: PFMI principle mapping (summary)

Architectural alignment, not certification. Each engine-implementable principle is satisfied by a named mechanism; entity and governance principles are operator-scope.

- **P4 Credit risk:** IM (off-ledger SPAN) plus the waterfall; default-fund stress testing to a Cover-1 or Cover-2 standard (pool-configured) as an attested off-ledger obligation.
- **P5 Collateral:** single-collateral pools (`PoolPolicy.collateralAsset`), with optional binding lender-conduct constraints; asset variety and transformation are a Phase 2 Lender and `CreditLine` mechanism on the lender's own capital, no rehypothecation of segregated margin.
- **P6 Margin:** SPAN scenario-grid IM and VM, methodology pinned in `RiskModelCatalog` at 99% or higher confidence over a configured, backtested margin period of risk, anti-procyclical, daily backtesting.
- **P7 Liquidity risk:** liquidity plan in `PoolPolicy`; off-ledger liquidity stress testing; Phase 2 committed facilities via the Lender.
- **P8 Settlement finality, P12 DvP:** Canton Global Synchronizer deterministic finality and atomic two-phase commit, by design.
- **P9 Money settlement:** regulated tokenized cash and stablecoins (commercial-bank or e-money, not central-bank money); the money-settlement risk is disclosed.
- **P13 Default rules and recovery:** `LiquidationState`, `DefaultAuction`, `DefaultWaterfall`, plus recovery tools (variation-margin gains haircutting, capped assessments, optional partial tear-up), rule-bound and disclosed.
- **P14 Segregation and portability:** per-member segregated `MemberAccount`s plus pool isolation (individual LSOC-style segregation) and the `Portability` flow.
- **P17 Operational risk:** degraded-mode attestor liveness in Phase 1; threshold or MPC attestor and redundant keepers in Phase 2.
- **P22 Communication, P23 Disclosure:** ISO 20022 and standard identifiers (LEI, UTI, UPI) accommodated at the integration boundary; a public quantitative disclosure export and a Disclosure Framework response are operator-authored.
- **Entity and governance principles (P1, P2, P15, P18, P19):** operator-scope, with on-ledger hooks (the authorization graph, governance and regulator-observer roles, permissioned-pool and eligibility-provider configuration).

---

## References

- Canton Foundation Protocol Development Fund: https://github.com/canton-foundation/canton-dev-fund
- CIP-0082 (Protocol Development Fund): https://github.com/canton-foundation/cips/blob/main/cip-0082/cip-0082.md
- CIP-0100 (Development Fund Governance): https://github.com/canton-foundation/cips/blob/main/cip-0100/cip-0100.md
- CIP-0103 (dApp Standard): https://github.com/canton-foundation/cips/blob/main/cip-0103/cip-0103.md
- JSCC, Mizuho, Nomura, and Digital Asset JGB collateral PoC on Canton (April 2026, JFSA Payment Innovation Project)
- LayerZero and Canton interoperability protocol launch (March 2026)
- DAML Finance documentation: Contingent Claims framework, Option Instrument templates, FpML swap interfaces, Lifecycle, Smart Contract Upgrades
- BIS/IOSCO Principles for Financial Market Infrastructures (PFMI), April 2012; CPMI-IOSCO Recovery of Financial Market Infrastructures, rev. 2017; CPMI-IOSCO Public Quantitative Disclosure Standards for CCPs, 2015
- ISDA Margin Survey, Year-End 2025; BIS OTC Derivatives Statistics, June 2025; ISDA Common Domain Model (CDM)
- LCH SwapClear total notional cleared, 2023 (LSEG): https://www.lseg.com/en/post-trade/clearing/lch-services/swapclear
- EthosX: https://www.ycombinator.com/companies/ethosx
