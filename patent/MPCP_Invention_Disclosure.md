

# MPCP Invention Disclosure

## Title

Machine Payment Control Protocol for Cryptographically Constrained Autonomous Transactions

Alternative titles:

- Cryptographic Authorization Pipeline for Autonomous Machine Payments
- Method and System for Policy-Constrained Machine Wallet Transactions
- Layered Authorization Framework for Autonomous Payment Execution
- Cryptographically Bounded Human-to-Agent Spending Delegation via Decentralized Identifiers

---

# Purpose of This Document

This document captures the conceptual foundation for a **provisional patent filing** related to the **Machine Payment Control Protocol (MPCP)**.

The purpose of this document is to record the **invention concept and architecture** behind MPCP in a form suitable for discussion with a patent attorney and preparation of a provisional patent filing.

The intent is **defensive intellectual property protection** while keeping the MPCP protocol itself open and implementable.

The patent would protect the **method and architecture of constrained autonomous payments**, not the MPCP specification itself.

The MPCP specification and reference implementation remain open.

> **Terminology note:** This document uses generic names appropriate for patent claims. In the MPCP reference implementation, PolicyGrant maps to `PolicyGrant`, BudgetAuthorization maps to `SignedBudgetAuthorization` (SBA), and PaymentAuthorization maps to `SignedPaymentAuthorization` (SPA).

---

# Problem Statement

Autonomous systems such as:

- autonomous vehicles
- electric vehicles
- charging infrastructure
- robots
- drones
- AI agents
- IoT infrastructure
- autonomous logistics systems

will increasingly perform **financial transactions without direct human approval**.

Existing payment systems assume one of the following models:

1. **Human authorization** (manual approval)
2. **Centralized API authorization**
3. **Smart contract escrow**
4. **Unlimited wallet control by a private key**

These models do not provide a safe framework for **bounded autonomous financial actions**.

Key challenges include:

- machines must operate autonomously
- financial exposure must remain limited
- payments must be verifiable after settlement
- policy enforcement must remain deterministic
- wallets must refuse unauthorized transactions

MPCP introduces a layered cryptographic authorization architecture designed specifically for **machine-to-infrastructure payments** and, by extension, **human-to-agent spending delegation**.

---

# Core Invention

The core invention is a **layered authorization artifact pipeline** that constrains autonomous machine payments through a sequence of cryptographically verifiable artifacts.

The authorization chain consists of:

Policy
→ PolicyGrant
→ BudgetAuthorization
→ PaymentAuthorization (containing a SettlementIntentHash binding)
→ Deterministic Settlement Verification

Each stage narrows the permitted scope of payment execution.

This creates **bounded autonomy**, allowing machines to perform transactions safely within defined limits.

---

# Authorization Artifact Chain

## 1. Policy

A Policy defines the authorization rules that govern a machine's payment behavior. It is evaluated by a policy authority to produce a PolicyGrant.

A Policy may specify:

- permitted operators and lots
- permitted payment rails
- permitted assets
- spending caps
- geographic or temporal constraints
- approval requirements

---

## 2. PolicyGrant

A PolicyGrant is the evaluated result of a Policy. It defines the operational boundaries under which a machine may perform payments for a specific session or scope.

It may specify:

- permitted payment rails
- permitted assets
- risk thresholds
- expiration times
- approval requirements

A PolicyGrant is issued by a **policy authority**, such as:

- fleet operator
- infrastructure operator
- service network
- enterprise administrator
- human principal (via decentralized identifier / DID key)

### Human-to-Agent Delegation Extension

When the policy authority is a human principal, the PolicyGrant carries additional fields that
bound the scope of the AI agent's spending authority:

**`allowedPurposes`** — an explicit list of merchant categories (e.g., `travel:hotel`,
`travel:flight`) that the agent may pay for. The agent enforces this constraint as session
authority before signing a PaymentAuthorization. The field appears in the signed PolicyGrant
for audit trail purposes.

