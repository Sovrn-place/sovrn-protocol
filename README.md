# Sovrn Protocol

An open specification for portable identity credentials across independent economic zones.

Sovrn Protocol defines how identity credentials are issued, verified, and ported across special economic zones, free ports, and smart cities. A KYC verification completed in one zone can be recognized by another zone without re-verification, while preserving user consent, privacy, and regulatory sovereignty.

---

## Scope

This specification defines:

- A decentralized identifier method (`did:sovrn:`) and the `.si` namespace
- W3C VC 2.0 credential schemas for zone-issued attestations
- A provider-agnostic adapter interface for KYC and verification providers
- A credential presentation format for cross-zone sharing
- A cross-zone verification flow with explicit consent semantics
- Alignment with existing and emerging identity standards
- On-chain credential storage for tamper-evident verification

Compatible identity providers include Privado ID, Holonym / Human ID, zkPassport, Anon Aadhaar, Sumsub, Persona, and any other provider that implements the adapter interface.

---

## 1. Universal ID and DID Method (`did:sovrn:`)

The Universal ID is the identity anchor in Sovrn Protocol. Every credential, reputation score, and cross-zone presentation is tied to a `.si` identity.

**DID method:** `did:sovrn:`
**Namespace:** `.si` (e.g., `alex.si`)
**Format:** `did:sovrn:{uuid}`

A `did:sovrn:{uuid}` resolves to the subject's public identity anchor containing:

- `.si` name
- Identity level (`LIGHT` -> `UNIVERSAL_ID` -> `VERIFIED`)
- Active credentials (types and issuing zones, never raw data)
- Reputation score and tier
- SHA-256 identity hash

The `.si` identity sits above the KYC layer. A user can hold a `.si` identity without any KYC credentials. KYC credentials from multiple providers and zones attach to the same `.si` anchor as they are obtained. Cross-zone portability works at the identity level, not at the credential level: Zone B recognizes `alex.si`, not a specific verification session.

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

---

## 2. Credential Schemas (W3C VC 2.0)

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

Every credential carries a SHA-256 hash over its canonicalized payload. This hash is the verification anchor for both database-stored and on-chain-stored credentials.

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

---

## 3. KycAdapter Interface

The `KycAdapter` is the integration contract. Any identity provider, whether open-source or commercial, can implement this interface to issue credentials conforming to the Sovrn Protocol.

