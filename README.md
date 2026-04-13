# Sovrn Protocol

**The first open standard for portable identity credentials across sovereign economic zones.**

Sovrn Protocol defines how identity credentials are issued, verified, and ported across independent special economic zones, free ports, and smart cities. A KYC verification completed in one zone can be recognized by another zone without re-verification - while preserving user consent, privacy, and regulatory sovereignty.

There is no existing standard for portable identity credentials across independent economic zones. Sovrn Protocol is the first attempt to define one.

---

## What this is

Sovrn Protocol is a specification. It defines:

- **A universal identity anchor** - The `did:sovrn:` method and `.si` namespace that every credential, reputation score, and cross-zone presentation attaches to
- **A credential schema** - A W3C VC 2.0 compliant data model for identity attestations issued by economic zones
- **A reputation protocol** - An open schema for portable reputation scores that travel with a user across zones
- **An adapter interface** - A TypeScript contract (`KycAdapter`) that any identity provider can implement to issue credentials in the Sovrn format
- **A cross-zone verification protocol** - How credentials issued by Zone A are presented to and accepted by Zone B
- **An intent state machine** - The application lifecycle every zone processes through Sovrn
- **A ledger event schema** - An immutable audit trail format for every state change, payment, and credential issuance
- **A federation onboarding spec** - What a zone provides to join the network
- **An escrow protocol** - How payments are held, released, and split between zones and the network
- **An on-chain anchoring layer** - SHA-256 hashes of every credential and ledger event, anchored on Base L2

Sovrn defines the standard. Identity providers implement it. Reference adapter implementations exist for both open verification tools (Privado ID, Holonym / Human ID, zkPassport, Anon Aadhaar) and licensed KYC providers (Sumsub, Persona). All produce identical credentials conforming to this specification.

The protocol is database-backed for speed (Track 1) and on-chain anchored for trust (Track 2). Both layers exist in the reference architecture.

---

## Quick example

A credential issued by a zone under this protocol:

```json
{
  "@context": ["https://www.w3.org/ns/credentials/v2"],
  "type": ["VerifiableCredential", "SovrnIdentityCredential"],
  "issuer": "did:sovrn:itana-ng",
  "credentialSubject": {
    "id": "did:sovrn:user-a3f8c1",
    "type": "KYC_VERIFIED",
    "level": 2,
    "issuingZone": "itana-ng",
    "hash": "sha256:a3f8c1e9b2d4..."
  }
}
```

The minimal adapter contract an identity provider implements:

```typescript
interface KycAdapter {
  readonly providerId: string
  readonly track: 'open' | 'licensed'
  readonly maxTier: 1 | 2 | 3

  startSession(params: SessionParams): Promise<SessionHandle>
  getStatus(sessionId: string): Promise<SessionStatus>
  handleWebhook(payload: unknown): Promise<AdapterResult>
}
```

That's the protocol in 20 lines. Full specification below.

---

## Specification

### 1. Universal ID (.si Namespace)

The Universal ID is the identity anchor in Sovrn Protocol. Every credential, reputation score, and cross-zone presentation is tied to a `.si` identity.

**Format:** `did:sovrn:{uuid}`
**Namespace:** `.si` (e.g., `alex.si`)
**Resolution:** `did:sovrn:{uuid}` resolves to the user's public identity anchor containing:

- `.si` name
- Identity level (`LIGHT` -> `UNIVERSAL_ID` -> `VERIFIED`)
- Active credentials (types and issuing zones, never raw data)
- Reputation score and tier
- SHA-256 identity hash

The `.si` identity sits **above** the KYC layer. Users claim a `.si` name first, then KYC credentials attach to that identity when triggered by actions. This separation means:

- A user can hold a `.si` identity without any KYC (`UNIVERSAL_ID` level)
- KYC credentials from multiple providers and zones all attach to the same `.si` anchor
- Cross-zone portability works at the identity level, not the credential level - Zone B recognizes `alex.si`, not a specific Sumsub session

The `.si` namespace is proprietary to Sovrn. The DID resolution method and identity anchor schema are part of this open protocol.

**Example DID document:**

