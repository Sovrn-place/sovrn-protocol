# Sovrn Protocol

**The first open standard for portable identity credentials across sovereign economic zones.**

Sovrn Protocol defines how identity credentials are issued, verified, and ported across independent special economic zones, free ports, and smart cities. A KYC verification completed in one zone can be recognized by another zone without re-verification - while preserving user consent, privacy, and regulatory sovereignty.

There is no existing standard for portable identity credentials across independent economic zones. Sovrn Protocol is the first attempt to define one.

---

## How this repository is organized

Sovrn Protocol has two layers:

**The open protocol (Tier 1)** - Fully specified in this repository under Apache 2.0. Everything a developer or standards body needs to implement, verify, or integrate against: the `did:sovrn:` method, W3C VC 2.0 credential schemas, the `KycAdapter` interface, the credential presentation format, the cross-zone verification flow, standards alignment, and on-chain anchoring.

**The proprietary platform (Tier 2)** - Described here so implementers know what surrounds the open protocol, but not specified in this repository. Covers reputation, the intent lifecycle, the ledger and audit layer, escrow and settlement, federation onboarding, and AI-assisted review. Full specifications are available to zone partners under agreement.

> The open protocol handles **identity and credentials**. The proprietary platform handles **operations, payments, and intelligence**. Both are part of how Sovrn works. Only the first is open-source.

---

# Tier 1 - The Open Protocol

The seven sections below are the complete specification of the open Sovrn Protocol. They are licensed under Apache 2.0.

## 1. Universal ID and DID Method (`did:sovrn:`)

The Universal ID is the identity anchor in Sovrn Protocol. Every credential, reputation score, and cross-zone presentation is tied to a `.si` identity.

**DID method:** `did:sovrn:`
**Namespace:** `.si` (e.g., `alex.si`)
**Format:** `did:sovrn:{uuid}`

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

The `.si` namespace is proprietary to Sovrn. The `did:sovrn:` resolution method and the identity anchor schema are part of this open protocol.

**DID document example:**

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

## 2. W3C VC 2.0 Credential Schemas

All credentials conform to the [W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/).

**Required fields:**

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

**Credential types:**

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

**Full credential example:**

```json
{
  "@context": ["https://www.w3.org/ns/credentials/v2"],
  "type": ["VerifiableCredential", "SovrnIdentityCredential"],
  "issuer": "did:sovrn:itana-ng",
  "issuanceDate": "2026-03-14T09:12:00Z",
  "expirationDate": "2028-03-14T09:12:00Z",
  "credentialSubject": {
    "id": "did:sovrn:a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "type": "KYC_ENHANCED",
    "level": 2,
    "issuingZone": "itana-ng",
    "hash": "sha256:a3f8c1e9b2d4..."
  },
  "proof": {
    "type": "Ed25519Signature2020",
    "created": "2026-03-14T09:12:00Z",
    "verificationMethod": "did:sovrn:itana-ng#key-1",
    "proofPurpose": "assertionMethod",
    "proofValue": "z58..."
  }
}
```

## 3. KycAdapter Interface

The `KycAdapter` is the integration contract. Any identity provider - open or licensed - implements this interface to issue credentials conforming to the Sovrn Protocol. Sovrn defines the standard; providers implement it.

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

An implementation is a pure translation layer. It takes provider-native outputs and emits Sovrn-compliant credentials. Sovrn's core never talks to any provider directly.

**Reference implementations** exist in both tracks:

- **Open track** - Privado ID, Holonym / Human ID, zkPassport, Anon Aadhaar
- **Licensed track** - Sumsub, Persona

All adapters emit credentials indistinguishable at the protocol layer. A credential issued via Privado ID and a credential issued via Sumsub are the same shape, carry the same hash format, and verify identically in any Sovrn-compliant zone.

## 4. Credential Presentation Format

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
  reputation?: {
    total: number                   // 0-100
    tier: 'Explorer' | 'Pioneer' | 'Ambassador' | 'Sovereign'
    zoneCount: number
    computedAt: string
    hash: string
  }

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
- Reputation tier and score (only if the user authorized sharing)
- Consent metadata: authorized fields, timestamp, consent hash
- Presenting and receiving zone identifiers

**What is never included:**

- Raw PII (name, address, document numbers, date of birth)
- Original documents (passports, utility bills, selfies)
- KYC session details or provider internals
- Non-consented credentials
- Any field not explicitly authorized in `consent.authorizedFields`

A `CredentialPresentation` is a signed envelope. The receiving zone verifies the issuing zone's signature on each credential against the federation registry before acting on it.

