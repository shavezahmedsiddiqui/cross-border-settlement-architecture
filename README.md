Cross-Border Settlement Interoperability — Reference Architecture
Status: Conceptual engineering specification · Implementation: Not provided · Production ready: No

Author: Shavez Ahmed Siddiqui
Research: Official website
Abstract
Domestic instant-payment networks can move funds rapidly inside their respective systems, but international payments still depend on multiple institutions, foreign-exchange arrangements, liquidity, compliance checks, and legally distinct settlement mechanisms. This reference architecture describes a provider-neutral orchestration layer for coordinating cross-border payments without conflating API acknowledgments, customer credits, or ledger postings with final settlement.

The design applies lessons from real-time payment systems and double-entry ledger engineering to cross-border corridors. It is a technical model, not a deployed product, a connector to FedNow/Pix, or a claim of direct access to any payment network.
Design goals
1. Preserve financial correctness even when providers fail or messages are duplicated.
2. Keep payment intent, authorization, funding, external settlement, and reconciliation as separate states.
3. Support eligible corridor-specific adapters without assuming shared settlement rules.
4. Maintain an auditable double-entry journal and independently verifiable reconciliation.
5. Make FX quotations, liquidity availability, and settlement finality explicit.
Non-goals
- Direct FedNow-to-Pix connectivity or access to regulated payment networks.
- A production-ready AML, sanctions, KYC, or licensing solution.
- Guaranteed atomic settlement across independent fiat and tokenized rails.
- Automatic cross-border payouts without participating institutions and valid commercial/regulatory arrangements.
System architecture
<img width="1576" height="2406" alt="mermaid-diagram (5)" src="https://github.com/user-attachments/assets/9e3a1fde-8b5b-4b10-ae12-c4c7b20f1208" />
Boundary rule: An accepted instruction, HTTP 200 response, transaction hash, or beneficiary-facing status is not automatically proof of final settlement. Evidence requirements are defined per rail and applicable contractual/legal framework.
Component responsibilities
Component	Responsibility	Key invariant
API gateway	Authenticate requests and bind idempotency keys to intents	Retries must not create a new payment unintentionally
Policy engine	Validate user authority, account/corridor limits, and compliance decisions	No disbursement without authorized checks
FX service	Quote price, expiry, spread, and currency conversion	Expired quotes cannot silently execute
Liquidity manager	Check prefunding and per-corridor funding availability	Commitments do not exceed permitted liquidity
Ledger	Record balanced postings and reservation releases	Debits equal credits per accounting currency and entity
Route adapter	Translate canonical intent to provider-specific instruction	Adapter cannot claim finality without evidence
Reconciler	Match ledger, network status, and partner statements	Uncertain external status remains visible
Audit log	Preserve identifiers, policy decisions, and state changes	Corrections are recorded, not silently overwritten


Canonical payment instruction (illustrative)
{
  "payment_id": "pay_01HXEXAMPLE",
  "idempotency_key": "client_request_123",
  "source_account_id": "acct_source",
  "beneficiary_id": "beneficiary_456",
  "source_currency": "USD",
  "destination_currency": "BRL",
  "source_amount_minor": 100000,
  "fx_quote_id": "fx_quote_789",
  "corridor": "US-BR",
  "requested_at": "2026-11-09T09:00:00Z"
}
This schema is illustrative only. Monetary precision, currencies, quotations, beneficiaries, and country-specific fields require additional validation. Never trust raw user-provided fields as authorization evidence.
Payment lifecycle
<img width="1048" height="2030" alt="mermaid-diagram (6)" src="https://github.com/user-attachments/assets/5742a416-de7c-457f-8903-63ce24b8b9bb" />
A state transition must be supported by authoritative evidence and executed according to deterministic policy. Do not treat a timeout as evidence of failure: an order can have been received and processed even when its acknowledgment was lost.
Double-entry ledger principles
For a hypothetical internal same-currency transfer of 500 units between customer liability accounts:
Account	Debit	Credit
Customer A liability	500	0
Customer B liability	0	500
Total	500	500