```json
{
  "@context": ["https://www.w3.org/ns/did/v1", "https://sovrn.place/ns/identity/v1"],
  "id": "did:sovrn:a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "authentication": [{
    "type": "Ed25519VerificationKey2020",
    "publicKeyMultibase": "z6Mkf5..."
  }],
  "service": [{
    "type": "SovrnIdentityAnchor",
    "serviceEndpoint": "https://sovrn.place/api/identity/resolve/alex.si"
  }],
  "sovrnMetadata": {
    "siName": "alex.si",
    "level": "VERIFIED",
    "credentialCount": 3,
    "reputationTier": "Pioneer",
    "identityHash": "sha256:a3f8c1..."
  }
}
```

### 2. Reputation Protocol

Reputation is portable across zones. It is computed from verifiable on-platform activity and attached to a `.si` identity. When a user presents credentials cross-zone, the reputation score travels with the presentation.

**The scoring algorithm is proprietary. The output schema is open.** Any zone admin, regulator, or auditor reading a reputation score across the network reads the same fields in the same format.

**Dimensions (each 0-20, total 0-100):**

| Dimension | What it measures |
|---|---|
| `tenure` | Time holding a `.si` identity and active participation across zones |
| `financial` | Payment history, escrow completion, refund rate |
| `compliance` | KYC tier achieved, cross-check results, sanctions status |
| `crossZone` | Breadth of credentials across distinct zones |
| `engagement` | Application completion rate, document quality, responsiveness |

**Tiers (from total score):**

| Tier | Range |
|---|---|
| Explorer | 0-24 |
| Pioneer | 25-49 |
| Ambassador | 50-74 |
| Sovereign | 75-100 |

**Multi-zone bonus (added to total, capped):**

| Distinct zones with credentials | Bonus |
|---|---|
| 2 | +5 |
| 3 | +8 |
| 4 | +9 |
| 5 or more | +10 (cap) |

**ReputationScore schema (what a zone receives during presentation):**

```typescript
interface ReputationScore {
  /** Subject DID */
  subject: string
  /** Total score, 0-100 (after multi-zone bonus, capped) */
  total: number
  /** Tier derived from total */
  tier: 'Explorer' | 'Pioneer' | 'Ambassador' | 'Sovereign'
  /** Per-dimension breakdown */
  dimensions: {
    tenure: number
    financial: number
    compliance: number
    crossZone: number
    engagement: number
  }
  /** Number of distinct zones contributing credentials */
  zoneCount: number
  /** Multi-zone bonus applied */
  multiZoneBonus: number
  /** Last recomputation timestamp (ISO 8601) */
  computedAt: string
  /** SHA-256 hash of the canonicalized score payload */
  hash: string
}
```

Zones consume the score, not the raw signals behind it. A zone admin seeing `tier: 'Ambassador'` knows what that means across the network without needing access to payment history, KYC sessions, or prior applications.

### 3. Credential Schema (W3C VC 2.0)

All credentials conform to the [W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/).

Required fields:

| Field | Description |
|---|---|
| `@context` | Must include `https://www.w3.org/ns/credentials/v2` |
| `type` | Must include `VerifiableCredential` and `SovrnIdentityCredential` |
| `issuer` | DID of the issuing zone (`did:sovrn:{zone-id}`) |
| `credentialSubject.id` | DID of the subject (`did:sovrn:{user-uuid}`) |
| `credentialSubject.type` | One of the credential types defined below |
| `credentialSubject.level` | Verification tier (1-3) where applicable |
| `credentialSubject.issuingZone` | Zone code of the issuer |
| `credentialSubject.hash` | SHA-256 hash of the canonicalized credential payload |

Every credential includes a SHA-256 hash computed from the canonicalized payload. This hash is the bridge between the database layer and the on-chain anchoring layer. The same hash is written to a Base L2 contract, making the credential tamper-evident without exposing any credential contents.

### 4. Credential Types

| Type | Description | Tier |
|---|---|---|
| `KYC_VERIFIED` | Identity verified via government ID + liveness | 1 |
| `KYC_ENHANCED` | Adds address proof and PEP screening | 2 |
| `KYC_FULL` | Adds source of funds and enhanced due diligence | 3 |
| `RESIDENCY_ACTIVE` | Active residency in a zone | - |
| `BUSINESS_REGISTERED` | Company incorporated in a zone | - |
| `TAX_COMPLIANT` | Tax obligations met for a filing period | - |
| `PAYMENT_CONFIRMED` | Payment settled through a protocol-compliant rail | - |

Tiered types (`KYC_*`) map to FATF-aligned verification depth. Non-tiered types represent zone-scoped attestations.