```typescript
interface KycAdapter {
  /** Unique identifier for this adapter */
  readonly providerId: string

  /** Human-readable name */
  readonly providerName: string

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

An implementation is a pure translation layer. It takes provider-native outputs and emits Sovrn-compliant credentials. The interface is provider-agnostic: compatible implementations can wrap zero-knowledge verification tools (Privado ID, Holonym, zkPassport, Anon Aadhaar) or regulated KYC providers (Sumsub, Persona) and the resulting credentials are indistinguishable at the protocol layer.

---

## 4. Credential Presentation Format

When a user presents credentials from one zone to another, the exact payload handed to the receiving zone is defined by this schema. No field outside the payload may be transmitted.

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

**Included:** Credential type, level, issuing zone, dates, hash; reputation tier and score if consented; consent metadata; presenting and receiving zone identifiers.

**Never included:** Raw PII, original documents, verification session details, provider internals, non-consented credentials, or any field not explicitly authorized in `consent.authorizedFields`.

A `CredentialPresentation` is a signed envelope. The receiving zone verifies the issuing zone's signature on each credential before acting on it.

---

## 5. Cross-Zone Verification Flow

When a user presents credentials from Zone A to Zone B:

1. **Consent** - The user authorizes sharing a specific subset of credentials. The authorization is recorded with a SHA-256 `consentHash`. No field is transmitted without being listed in `consent.authorizedFields`.
2. **Delivery** - Zone B receives a `CredentialPresentation` payload (section 4). Signatures are verified against the federation registry.
3. **Resolution** - Zone B's verifier resolves the presentation against one of three paths:
   - **Accept** - Skip re-verification. Record the presentation with consent metadata.
   - **Delta** - Accept the base credential, request only additional checks not covered by the original.
   - **Fresh** - Reject the presentation and require a new verification (e.g. when zone policy requires it regardless of prior credentials).

**What crosses zones:** attestations, reputation scores if consented, compliance status, consent records, credential hashes.

**What never crosses zones:** original documents, raw PII, verification session details, provider internals, non-consented fields.

Raw personal data remains with the original issuer and its verification provider.

---

## 6. Reputation Schema

The protocol defines an output schema for portable reputation scores. Scoring methodology is implementation-defined; the schema that travels between zones is not.

**Dimensions (each 0-20, total 0-100):**

| Dimension | What it measures |
|---|---|
| `tenure` | Time holding a `.si` identity and active participation |
| `financial` | Payment history, escrow completion, refund rate |
| `compliance` | KYC tier achieved, cross-check results, sanctions status |
| `crossZone` | Breadth of credentials across distinct zones |
| `engagement` | Application completion rate, document quality, responsiveness |

**Tiers (derived from total):**

| Tier | Range |
|---|---|
| Explorer | 0-24 |
| Pioneer | 25-49 |
| Ambassador | 50-74 |
| Sovereign | 75-100 |

The reputation object embedded in a `CredentialPresentation` (section 4, `reputation` field) is the authoritative wire format. Any zone, regulator, or integrator reading a reputation score across the network reads the same fields.

---

## 7. Standards Alignment

Sovrn Protocol is designed to interoperate with existing and emerging identity standards.

| Standard | Role |
|---|---|
| [W3C VC 2.0](https://www.w3.org/TR/vc-data-model-2.0/) | Credential data model |
| [OID4VCI](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html) | Credential issuance |
| [OID4VP](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html) | Credential presentation |
| [ISO 18013-5 (mdoc)](https://www.iso.org/standard/69084.html) | Mobile document format |
| EUDI Wallet | Compatibility targeted for December 2026 |
| FATF Recommendations | Framing for tiered KYC depth |

An OID4VCI / OID4VP profile for Sovrn credentials is planned for 2026.

---

## 8. On-Chain Credential Storage

On-chain credential storage is planned. SHA-256 hashes of credentials, ledger events, and reputation scores will be recorded on-chain for tamper-proof verification. Only hashes are stored on-chain; no credential contents, PII, or application data.

**What will be stored on-chain:** credential hashes, reputation score hashes, terminal hashes of audit chains.

**What will not be stored on-chain:** raw credentials, personal data, application contents, or any field that would compromise privacy if exposed.

Any verifier will be able to independently check that a presented credential matches the on-chain hash, making forgery or retroactive alteration detectable without reliance on a trusted database. The initial deployment target is Base L2, with the contract address and ABI to be published in this repository when deployed.

---

## Repository structure

```
/schemas       W3C VC 2.0 credential JSON-LD schemas
/adapters      KycAdapter interface and reference implementations
/protocol      Cross-zone verification and presentation specifications
/examples      Example credentials, presentations, and adapter implementations
/docs          Full specification documents
```

Directories will be populated as components are extracted and published here.

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

Sovrn Protocol is currently maintained by [Sovrn](https://sovrn.place). Schema changes go through an RFC process: open an issue with the proposed change, its rationale, and a backwards compatibility analysis. Breaking changes require a major version bump.

As adoption grows, governance will expand to include implementers, zones, and participating standards bodies.

---

## Security

Security issues: email **contactus@sovrnplace.com**. Do not open public issues for security vulnerabilities.

Responsible disclosure reports receive acknowledgement within 72 hours.

---

## License

Specifications, schemas, and interface definitions in this repository are licensed under [Apache 2.0](LICENSE).

---

## Platform

Sovrn Protocol is implemented by the Sovrn platform, which additionally provides:

- Zone administration and application management
- AI-assisted compliance review
- Programmable escrow and multi-currency settlement
- Reputation scoring across five dimensions
- Federation onboarding for new zones
- Immutable audit trail with on-chain credential storage

For platform access or zone partnership inquiries: **contactus@sovrnplace.com**

---

## Contributing

If you are building identity infrastructure for an economic zone, implementing a KYC adapter against this specification, contributing to a related standard (W3C VC, OID4VCI, ISO mdoc, EUDI), or researching cross-border credential portability, open an issue or reach out.

Links: [sovrn.place](https://sovrn.place) · [linktr.ee/joinsovrn](https://linktr.ee/joinsovrn) · [newsletter](https://sovrn-newsletter.beehiiv.com/)

Contact: **contactus@sovrnplace.com**