**`revocationEndpoint`** — a URL at which the human principal's wallet or service accepts
revocation queries. Merchants and service providers query this endpoint before accepting
payment to determine whether the delegation has been cancelled. The verification pipeline
remains stateless; revocation is a separate online check.

The PolicyGrant is signed by the human's DID key (e.g., `did:key`, `did:web`, `did:xrpl`).
Key resolution follows the DID document or HTTPS well-known endpoint for the issuer domain.
This allows a merchant to verify the human's signature without a prior relationship with
that human's identity infrastructure.

This creates a **delegated authorization model** in which:
1. The human signs once (the PolicyGrant) using their DID key
2. The AI agent pre-loads the signed authorization chain
3. Merchants verify the full chain locally, without calling back to the human
4. The human can cancel the delegation via the revocation endpoint at any time

---

## 3. BudgetAuthorization

A BudgetAuthorization defines a **bounded financial envelope** for a machine session.

It may specify:

- maximum spend amount
- currency
- allowed payment rails
- allowed assets
- destination allowlists
- session expiration
- budget scope (SESSION, DAY, VEHICLE, FLEET, or TRIP)

The BudgetAuthorization ensures that machines and agents cannot exceed predefined spending limits.

**TRIP scope extension:** For human-to-agent delegation spanning multiple days or sessions
(e.g., a multi-day travel itinerary), the BudgetAuthorization may carry a `TRIP` scope.
Under TRIP scope, the session authority (AI agent) maintains a cumulative spend counter across
all sessions within the trip, and the maximum spend applies across the full trip rather than
resetting per session. This enables a single pre-authorized budget to cover a bounded multi-day
spending period without requiring the human principal to re-authorize for each session.

---

## 4. PaymentAuthorization

A PaymentAuthorization authorizes a **specific payment decision**.

It binds:

- session identifier
- policy hash
- selected payment rail
- selected asset
- price or quote
- settlement parameters

The PaymentAuthorization also contains a **SettlementIntentHash** — a canonical cryptographic hash of the payment intent. This hash is computed from the settlement parameters (rail, asset, destination, amount) and embedded in the PaymentAuthorization before signing.

The SettlementIntentHash ensures that the **executed settlement transaction matches exactly the authorized intent**. Any modification to the settlement parameters after signing invalidates the hash and fails verification.

---

## 5. Deterministic Verification

A verifier can validate a payment by checking:

1. Policy constraints (via PolicyGrant)
2. BudgetAuthorization limits
3. PaymentAuthorization approval
4. SettlementIntentHash match (hash of executed settlement must match the hash embedded in the PaymentAuthorization)
5. settlement transaction fields

Verification does **not require contacting the issuer**.

This enables deterministic, third‑party verification of autonomous payments.

---

# Machine Wallet Guardrails

An important extension of the invention is the concept of **Machine Wallet Guardrails**.

A machine wallet must refuse to sign a payment transaction unless:

1. a valid PolicyGrant exists
2. a valid BudgetAuthorization exists
3. a valid PaymentAuthorization exists
4. the requested transaction matches the authorized intent

This converts MPCP from a payment protocol into a **wallet-level safety framework for autonomous machines**.

## Wallet Dual Role

In an autonomous deployment, the machine wallet plays two roles in the authorization pipeline:

**Session authority** — the wallet itself creates and signs the BudgetAuthorization before the session begins. This establishes the session-level spending envelope and binds it to the PolicyGrant. The BudgetAuthorization is signed with a key held by the wallet acting in this capacity.

**Payment decision service** — for each payment request, the wallet evaluates the request against the loaded authorization chain and, if approved, creates and signs the PaymentAuthorization locally. No central approval service is contacted. The signed PaymentAuthorization serves as cryptographic proof of the wallet's approval.

This dual role enables fully offline machine payments: the wallet pre-authorizes its own spending envelope and makes per-payment decisions locally, with the complete artifact chain available for post-settlement verification by any verifier.

---

# Identity Binding

MPCP artifacts carry `issuer` and `issuerKeyId` fields identifying the policy authority.
Key resolution follows one of two mechanisms:

1. **HTTPS well-known** — verifiers fetch `https://{issuer-domain}/.well-known/mpcp-keys.json`
   to retrieve the authority's public key as a JWK document
2. **DID resolution** — when `issuer` is a DID URI (e.g., `did:key:z6Mk...`, `did:web:...`,
   `did:xrpl:...`), verifiers resolve the DID document to obtain the public key

This allows MPCP artifacts to be linked to real-world entities such as:

- fleet operators (did:web or domain)
- charging networks
- infrastructure providers
- **human principals** (did:key, did:web, did:xrpl, or other DID methods)
- AI agent platforms (did:web)

For human-to-agent delegation, the `issuer` is the human's DID key. Merchants can verify
the human's PolicyGrant signature against the DID-derived public key without any prior
relationship with that human's identity provider.

MPCP verification itself **does not require identity resolution to be online** — pre-configured
keys (JWK) may substitute for live DID resolution in offline environments.

### On-Chain Identity and Revocation

An optional extension replaces the hosted `revocationEndpoint` with an **on-chain artifact**:

- The PolicyGrant is minted as a non-transferable non-fungible token (NFT) on a distributed
  ledger (e.g., XRPL, Hedera)
- The NFT's existence indicates the grant is active; burning the NFT constitutes revocation
- Merchants check NFT existence on-chain rather than querying a hosted endpoint
- The `grantId` maps to the NFT token identifier; an optional `anchorRef` field carries the
  on-chain pointer

This eliminates the requirement for the human principal to maintain an always-on revocation
service, while preserving the ability to cancel the delegation.

---

# Public Ledger Anchoring (Optional)

The SettlementIntentHash may optionally be anchored to a distributed ledger.

Possible anchoring networks include:

- XRPL
- Hedera
- Ethereum
- timestamping networks

Anchoring provides:

- public auditability
- timestamp proofs
- tamper resistance

Ledger anchoring is optional and outside the core protocol.

---

# Primary Novel Elements

The potentially patentable elements include:

1. policy evaluation producing a bounded authorization grant
2. layered cryptographic authorization artifacts for autonomous payments
3. bounded machine wallet financial autonomy
4. wallet dual-role: issuing its own spending authorization and making per-payment decisions locally
5. deterministic post-settlement verification
6. intent‑hash binding between authorization and settlement
7. wallet‑level refusal to sign unauthorized transactions
8. **human DID-signed policy grant delegating bounded spending authority to an AI agent** — a human principal signs a PolicyGrant using a decentralized identifier key, establishing the AI agent as session authority and payment decision service within defined bounds
9. **purpose-constrained agent spending via signed artifact** — the `allowedPurposes` field in the signed PolicyGrant restricts the merchant categories the agent may pay, enforced by the agent as session authority with the constraint recorded in the signed artifact chain for audit
10. **revocable bounded delegation with stateless verifier** — the delegation includes a `revocationEndpoint` enabling the human to cancel mid-execution; the MPCP verifier pipeline remains stateless and synchronous; revocation is checked by the merchant as a separate step
11. **multi-session cumulative budget scope (TRIP)** — a BudgetAuthorization scope covering multiple sessions for a single trip or project, with cumulative spend tracked by the session authority across sessions
12. **on-chain NFT-backed policy grant revocation** — the PolicyGrant is represented as a non-transferable NFT on a distributed ledger; NFT existence indicates an active grant; NFT burn constitutes revocation, eliminating the need for a hosted revocation service

---

# Prior Art Comparison

The following comparison helps distinguish MPCP from adjacent systems and payment models.

## Traditional Payment Systems

Traditional payment systems typically rely on:

- direct human approval
- centralized merchant APIs
- issuer-side authorization checks
- opaque internal risk logic

These systems do not provide a portable, cryptographically verifiable authorization chain that can be replayed and validated independently after settlement.

MPCP differs by introducing:

- layered authorization artifacts
- machine wallet guardrails
- deterministic post-settlement verification
- explicit binding between policy, budget, payment approval, and settlement intent

---

## Private-Key Wallet Control

Conventional crypto wallets allow any holder of a private key to initiate transactions.

This model provides:

- direct cryptographic control
- simple transaction signing

However, it does not provide:

- bounded machine spending
- policy-derived authorization layers
- independent verification of why a payment was allowed
- wallet-level refusal based on structured authorization artifacts

MPCP differs by requiring a machine wallet to validate layered policy and budget artifacts before signing a payment transaction.

---

## Smart Contract Escrow and Contract-Based Payments

Smart contracts can enforce payment logic on-chain.

These systems may provide:

- programmable disbursement logic
- conditional release of funds
- on-chain auditability

However, they do not inherently provide:

- off-chain machine wallet guardrails
- portable policy artifacts
- a layered authorization chain spanning policy, budget, and payment approval
- deterministic verification of an off-chain authorization chain independent of contract execution

MPCP differs by operating as an authorization framework above the settlement layer rather than requiring settlement logic to be embedded directly in a smart contract.

---

## OAuth and API Authorization Systems

OAuth and similar authorization systems allow applications to access services based on delegated authorization.

These systems provide:

- scoped authorization
- delegated access rights
- token-based service access

However, they do not provide:

- financial authorization for autonomous machine payments
- budget-bound payment permissions
- deterministic settlement verification
- a chain of signed artifacts tied to payment execution

MPCP is conceptually analogous to OAuth in that it introduces structured delegated authorization, but it applies this concept to autonomous financial transactions rather than API access.

---

## Account Abstraction and Smart Wallets

Account abstraction systems and programmable wallets may support:

- custom transaction policies
- session keys
- spending controls
- automated signing rules

However, these mechanisms generally keep authorization logic inside the wallet or contract implementation itself.

They do not necessarily provide:

- portable authorization artifacts
- cross-system interoperability
- recipient-verifiable authorization chains
- deterministic post-settlement replay independent of wallet internals

MPCP differs by externalizing authorization into explicit, replayable artifacts that can be validated by any verifier.

---

## Existing Machine or Agent Payment Systems

Emerging agentic and machine payment systems may support:

- automated payment initiation
- hosted wallet logic
- workflow-controlled disbursement
- API-driven machine commerce

However, these approaches generally do not provide a formalized artifact chain that separates:

- policy authorization
- bounded budget authorization
- specific payment authorization
- deterministic settlement verification

MPCP differs by defining a layered cryptographic authorization architecture specifically for bounded autonomous payments.

---

## Summary of Novelty

MPCP introduces a distinct combination of elements not typically found together in prior systems:

1. layered payment authorization artifacts
2. bounded machine wallet autonomy
3. deterministic verification against settlement intent
4. post-settlement replay without issuer contact
5. wallet-level refusal to sign unauthorized machine transactions

This combination forms the basis of the invention and distinguishes MPCP from adjacent payment, wallet, and authorization systems.

---

# Example Use Cases

## Example 1: Electric Vehicle Charging

1. A fleet operator defines a Policy allowing charging along a defined route.
2. The fleet policy authority evaluates the Policy and issues a PolicyGrant to the vehicle.
3. The vehicle wallet creates and signs a BudgetAuthorization for the charging session, binding it to the PolicyGrant.
4. A charging station generates a price quote.
5. The vehicle wallet evaluates the quote against the PolicyGrant and BudgetAuthorization, then creates and signs a PaymentAuthorization for the quote, embedding a SettlementIntentHash.
6. The vehicle wallet signs and submits the payment transaction only if all constraints are satisfied.
7. Settlement occurs on a payment rail.
8. The charging operator verifies the settlement against the full authorization artifact chain.
9. The transaction can later be independently verified by any party holding the artifact chain.

---

## Example 2: Human-to-Agent Travel Budget Delegation

1. A human principal (Alice) defines a travel policy allowing hotel, flight, and transport
   payments up to $800 for a 3-day trip, with restricted merchant categories and a revocation
   capability.