### 5. KycAdapter Interface

The `KycAdapter` is the integration contract. Any identity provider - open or licensed - implements this interface to issue credentials conforming to the Sovrn Protocol.

```typescript
interface KycAdapter {
  /** Unique identifier for this adapter */
  readonly providerId: string

  /** Human-readable name */
  readonly providerName: string

  /** Whether this is an open or licensed provider */
  readonly track: 'open' | 'licensed'

  /** Maximum KYC tier this adapter supports */
  readonly maxTier: 1 | 2 | 3

  /** Start a verification session */
  startSession(params: {
    userId: string
    tier: 1 | 2 | 3
    zoneCode: string
    resumeId?: string
  }): Promise<{
    sessionId: string
    accessToken: string
    expiresAt: string
  }>

  /** Check session status (polling fallback) */
  getStatus(sessionId: string): Promise<{
    status: 'pending' | 'in_progress' | 'approved' | 'rejected' | 'review'
    tier?: 1 | 2 | 3
    credentialHash?: string
  }>

  /** Handle webhook from provider */
  handleWebhook(payload: unknown): Promise<{
    sessionId: string
    result: 'approved' | 'rejected' | 'review'
    tier: 1 | 2 | 3
    credentialData: Record<string, unknown>
  }>
}
```

An implementation is a pure translation layer: it takes provider-native outputs and emits Sovrn-compliant credentials. Sovrn's core never talks to any provider directly.

### 6. Reference Implementations

The protocol is provider-agnostic. Reference adapters exist for:

**Open track** - adapters that wrap open or decentralized verification tools:

- Privado ID (zero-knowledge proofs)
- Holonym / Human ID (humanity verification)
- zkPassport (passport ZK proofs)
- Anon Aadhaar (Indian identity, ZK)

**Licensed track** - adapters that wrap regulated KYC providers:

- Sumsub
- Persona

All adapters emit credentials indistinguishable at the protocol layer. A credential issued via Privado ID and a credential issued via Sumsub are the same shape, carry the same hash format, and verify identically in any Sovrn-compliant zone.

### 7. Federation Configuration

Each zone configures its verification policy:

```typescript
interface FederationKycConfig {
  /** Minimum KYC tier for basic access */
  minTier: 1 | 2 | 3
  /** Accept cross-zone credentials? */
  acceptsCrossZone: boolean
  /** Which zones' credentials are accepted */
  acceptedZones: string[] | ['*']
  /** Checks that must always be done locally */
  mandatoryLocalChecks: string[]
}
```

This lets a zone accept credentials from some peers, require fresh verification from others, and preserve non-waivable local checks (e.g. domestic sanctions screening) regardless of presented credentials.

### 8. Federation Onboarding

To join the Sovrn network, a zone provides a complete `FederationConfig`. This is the "how to become a Sovrn zone" specification - it captures everything a zone declares about itself, its regulatory stance, its payment setup, and its administrative policies.

```typescript
interface FederationConfig {
  /** Zone identity */
  zoneId: string                    // e.g. 'itana-ng'
  zoneName: string                  // e.g. 'Itana Digital Free Zone'
  country: string                   // ISO 3166-1 alpha-2
  regulator: string                 // Name of the regulatory authority
  legalFramework: string            // Reference to the enabling statute or SEZ act
  did: string                       // did:sovrn:{zoneId}
  publicKey: string                 // multibase-encoded zone signing key

  /** KYC policy (see FederationKycConfig above) */
  kyc: FederationKycConfig

  /** Payment policy */
  payment: {
    providers: Array<'stripe' | 'paystack' | 'flutterwave' | 'circle' | string>
    currencies: string[]            // ISO 4217 + 'USDC'
    feeSplit: {
      zone: number                  // 0.85 typical
      network: number               // 0.15 typical
    }
    escrowTimeoutDays: number       // auto-refund threshold
  }

  /** Services offered by this zone */
  services: Array<
    | 'RESIDENCY'
    | 'BUSINESS_INCORPORATION'
    | 'TAX_FILING'
    | 'BANKING'
    | 'REAL_ESTATE'
    | 'EVENT_ACCESS'
  >

  /** Admin policy */
  admin: {
    reviewMode: 'manual' | 'ai_assisted' | 'auto'
    autoAdvanceThreshold: number    // AI confidence score (0-1) for auto-advance
    aiProvider: 'gemini-flash' | 'claude' | 'none'
  }

  /** Reputation policy */
  reputation: {
    /** Minimum reputation tier accepted for cross-zone credentials */
    minTierAccepted: 'Explorer' | 'Pioneer' | 'Ambassador' | 'Sovereign'
    /** Whether this zone contributes to reputation computation */
    contributes: boolean
  }
}
```

