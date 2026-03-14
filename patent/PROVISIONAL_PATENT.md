# PROVISIONAL PATENT APPLICATION — DRAFT

**Title:** Method and System for Cryptographically Bounded Autonomous Payment Authorization
with Human-to-Agent Delegation and Decentralized Identity

> **DRAFT — FOR ATTORNEY REVIEW**
> This document is a working draft prepared to establish conceptual priority and support
> preparation of a formal provisional patent application. It is not a legal filing.
> A qualified patent attorney should review and finalize before submission.

---

## Field of the Invention

The present invention relates to authorization systems for autonomous financial transactions,
and more particularly to a layered cryptographic artifact pipeline for bounding the spending
authority of autonomous machines and AI agents under human-defined policies.

---

## Background

Autonomous systems — including autonomous vehicles, delivery robots, IoT infrastructure,
and AI software agents — increasingly perform financial transactions without direct human
approval at the moment of payment. Existing payment infrastructure provides no satisfactory
framework for **bounded autonomous financial action**:

- **Human authorization** requires a human in the loop at payment time, defeating autonomy
- **Centralized API authorization** creates single points of failure and requires online
  connectivity at payment time
- **Smart contract escrow** requires settlement logic to be embedded in on-chain contracts,
  limiting flexibility and increasing cost
- **Private key wallet control** permits unlimited spending by any holder of the key, with
  no policy enforcement or post-settlement auditability

None of these approaches provide a portable, cryptographically verifiable authorization chain
that can be (a) enforced locally by the spending agent, (b) verified independently by the
payment recipient, and (c) audited after settlement by any third party.

The problem is made more acute by the emergence of AI agents that act on behalf of human
principals. A human may wish to delegate a bounded spending authority — e.g., $800 for a
multi-day travel itinerary — to an AI agent, with the ability to:

- restrict spending to specific merchant categories
- cancel the delegation mid-execution
- obtain a verifiable audit trail of every payment made under the delegation

No existing system provides all of these properties in combination.

---

## Summary of the Invention

The present invention provides a **layered cryptographic authorization pipeline** for
autonomous machine and AI agent payments. The pipeline constrains payment execution through
a sequence of cryptographically signed artifacts, each narrowing the permitted scope of the
next. The full artifact chain enables independent post-settlement verification by any party.

In a first aspect, the invention provides a method comprising:

1. evaluating a payment policy to produce a signed PolicyGrant defining the permitted
   payment parameters for a principal (fleet operator, enterprise, or human)
2. issuing a signed BudgetAuthorization defining a bounded spending envelope, linked to
   the PolicyGrant by a canonical policy hash
3. generating a signed PaymentAuthorization for each specific payment decision, embedding a
   canonical SettlementIntentHash computed from the settlement parameters
4. verifying a settled payment locally against the artifact chain without contacting the
   policy authority

In a second aspect, the invention extends the pipeline to **human-to-agent delegation**:

1. a human principal signs a PolicyGrant using a decentralized identifier (DID) key,
   specifying: the AI agent as subject, permitted merchant categories (`allowedPurposes`),
   a revocation endpoint, and a budget scope covering a multi-day trip or project (`TRIP`)
2. the AI agent acts as session authority, creating and signing a BudgetAuthorization that
   pre-loads the spending envelope for the full delegation period
3. the AI agent enforces purpose constraints at each payment decision before signing a
   PaymentAuthorization; decisions outside the permitted categories are refused without
   creating any authorization artifact
4. merchants verify the full artifact chain (human DID signature through to settlement)
   locally without contacting the human; optionally, they query a revocation endpoint before
   accepting payment
5. the human cancels the delegation by posting to the revocation endpoint; the MPCP verifier
   pipeline remains stateless

In a third aspect, the invention provides **on-chain policy grant representation**:

1. the PolicyGrant is minted as a non-transferable NFT on a distributed ledger
2. NFT existence indicates an active grant; NFT burn constitutes revocation
3. merchants check NFT existence on-chain, eliminating the need for a hosted revocation service
4. an optional `anchorRef` field in the PolicyGrant carries the on-chain token identifier

---

## Detailed Description

### 1. Authorization Artifact Pipeline

#### 1.1 Policy and PolicyGrant

A Policy defines the rules governing a principal's payment behavior. It is evaluated by a
policy authority to produce a **PolicyGrant** artifact.

The PolicyGrant is a signed data structure that specifies:

- `grantId` — unique identifier for this delegation
- `policyHash` — SHA-256 hash of the canonical policy document (`SHA256("MPCP:Policy:1.0:" + canonicalJson(policy))`)
- `allowedRails` — permitted payment rails
- `allowedAssets` — permitted on-chain assets
- `expiresAt` — ISO 8601 expiration
- `issuer` — identifier of the policy authority (DID URI or domain)
- `issuerKeyId` — key identifier within the authority's key set
- `signature` — signature over `SHA256("MPCP:PolicyGrant:1.0:" + canonicalJson(grantPayload))`