2. Alice signs a PolicyGrant using her DID key, specifying `allowedPurposes`, a
   `revocationEndpoint`, and `expiresAt` covering the trip duration.
3. Alice's AI trip-planning agent receives the signed PolicyGrant and creates and signs a
   BudgetAuthorization with TRIP scope, binding it to the PolicyGrant and establishing the
   $800 spending envelope across all sessions of the trip.
4. For each booking (hotel, train, car rental), the agent checks whether the merchant category
   is in `allowedPurposes` and whether the cumulative spend would remain within the budget.
5. For approved bookings, the agent creates and signs a PaymentAuthorization embedding a
   SettlementIntentHash.
6. Merchants verify the full authorization chain locally without contacting Alice.
7. When a restaurant booking is attempted, the agent refuses to sign because `travel:dining`
   is not in `allowedPurposes` — no payment artifact is created.
8. When Alice cancels the trip, she revokes the PolicyGrant via the `revocationEndpoint`;
   subsequent merchant revocation checks return `revoked: true` and refuse payment.
9. A post-trip audit verifies all settlement bundles independently from the signed artifact chain.

---

# Patent Scope Strategy

The patent should focus on **the method of constrained autonomous payments and bounded delegation**, not the MPCP specification.

Protected concepts may include:

- layered authorization artifacts
- machine wallet guardrails
- intent hashing
- deterministic payment verification
- bounded autonomous machine payments
- **human DID-signed delegation to AI agents with purpose constraints and revocation**
- **multi-session cumulative budget scope for trip/project delegation**
- **on-chain NFT-backed policy grant with burn-based revocation**

The MPCP specification itself should remain open.

---

# Defensive Patent Strategy

The purpose of filing a provisional patent is to:

- establish a priority date
- prevent hostile patent claims
- preserve commercialization options
- protect the architecture

The MPCP protocol and reference implementation remain open source.

---

# Potential Claim Skeleton

A method for controlling autonomous machine payments comprising:

1. evaluating a machine payment policy to produce a bounded authorization grant defining permitted payment parameters
2. issuing a bounded budget authorization artifact defining session spending limits, linked to the authorization grant
3. generating a signed payment authorization artifact for a specific payment decision, embedding a canonical settlement intent hash
4. verifying a settlement transaction against the artifact chain without contacting the issuer

Additional claim directions:

- wherein the machine wallet itself issues the budget authorization and payment authorization (wallet dual-role)
- wherein settlement intent hash is computed as a deterministic canonical hash of the settlement parameters and embedded in the payment authorization before signing
- wherein verification is performed locally by a recipient without network access to the issuing authority
- wherein the policy grant is signed by a human principal using a decentralized identifier key and specifies one or more permitted merchant categories, and wherein the AI agent enforces said categories before creating a payment authorization
- wherein the policy grant specifies a revocation endpoint and wherein service providers query said endpoint before accepting payment, independently of the stateless verifier pipeline
- wherein the budget authorization specifies a multi-session cumulative scope and wherein the session authority maintains a cumulative spend counter across sessions
- wherein the policy grant is represented as a non-transferable non-fungible token on a distributed ledger and wherein revocation is effected by burning said token

---

# Future Extensions

Possible additional claims may include:

- machine wallet guardrail enforcement
- multi‑policy authorization intersections
- fleet policy aggregation
- autonomous infrastructure payments
- cross‑rail settlement verification
- on-chain policy document anchoring (policy hash anchored to HCS or XRPL for tamper-evident audit)
- transferable vs non-transferable policy grant NFTs and associated authorization semantics
- policy grant issuance as a service (Policy Authority as a managed service with DID management, revocation hosting, and spend tracking)
- enterprise IAM integration for AI agent spending authorization (organizational policy authority issuing grants to AI agents on behalf of employees)

---

# Notes

This document is **not a legal patent filing**.

It serves as an internal concept document supporting preparation of a provisional patent application.

A qualified patent attorney should draft the final filing.