A zone joining the network submits this configuration to Sovrn. Once accepted, the zone's DID is registered in the federation registry and it can begin issuing credentials and receiving cross-zone presentations.

### 9. Credential Presentation Format

When a user presents credentials from Zone A to Zone B, the exact payload handed to Zone B's verifier is defined by this schema. Nothing not listed in the payload may be transmitted.

```typescript
interface CredentialPresentation {
  /** Subject's DID */
  subject: string                   // did:sovrn:{uuid}
  /** Credentials included in this presentation */
  credentials: Array<{
    type: SovrnCredentialType
    level?: 1 | 2 | 3
    issuingZone: string
    issuedAt: string                // ISO 8601
    expiresAt?: string              // ISO 8601
    hash: string                    // sha256:...
  }>
  /** Attached reputation (if consented) */
  reputation?: ReputationScore
  /** Consent metadata */
  consent: {
    /** What fields the user authorized sharing */
    authorizedFields: string[]
    /** When the user authorized this presentation */
    authorizedAt: string            // ISO 8601
    /** Unique identifier of the presentation event */
    presentationId: string
    /** SHA-256 hash of the canonicalized consent record */
    consentHash: string
  }
  /** Presenting zone (typically the user's home zone) */
  presentingZone: string
  /** Receiving zone */
  receivingZone: string
}
```

**What is included:**

- Credential type, level, issuing zone, issue date, expiry, hash
- Reputation score and tier (only if the user authorized sharing)
- Consent metadata: authorized fields, timestamp, consent hash
- Presenting and receiving zone identifiers

**What is never included:**

- Raw PII (name, address, document numbers, date of birth)
- Original documents (passports, utility bills, selfies)
- KYC session details or provider internals
- Non-consented credentials
- Any field not explicitly authorized in `consent.authorizedFields`

A `CredentialPresentation` is a signed envelope. Zone B verifies the issuing zone's signature on each credential against the federation registry. Zone B records the presentation as a `CredentialPresentation` ledger event (see section 12) and may then resolve it into one of the three verification paths in section 10.

### 10. Cross-Zone Verification Flow

When a user presents credentials from Zone A to Zone B:

1. The user consents to sharing specific credentials (section 9)
2. Zone B receives a `CredentialPresentation` payload as defined above
3. Zone B's admin resolves the presentation against one of three paths:
   - **Accept** - skip re-verification, create a `CredentialPresentation` ledger event with consent metadata
   - **Delta** - accept the base credential, request only additional checks not covered by the original
   - **Fresh** - reject the presentation and require a new verification (e.g. when zone policy requires it regardless of prior credentials)

**What crosses zones:** Attestations, reputation scores, compliance status, credential hashes, consent records
**What never crosses:** Original documents, raw PII, KYC session details, provider internals

The protocol is explicit about what is and is not portable. Raw personal data stays with the original issuer and its KYC provider.

### 11. Intent State Machine

An **Intent** is the protocol term for an application a user submits to a zone - residency, business incorporation, tax filing, any service the zone offers. Every zone processes applications through Sovrn using the same state machine. This is the "operating system" layer for zone operations.

**States:**

```
DRAFT              User is filling out the application
SUBMITTED          User has submitted for review
AI_REVIEW          Application under AI-assisted screening
IDENTITY_VERIFIED  KYC credentials attached and verified
PENDING_AUTHORITY  Awaiting human review from the zone authority
APPROVED           Authority has approved, awaiting activation
ACTIVE             Application is active (e.g. residency granted)
REJECTED           Rejected at any review stage
CANCELLED          User cancelled
EXPIRED            Timed out without resolution
```

**Transitions:**