**Human delegation extensions:**

- `allowedPurposes` — optional array of permitted merchant category strings (e.g.,
  `["travel:hotel", "travel:flight"]`); agent-enforced; recorded in artifact for audit
- `revocationEndpoint` — optional URL; contract: `GET {endpoint}?grantId={id}` returns
  `{ "revoked": boolean, "revokedAt": "ISO8601" }`

The policy authority may be a fleet operator, enterprise IAM service, or a **human principal
identified by a decentralized identifier**. When the issuer is a DID, verifiers resolve the
public key via DID document resolution or HTTPS well-known endpoint.

#### 1.2 BudgetAuthorization (Signed)

The BudgetAuthorization defines a bounded financial envelope linked to a PolicyGrant.

Key fields:

- `budgetId` — unique identifier
- `grantId` — references the PolicyGrant
- `policyHash` — must match PolicyGrant.policyHash
- `maxAmountMinor` — maximum authorized spend in atomic units
- `budgetScope` — SESSION | DAY | VEHICLE | FLEET | **TRIP**
- `allowedRails`, `allowedAssets`, `destinationAllowlist`
- `expiresAt`

The BudgetAuthorization is signed as: `sign(SHA256("MPCP:SBA:1.0:" + canonicalJson(authorization)))`.

**TRIP scope:** When `budgetScope` is `TRIP`, the maximum spend applies cumulatively across
all sessions of a single trip or project. The session authority (AI agent) maintains a running
cumulative spend counter; each payment reduces the remaining allowance.

#### 1.3 PaymentAuthorization (Signed) with SettlementIntentHash

For each approved payment, the session authority creates a PaymentAuthorization embedding
a **SettlementIntentHash** — a canonical hash of the settlement parameters:

```
intentHash = SHA256("MPCP:SettlementIntent:1.0:" + canonicalJson(settlementIntent))
```

where `settlementIntent` includes: rail, asset, amount, destination, and creation timestamp.

The PaymentAuthorization binds: session ID, policy hash, budget ID, rail, asset, amount,
destination, and the intent hash. It is signed by the session authority.

Any modification to the settlement parameters after signing invalidates the intent hash and
fails verification.

#### 1.4 Deterministic Verification

A verifier validates a payment by checking, in order:

1. PolicyGrant signature (against resolved public key)
2. BudgetAuthorization signature and linkage to PolicyGrant (matching `policyHash`, `grantId`)
3. PaymentAuthorization signature and linkage to BudgetAuthorization (matching `budgetId`, `policyHash`)
4. SettlementIntentHash match (hash of executed settlement parameters matches embedded hash)
5. Policy constraints (rails, assets, destinations, budget remaining)

Verification does not require contacting the policy authority. The complete artifact chain
is self-contained and replayable by any party.

---

### 2. Human-to-Agent Delegation

The authorization pipeline is applied to the case where a human principal delegates bounded
spending authority to an AI agent.

#### 2.1 Delegation Flow

```
Human (DID key) → signs PolicyGrant (allowedPurposes, revocationEndpoint, TRIP scope)
                → AI Agent receives PolicyGrant
                → AI Agent creates and signs BudgetAuthorization (TRIP scope)
                → For each payment:
                    Agent checks allowedPurposes → if not permitted, refuse
                    Agent checks cumulative budget → if exceeded, refuse
                    Agent creates and signs PaymentAuthorization
                    Merchant verifies artifact chain locally
                    Merchant optionally checks revocationEndpoint
```

#### 2.2 Purpose Constraints

The human specifies `allowedPurposes` in the signed PolicyGrant. The AI agent, acting as
session authority, checks each payment request against this list before signing a
PaymentAuthorization. Requests with a merchant category not in `allowedPurposes` are refused;
no authorization artifact is created for refused requests.

The `allowedPurposes` field is recorded in the signed PolicyGrant, making the human's intent
part of the verifiable artifact chain.

#### 2.3 Revocation

The PolicyGrant may specify a `revocationEndpoint`. The endpoint contract is:

```
GET {revocationEndpoint}?grantId={grantId}
Response: { "revoked": boolean, "revokedAt": "ISO8601" }
```

The MPCP verifier pipeline does not call this endpoint — it remains stateless and synchronous.
Merchants call the endpoint as a separate step before accepting payment. If the grant is
revoked, the merchant refuses payment regardless of the artifact chain's validity.

This separation preserves the offline-capable nature of the core verifier while allowing
online revocation for deployed systems where connectivity is available.

---

### 3. On-Chain Policy Grant Representation

#### 3.1 NFT-Backed Policy Grant

As an optional extension, the PolicyGrant may be minted as a non-transferable NFT on a
distributed ledger (e.g., XRPL NFToken, Hedera HTS):