Invariant: Total debits = total credits within each balanced accounting journal and currency. A cross-currency movement is not balanced by adding USD and BRL amounts together; FX transactions require explicit currency legs, exchange-rate terms, and corresponding accounts. Reservations, available balances, external settlement assets, fees, and beneficiary obligations must be accounted for separately.

Recommended patterns:
- Atomic commit for journal postings belonging to the same accounting transaction.
- Append-only journal entries; correct errors with authorized adjusting/reversing postings.
- Serializable isolation, per-account ordering, or proven concurrency controls for balances.
- Durable idempotency records linked atomically to payment creation.
- Distinct available, reserved, and posted balance projections.
Liquidity and routing policy
Choose a route from eligible and operationally available corridors, not just the lowest advertised FX spread. Evaluate:
- Effective FX price, fixed/variable fees, and quote expiry.
- Prefunded or committed liquidity in destination currency.
- Settlement cutoffs, holidays, service availability, and legal finality.
- Counterparty exposure and provider concentration.
- Execution reliability, latency, exception rates, and beneficiary reachability.
The router should reject or queue transactions when no eligible route meets policy limits. It must not invent new access to FedNow, Pix, or tokenized rails.
Resilience and failure scenarios
Scenario	Expected safe behavior
Same request submitted twice	Return existing payment identity/outcome; do not send a second payout
Provider accepts payment but times out	Mark OutcomeUnknown; reconcile before retrying
FX quote expires before authorization	Require new quote and user/policy confirmation as applicable
Insufficient destination liquidity	Refuse routing or defer with clear pending status
External settlement confirmed, internal update fails	Recover from durable evidence and post exactly one corresponding ledger event
Token transaction fails after fiat settlement	Record residual obligation; use approved compensation workflow, not presumed fiat rollback
Reconciliation mismatch	Open exception with preserved audit trail and controlled resolution


Key measures
Track settlement-confirmation latency, quote-expiry rate, funding utilization, uncertain outcomes, reconciliation exception age, duplicate-suppression effectiveness, and payment completion by corridor. Availability alone is not a sufficient measure of correctness.
Interoperability standards and considerations
- ISO 20022: A financial message framework, not itself a universal settlement network. Implementations can vary by scheme; validate required field mappings and versions.
- Domestic instant payments: FedNow and Pix/SPI have distinct national participants, currencies, rules, and settlement arrangements. Neither is a generic direct API for an unaffiliated platform.
- FX principal risk: Where available, payment-versus-payment mechanisms can mitigate principal settlement risk; do not assume linked systems provide atomic PvP.
- Tokenized assets: Token rights, custody, issuer obligations, and redemption rules require separate legal and technical evaluation.
Suggested repository structure
cross-border-settlement-architecture/
├── README.md
├── docs/
│   ├── corridor-adapter-contract.md
│   ├── ledger-invariants.md
│   ├── transaction-state-machine.md
│   ├── fx-liquidity-policy.md
│   └── failure-scenarios.md
├── schemas/
│   └── payment-intent.example.json
└── tests/
    └── scenarios.md
The paths above are proposed future files, not code implemented in this repository. A later prototype should include reproducible tests for idempotency, concurrent debits, state recovery after a lost acknowledgment, cross-currency balancing, and reconciliation discrepancies.
Further reading
- BIS — Project Nexus
- Financial Stability Board — Cross-border payments roadmap
- Federal Reserve — FedNow Service
- Banco Central do Brasil — Pix
- ISO — ISO 20022
Disclaimer
This document is an educational, provider-neutral architecture proposal. It does not constitute financial, legal, or regulatory advice, and it does not claim actual payment scheme access, regulatory approval, production implementation, or tested throughput. Independent review is required before any deployment handling real funds.