```
DRAFT             -> SUBMITTED          (user action)
SUBMITTED         -> AI_REVIEW          (auto; gate: payment escrowed)
AI_REVIEW         -> IDENTITY_VERIFIED  (auto; gate: KYC credentials present)
AI_REVIEW         -> REJECTED           (auto; gate: AI screening failed)
IDENTITY_VERIFIED -> PENDING_AUTHORITY  (auto; gate: zone review required)
IDENTITY_VERIFIED -> APPROVED           (auto; gate: AI confidence >= threshold)
PENDING_AUTHORITY -> APPROVED           (authority action)
PENDING_AUTHORITY -> REJECTED           (authority action)
APPROVED          -> ACTIVE             (auto; gate: instrument issued)
SUBMITTED         -> CANCELLED          (user action)
*                 -> EXPIRED            (auto; gate: zone timeout reached)
```

**Gate checks** run before each automatic transition. If a gate fails, the intent stays in its current state and the failure is recorded. Gates include: payment escrow verified, credentials attached at required tier, mandatory local checks passed, AI confidence above threshold, zone policy satisfied.

**LedgerEvent emitted on every transition** (see section 12). The event records the from-state, to-state, actor, gate results, and a SHA-256 hash of the transition payload. This makes the application lifecycle a tamper-evident sequence.

### 12. Ledger Event Schema

Every significant action in the protocol creates a `LedgerEvent`. The ledger is the immutable audit trail. Zones, regulators, and auditors all read the same format.

```typescript
interface LedgerEvent {
  /** Unique event identifier (ULID) */
  id: string
  /** Event type - narrow, specific */
  type: string                      // e.g. 'INTENT_APPROVED', 'PAYMENT_ESCROWED'
  /** Broad category for indexing */
  category: 'IDENTITY' | 'PAYMENT' | 'INTENT' | 'CREDENTIAL' | 'REPUTATION'
  /** Entity the event is about */
  entityId: string
  entityType: 'User' | 'Intent' | 'Payment' | 'Credential' | 'Zone'
  /** Actor that triggered the event */
  actorId: string                   // did:sovrn:{uuid}
  actorType: 'User' | 'Zone' | 'System'
  /** Zone context (if applicable) */
  zoneId?: string
  /** Payload snapshot - type-specific */
  metadata: Record<string, unknown>
  /** ISO 8601 timestamp */
  timestamp: string
  /** SHA-256 hash over the canonicalized event */
  hash: string
  /** Previous event hash in the entity's chain (hash chain) */
  prevHash?: string
}
```

**Categories:**

- `IDENTITY` - Universal ID claim, level upgrades, credential attach/detach
- `PAYMENT` - Escrow, release, refund, fee split
- `INTENT` - State transitions in the intent state machine
- `CREDENTIAL` - Issuance, presentation, revocation
- `REPUTATION` - Score recomputation, tier changes

Events are append-only. Nothing is ever deleted or modified. Each event hash incorporates the previous event's hash for the same entity, forming a tamper-evident chain. The terminal hash of a chain is what gets anchored on Base L2 in Track 2.

A zone's full history, a user's full identity journey, and a payment's full lifecycle are all reconstructible from the ledger alone.

### 13. Payment Escrow Protocol

Payments in Sovrn are held in escrow between submission and service delivery. The escrow protocol defines the lifecycle, the release conditions, and the fee split between the zone and the network.

**States:**

```
PENDING      User has initiated payment, not yet confirmed
IN_ESCROW    Payment confirmed and held pending service delivery
SETTLED      Released: 85% to zone, 15% to network
REFUNDED     Returned to user (rejection or timeout)
FAILED       Payment failed before reaching escrow
```

**Transitions:**

```
PENDING    -> IN_ESCROW  (provider confirms capture)
PENDING    -> FAILED     (provider rejects)
IN_ESCROW  -> SETTLED    (intent transitions to APPROVED)
IN_ESCROW  -> REFUNDED   (intent transitions to REJECTED or CANCELLED)
IN_ESCROW  -> REFUNDED   (zone escrow timeout reached - auto-refund)
```

**Default fee split:** 85% zone, 15% network. Zones may negotiate alternative splits via the `FederationConfig.payment.feeSplit` field.

**Escrow timeout:** Each zone declares `escrowTimeoutDays` in its federation config. If an intent does not resolve to `APPROVED` or `REJECTED` within the window, the payment auto-refunds and the intent transitions to `EXPIRED`.

**LedgerEvent on every transition.** Every escrow state change emits a `PAYMENT` category ledger event with the amount, currency, provider, and fee split recorded.

The escrow protocol is payment-provider-agnostic. The reference implementation supports Stripe, Paystack, Flutterwave, and Circle (USDC). Any provider that can hold and release funds on webhook can be wired in.