## 5. Cross-Zone Verification Flow

When a user presents credentials from Zone A to Zone B:

1. **Consent** - The user explicitly authorizes sharing a specific subset of credentials. The authorization is recorded with a SHA-256 `consentHash`. No field is transmitted without being listed in `consent.authorizedFields`.
2. **Delivery** - Zone B receives a `CredentialPresentation` payload (section 4). Signatures are verified against the federation registry.
3. **Resolution** - Zone B's verifier resolves the presentation against one of three paths:
   - **Accept** - Skip re-verification. Record the presentation with consent metadata.
   - **Delta** - Accept the base credential, request only additional checks not covered by the original.
   - **Fresh** - Reject the presentation and require a new verification (e.g. when zone policy requires it regardless of prior credentials).

**What crosses zones:**

- Attestations (credential type, level, issuing zone, dates, hash)
- Reputation scores and tiers (only if consented)
- Compliance status
- Consent records

**What never crosses zones:**

- Original documents
- Raw PII
- KYC session details
- Provider internals
- Any non-consented field

The protocol is explicit about what is and is not portable. Raw personal data stays with the original issuer and its KYC provider.

## 6. Standards Alignment

Sovrn Protocol is designed to be interoperable with existing and emerging identity standards.

| Standard | Role |
|---|---|
| [W3C VC 2.0](https://www.w3.org/TR/vc-data-model-2.0/) | Credential data model |
| [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html) | Credential issuance |
| [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html) | Credential presentation |
| [ISO 18013-5 (mdoc)](https://www.iso.org/standard/69084.html) | Mobile document format |
| EUDI Wallet | Target compatibility by Dec 2026 |
| FATF Recommendations | AML/CFT compliance framing for tiered verification |

An OID4VCI / OID4VP profile for Sovrn credentials is a 2026 target. EUDI Wallet compatibility is a December 2026 target.

## 7. On-Chain Anchoring

Every credential's SHA-256 hash is written to a contract on Base L2. This creates a tamper-evident verification layer without exposing any credential contents.

- **Track 1 (database-backed)** - Credential storage and lookup happen in a standard relational database for speed. Suitable for immediate verification in the issuing zone.
- **Track 2 (on-chain anchored)** - The credential hash is mirrored to a Base L2 contract. Any verifier can independently check that a presented credential matches the on-chain hash, making forgery or retroactive alteration detectable without a trusted database.

**What is anchored:** Credential hashes (section 2), the terminal hashes of ledger event chains (see Tier 2, Ledger & Audit Layer), and reputation score hashes (see Tier 2, Reputation Protocol).

**What is not anchored:** Raw credentials, PII, application contents, or any field that would compromise privacy if exposed on-chain. Only hashes.

Both layers exist in the reference architecture. Verification works against Track 1 alone; Track 2 provides cross-zone and cross-time trust.

---

# Tier 2 - Additional Protocol Layers

Sovrn Protocol does not stop at credentials. The proprietary platform that implements the open protocol also runs the operational, financial, and intelligence layers that zones need to actually process applications, settle payments, and hold regulators accountable.

These layers are described below so implementers and integrators understand what surrounds the open protocol. **Full specifications are available to zone partners, implementers, and standards bodies under agreement.** Contact **contactus@sovrnplace.com**.

The open protocol handles identity and credentials. The proprietary platform handles operations, payments, and intelligence.

## 8. Reputation Protocol

Sovrn computes a portable reputation score for every identity, derived from verifiable on-platform activity across five dimensions: **tenure**, **financial**, **compliance**, **cross-zone**, and **engagement**. Each dimension scores 0-20 for a total of 0-100, placing the identity in one of four tiers: **Explorer** (0-24), **Pioneer** (25-49), **Ambassador** (50-74), or **Sovereign** (75-100). A multi-zone bonus (up to +10) rewards credentials spanning multiple zones.

The output schema that zones receive is fully defined in the Credential Presentation Format above (section 4, `reputation` field). **The scoring methodology is proprietary. The output schema is open** - any zone, regulator, or auditor reading a reputation score across the network reads the same fields in the same format.

Full scoring specification available under zone partnership.

## 9. Intent Lifecycle

An **Intent** is the platform term for an application a user submits to a zone - residency, business incorporation, tax filing, any service the zone offers. Every zone processes applications through a shared state machine: `DRAFT` -> `SUBMITTED` -> `AI_REVIEW` -> `IDENTITY_VERIFIED` -> `PENDING_AUTHORITY` -> `APPROVED` -> `ACTIVE`, with terminal `REJECTED`, `CANCELLED`, and `EXPIRED` states.

Each transition runs **gate checks** (payment escrowed, KYC credentials at required tier, mandatory local checks passed, AI confidence threshold met) and emits a ledger event. This gives every application a tamper-evident lifecycle.

Full intent state machine specification, gate definitions, and transition rules available under zone partnership.

## 10. Ledger and Audit Layer

Every significant action in the platform creates a `LedgerEvent`: identity changes, credential issuances, intent transitions, payments, reputation recomputations. Events are append-only, each carrying a SHA-256 hash over its canonicalized payload. Hashes form a chain per entity - every event references the previous event's hash - making the log tamper-evident without a trusted database.

Terminal chain hashes are anchored on Base L2 (see section 7). This gives zones, regulators, and auditors an independently verifiable audit trail.

Full ledger event schema, category taxonomy, and hash chain specification available under partnership.

## 11. Escrow and Settlement

Payments to zones are held in programmable escrow between submission and service delivery. The escrow protocol supports multiple payment rails (Stripe, Paystack, Flutterwave, Circle / USDC) and automates release, refund, and fee splits. Escrow states follow a defined lifecycle (`PENDING` -> `IN_ESCROW` -> `SETTLED` / `REFUNDED`) with release driven by intent resolution and refunds triggered by rejection, cancellation, or timeout.

The default fee split is 85 percent to the zone, 15 percent to the network. Zones may negotiate alternative terms.

Commercial terms and the full escrow specification are covered by zone agreement. Contact **contactus@sovrnplace.com**.

## 12. Federation Onboarding

Zones joining the Sovrn network provide a federation configuration covering zone identity, KYC policy (minimum tier, cross-zone acceptance, mandatory local checks), payment setup (providers, currencies, fee split), services offered, administrative policy (review mode, AI assistance, thresholds), and reputation policy. Once accepted, the zone's DID is registered in the federation registry and it can issue credentials and receive cross-zone presentations.

**The full onboarding specification is available for prospective zone partners.** Contact **contactus@sovrnplace.com**.

## 13. AI-Assisted Review

Sovrn provides AI-assisted application screening that runs automated compliance assessment and surfaces applications above a confidence threshold for auto-advance, while routing borderline and high-risk applications to human authority review. The system is explicitly **human-in-the-loop for rejections** - no application is automatically rejected without a human decision.

The review layer handles document quality checks, sanctions screening interpretation, consistency scoring, and per-zone policy enforcement.

Architecture details, model selection, and prompt contracts are available under NDA.

---

## Repository structure

```
/schemas       W3C VC 2.0 credential JSON-LD schemas (Tier 1)
/adapters      KycAdapter interface + reference implementations (Tier 1)
/protocol      Cross-zone verification protocol specification (Tier 1)
/examples      Example credentials, presentations, adapter implementations
/docs          Full specification documents (Tier 1 only)
```

Directories will be populated as Tier 1 components are extracted from the reference implementation and published here.

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

Sovrn Protocol is currently maintained by [Sovrn](https://sovrn.place). Schema changes to Tier 1 go through an RFC process - open an issue with the proposed change, its rationale, and a backwards compatibility analysis. Breaking changes require a major version bump.

As adoption grows, governance will expand to include implementers, zones, and participating standards bodies.

---

## Security

Security issues: email **contactus@sovrnplace.com**. Do not open public issues for security vulnerabilities.

Responsible disclosure reports receive acknowledgement within 72 hours.

---

## License

**Tier 1 only.** The specifications, schemas, and interface definitions for the seven Tier 1 sections of this document are licensed under [Apache 2.0](LICENSE).

**Tier 2 is described, not licensed.** The additional protocol layers (reputation, intent lifecycle, ledger and audit, escrow and settlement, federation onboarding, AI-assisted review) are part of the proprietary Sovrn platform. Their descriptions in this README are informational. Full specifications and implementation details are available to zone partners and qualified implementers under agreement.

---

## Contributing

Sovrn Protocol is maintained by [Sovrn](https://sovrn.place). If you are:

- Building identity infrastructure for an economic zone
- Implementing a KYC adapter against this specification
- Contributing to a related standard (W3C VC, OID4VCI, ISO mdoc, EUDI)
- Researching cross-border credential portability

Open an issue or reach out. This is a first attempt at a standard that does not yet exist. Feedback from implementers, zones, and standards bodies shapes what it becomes.

For zone partnerships, full Tier 2 specifications, or commercial questions: **contactus@sovrnplace.com**.