- The NFT is minted with `tfTransferable = false` (non-transferable flag)
- The NFT's URI field references the full PolicyGrant JSON (via IPFS, HCS, or HTTPS)
- The `grantId` in the PolicyGrant corresponds to the NFT token identifier
- An optional `anchorRef` field in the PolicyGrant carries the on-chain token reference

**Revocation:** The human burns the NFT to revoke the grant. Merchants check NFT existence
on-chain; a non-existent or burned token indicates revocation. This eliminates the requirement
for the human principal to maintain an always-on revocation service.

#### 3.2 Policy Document Anchoring

The policy document (from which `policyHash` is derived) may be published to a distributed
consensus service (e.g., Hedera HCS) to create a tamper-evident, timestamped public record.
Any party can verify the `policyHash` by fetching the corresponding ledger message and
recomputing the hash.

---

### 4. Machine Wallet Dual Role

In an autonomous deployment, the session authority (machine wallet or AI agent) plays two
roles in the authorization pipeline:

**Session authority** — creates and signs the BudgetAuthorization, establishing the spending
envelope for the session or trip.

**Payment decision service** — for each payment request, evaluates the request against the
loaded authorization chain and, if approved, creates and signs the PaymentAuthorization
locally. No central approval service is contacted.

This dual role enables payments in offline or low-connectivity environments: the agent
pre-authorizes its own spending envelope and makes per-payment decisions locally, with the
complete artifact chain available for post-settlement verification.

---

## Claims (Informal — For Attorney Review)

**Claim 1.** A method for authorizing autonomous payments comprising:
- evaluating a payment policy to produce a signed authorization grant specifying permitted payment parameters;
- issuing a signed budget authorization artifact defining a maximum spending amount, linked to the authorization grant by a cryptographic policy hash;
- generating, for a specific payment decision, a signed payment authorization artifact embedding a canonical hash of the settlement intent parameters;
- verifying, by a payment recipient, the settlement transaction against the artifact chain without contacting the policy authority.

**Claim 2.** The method of Claim 1, wherein the authorization grant is signed by a human principal using a decentralized identifier key and specifies one or more permitted merchant categories, and wherein a session authority enforces said categories before generating a payment authorization artifact.

**Claim 3.** The method of Claim 1, wherein the authorization grant specifies a revocation endpoint, and wherein a payment recipient queries said endpoint before accepting a payment, independently of the stateless verification pipeline.

**Claim 4.** The method of Claim 1, wherein the budget authorization specifies a multi-session cumulative scope, and wherein the session authority maintains a running cumulative expenditure across sessions against the maximum spending amount.

**Claim 5.** The method of Claim 1, wherein the session authority that issues the budget authorization is also the payment decision service that generates payment authorization artifacts, enabling fully local authorization without contacting a central service.

**Claim 6.** The method of Claim 1, wherein the authorization grant is minted as a non-transferable token on a distributed ledger, and wherein revocation of the grant is effected by burning said token.

**Claim 7.** The method of Claim 1, wherein the settlement intent hash is computed as a deterministic canonical hash of settlement parameters including rail, asset, amount, and destination, and wherein any modification to said parameters after signing causes verification to fail.

**Claim 8.** A system for bounded autonomous agent spending comprising:
- a policy authority that signs an authorization grant using a decentralized identifier key and specifies permitted merchant categories and a revocation endpoint;
- an AI agent that acts as session authority, creates a budget authorization with multi-session scope, and enforces permitted merchant categories before generating payment authorizations;
- a verification pipeline that validates the artifact chain locally, wherein the pipeline is stateless with respect to revocation.

---

## Abstract

A method and system for cryptographically bounded autonomous payment authorization. A layered
pipeline of signed artifacts — PolicyGrant, BudgetAuthorization, PaymentAuthorization, and
SettlementIntent — constrains the spending authority of autonomous machines and AI agents.
Each artifact in the chain cryptographically references the previous, enabling any recipient
to verify a payment's authorization independently and after settlement without contacting the
issuing authority. Extensions provide: (a) human DID-signed delegation to AI agents with
purpose constraints and revocable authorization; (b) multi-session cumulative budget scopes
for multi-day or project-based delegations; (c) on-chain NFT-backed policy grants where
revocation is effected by burning the token. The verification pipeline remains stateless;
revocation checks are performed by merchants as a separate step.

---

## Notes

This document is a **working draft** for attorney review. It is not a legal filing.

Key open questions for attorney review:
1. Claim breadth vs. prior art in OAuth/OIDC delegation and smart wallet session keys
2. Whether the human-DID-signed delegation claims are sufficiently distinct from existing VC/VP authorization schemes
3. Strategy for the NFT-backed revocation claim given existing NFT collateral and access control prior art
4. Whether to pursue a single broad provisional or separate applications for the core pipeline vs. the human-to-agent delegation extension