### 14. On-Chain Anchoring

Every credential's SHA-256 hash, every ledger event's SHA-256 hash, and every reputation score's SHA-256 hash is written to a contract on Base L2. This creates a tamper-evident verification layer without exposing any underlying data.

- **Track 1 (database-backed)** - Credential, ledger, and reputation storage happen in a standard relational database for speed. Suitable for immediate verification in the issuing zone.
- **Track 2 (on-chain anchored)** - The hash is mirrored to a Base L2 contract. Any verifier can independently check that a presented credential, a claimed ledger chain, or a reported reputation score matches its on-chain hash. Forgery, retroactive alteration, or log tampering becomes detectable without a trusted database.

Both layers exist in the reference architecture. Verification works against Track 1 alone; Track 2 provides cross-zone and cross-time trust.

---

## Standards alignment

Sovrn Protocol is designed to be interoperable with existing and emerging identity standards.

| Standard | Role |
|---|---|
| [W3C VC 2.0](https://www.w3.org/TR/vc-data-model-2.0/) | Credential data model |
| [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html) | Credential issuance |
| [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html) | Credential presentation |
| [ISO 18013-5 (mdoc)](https://www.iso.org/standard/69084.html) | Mobile document format |
| EUDI Wallet | Target compatibility by Dec 2026 |
| FATF Recommendations | AML/CFT compliance framing for tiered verification |

---

## Status

| Component | Status |
|---|---|
| Universal ID (.si namespace, did:sovrn method) | Defined |
| W3C VC 2.0 credential schema | Defined |
| Reputation schema and tier model | Defined |
| KycAdapter interface | Defined |
| Federation config and onboarding schema | Defined |
| Credential presentation format | Defined |
| Cross-zone verification protocol | Defined |
| Intent state machine | Defined |
| Ledger event schema | Defined |
| Escrow protocol | Defined |
| SHA-256 hash on every credential, event, and score | Defined |
| Sumsub reference adapter | Implemented |
| Persona reference adapter | Implemented |
| Open-track reference adapters (Privado, Holonym, zkPassport, Anon Aadhaar) | Specified, not yet open-sourced |
| On-chain anchoring (Base L2 contract) | Specified (Track 2) |
| EUDI Wallet compatibility | Target Dec 2026 |
| OID4VCI / OID4VP profile | Target 2026 |

---

## Repository structure

```
/schemas       W3C VC 2.0 credential JSON-LD schemas, reputation schema, ledger event schema
/adapters      KycAdapter interface + reference implementations
/protocol      Cross-zone verification, intent state machine, escrow, federation onboarding specs
/examples      Example credentials, presentations, ledger chains, adapter implementations
/docs          Full specification documents
```

Directories will be populated as components are extracted from the reference implementation and published here.

---

## Implementing an adapter

To add support for a new identity provider:

1. Implement the `KycAdapter` interface in `/adapters/<provider-id>/`
2. Map the provider's session lifecycle to the four methods (`startSession`, `getStatus`, `handleWebhook`)
3. Translate the provider's verification output into a `SovrnIdentityCredential` at the correct tier
4. Compute the SHA-256 hash over the canonicalized credential payload
5. Submit a pull request with the adapter and at least one end-to-end example

Adapters must be pure translation layers. They must not modify the credential schema, invent new types, or bypass the hash computation.

---

## Governance

Sovrn Protocol is currently maintained by [Sovrn](https://sovrn.place). Schema changes go through an RFC process - open an issue with the proposed change, its rationale, and a backwards compatibility analysis. Breaking changes require a major version bump.

As adoption grows, governance will expand to include implementers, zones, and participating standards bodies.

---

## Security

Security issues: email **contactus@sovrnplace.com**. Do not open public issues for security vulnerabilities.

Responsible disclosure reports receive acknowledgement within 72 hours.

---

## License

Specifications, schemas, and interface definitions in this repository are licensed under [Apache 2.0](LICENSE).

---

## Contributing

Sovrn Protocol is maintained by [Sovrn](https://sovrn.place). If you are:

- Building identity infrastructure for an economic zone
- Implementing a KYC adapter against this specification
- Contributing to a related standard (W3C VC, OID4VCI, ISO mdoc, EUDI)
- Researching cross-border credential portability

Open an issue or reach out. This is a first attempt at a standard that does not yet exist. Feedback from implementers, zones, and standards bodies shapes what it becomes.
