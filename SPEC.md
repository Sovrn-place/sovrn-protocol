# Sovrn Protocol Specification

**Status:** Draft
**Version:** 0.1
**Editors:** Sovrn core team
**License:** Apache 2.0

## Table of contents

1. [Universal ID and DID Method](#1-universal-id-and-did-method)
2. [Credential Schemas (W3C VC 2.0)](#2-credential-schemas-w3c-vc-20)
3. [KycAdapter Interface](#3-kycadapter-interface)
4. [Credential Presentation Format](#4-credential-presentation-format)
5. [Cross-Zone Verification Flow](#5-cross-zone-verification-flow)
6. [Reputation Schema](#6-reputation-schema)
7. [Standards Alignment](#7-standards-alignment)
8. [On-Chain Credential Storage](#8-on-chain-credential-storage)
9. [Canonicalization and Hashing](#9-canonicalization-and-hashing)

---

## 1. Universal ID and DID Method

The Universal ID is the identity anchor in Sovrn Protocol. Every credential, reputation score, and cross-zone presentation is tied to a `.si` identity.

**DID method:** `did:sovrn:`
**Method-specific-id format:** UUIDv4, lowercase, hyphenated (per RFC 4122 §3)
**Example:** `did:sovrn:a1b2c3d4-e5f6-4890-abcd-ef1234567890`
**Namespace:** `.si` (e.g., `alex.si`) - see [§1.1 Namespace](#11-namespace)

A `did:sovrn:{uuid}` resolves to an identity anchor containing:

- `.si` name
- Identity level - one of `LIGHT`, `UNIVERSAL_ID`, `VERIFIED`
- Active credential types and issuing zones (never raw credential data)
- Reputation score and tier
- SHA-256 identity hash

The `.si` identity sits above the KYC layer. A user can hold a `.si` identity without any KYC credentials. KYC credentials from multiple providers and zones attach to the same `.si` anchor as they are obtained. Cross-zone portability operates at the identity level, not the credential level: Zone B recognizes `alex.si`, not a specific verification session.

### 1.1 Namespace

The `.si` namespace in this protocol is an **internal Sovrn identity namespace**, not the DNS country-code TLD for Slovenia (operated by ARNES). A `.si` name such as `alex.si` is not a DNS hostname and MUST NOT be resolved via DNS. Implementations displaying `.si` names to users should make this distinction visually clear where ambiguity is possible.

### 1.2 DID Resolution

Resolution of `did:sovrn:{uuid}` in v0.1 is performed via the federation registry at `https://sovrn.place/api/identity/resolve/{did}`.

### 1.3 DID Document

```json
{
  "@context": [
    "https://www.w3.org/ns/did/v1",
    "https://sovrn.place/ns/identity/v1"
  ],
  "id": "did:sovrn:a1b2c3d4-e5f6-4890-abcd-ef1234567890",
  "authentication": [{
    "id": "did:sovrn:a1b2c3d4-e5f6-4890-abcd-ef1234567890#key-1",
    "type": "Ed25519VerificationKey2020",
    "controller": "did:sovrn:a1b2c3d4-e5f6-4890-abcd-ef1234567890",
    "publicKeyMultibase": "z6Mkf5..."
  }],
  "service": [{
    "id": "did:sovrn:a1b2c3d4-e5f6-4890-abcd-ef1234567890#anchor",
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

The `sovrnMetadata` property is defined under the `https://sovrn.place/ns/identity/v1` JSON-LD context (forthcoming).

---

## 2. Credential Schemas (W3C VC 2.0)

All credentials conform to the [W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/).

### 2.1 Required fields

| Field | Description |
|-------|-------------|
| `@context` | MUST include `https://www.w3.org/ns/credentials/v2` and `https://sovrn.place/ns/credentials/v1` |
| `type` | MUST include `VerifiableCredential` and `SovrnIdentityCredential` |
| `issuer` | DID of the issuing zone (`did:sovrn:{zone-id}`) |
| `validFrom` | ISO 8601 timestamp (VC 2.0; replaces `issuanceDate` from VC 1.1) |
| `validUntil` | ISO 8601 timestamp (optional) |
| `credentialSubject.id` | DID of the subject (`did:sovrn:{user-uuid}`) |
| `credentialSubject.type` | One of the credential types in §2.2 |
| `credentialSubject.level` | Verification tier (1-3) where applicable |
| `credentialSubject.issuingZone` | Zone code of the issuer |
| `credentialSubject.hash` | SHA-256 hash over the JCS-canonicalized credential payload, prefixed `sha256:` |
| `proof` | Data Integrity proof (see §2.3) |

The `credentialSubject.hash` and all Sovrn-specific terms are defined under the `https://sovrn.place/ns/credentials/v1` JSON-LD context (forthcoming). Implementations performing strict JSON-LD processing MUST include this context.

### 2.2 Credential types

| Type | Description | Tier |
|------|-------------|------|
| `KYC_VERIFIED` | Identity verified via government ID + liveness | 1 |
| `KYC_ENHANCED` | Adds address proof and PEP screening | 2 |
| `KYC_FULL` | Adds source of funds and enhanced due diligence | 3 |
| `RESIDENCY_ACTIVE` | Active residency in a zone | - |
| `BUSINESS_REGISTERED` | Company incorporated in a zone | - |
| `TAX_COMPLIANT` | Tax obligations met for a filing period | - |
| `PAYMENT_CONFIRMED` | Payment settled through a protocol-compliant rail | - |

Tiered types (`KYC_*`) map to FATF-aligned verification depth. Non-tiered types represent zone-scoped attestations.

### 2.3 Proof format

v0.1 credentials use `DataIntegrityProof` with cryptosuite `eddsa-rdfc-2022` (VC Data Integrity). `Ed25519Signature2020` is accepted for backwards compatibility but is considered legacy and will be removed in v1.0.

### 2.4 Full example

```json
{
  "@context": [
    "https://www.w3.org/ns/credentials/v2",
    "https://sovrn.place/ns/credentials/v1"
  ],
  "type": ["VerifiableCredential", "SovrnIdentityCredential"],
  "issuer": "did:sovrn:itana-ng",
  "validFrom": "2026-03-14T09:12:00Z",
  "validUntil": "2028-03-14T09:12:00Z",
  "credentialSubject": {
    "id": "did:sovrn:a1b2c3d4-e5f6-4890-abcd-ef1234567890",
    "type": "KYC_ENHANCED",
    "level": 2,
    "issuingZone": "itana-ng",
    "hash": "sha256:a3f8c1e9b2d4..."
  },
  "proof": {
    "type": "DataIntegrityProof",
    "cryptosuite": "eddsa-rdfc-2022",
    "created": "2026-03-14T09:12:00Z",
    "verificationMethod": "did:sovrn:itana-ng#key-1",
    "proofPurpose": "assertionMethod",
    "proofValue": "z58..."
  }
}
```

---

## 3. KycAdapter Interface

The `KycAdapter` is the integration contract. Any identity provider can implement this interface to issue credentials conforming to Sovrn Protocol.

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
    expiresAt: string          // ISO 8601
  }>

  /** Check session status (polling fallback) */
  getStatus(sessionId: string): Promise<{
    status: SessionStatus
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

type SessionStatus =
  | 'pending'
  | 'in_progress'
  | 'approved'
  | 'rejected'
  | 'review'
  | 'expired'
  | 'abandoned'

/** Error taxonomy - implementations MUST throw one of these */
class KycAdapterError extends Error {
  constructor(
    public readonly code: KycAdapterErrorCode,
    public readonly providerId: string,
    message: string,
    public readonly retriable: boolean,
  ) { super(message) }
}

type KycAdapterErrorCode =
  | 'PROVIDER_UNAVAILABLE'      // upstream provider is down or unreachable
  | 'INVALID_SESSION'           // session not found, expired, or belongs to another user
  | 'INVALID_WEBHOOK_SIGNATURE' // webhook payload failed signature verification
  | 'TIER_NOT_SUPPORTED'        // requested tier exceeds this adapter's maxTier
  | 'RATE_LIMITED'              // provider rate limit hit; retry after delay
  | 'MALFORMED_PAYLOAD'         // webhook or session payload could not be parsed
  | 'NOT_SUPPORTED'             // operation not supported by this adapter
```

### 3.1 Compatible providers

Compatible identity providers include Privado ID, Holonym / Human ID, zkPassport, Anon Aadhaar, Sumsub, Persona, and any other provider that implements the `KycAdapter` interface.

Both open-source and commercial providers emit W3C VC 2.0 credentials with identical schema and hash semantics. A credential issued via any compliant adapter is indistinguishable from any other at the protocol layer.

### 3.2 Implementation requirements

Adapters MUST be pure translation layers. They MUST NOT modify the credential schema, invent new credential types, or bypass hash computation. Adapters MAY omit methods only for operations the underlying provider does not support (e.g., an adapter with no webhook delivery MUST throw `NOT_SUPPORTED` if `handleWebhook` is called, rather than silently succeeding).

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
    computedAt: string              // ISO 8601
    hash: string                    // sha256:...
  }

  /** Consent metadata */
  consent: {
    authorizedFields: string[]
    authorizedAt: string            // ISO 8601
    presentationId: string
    consentHash: string             // sha256:...
  }

  /** Presenting zone */
  presentingZone: string

  /** Receiving zone */
  receivingZone: string
}
```

**Included:** credential type, level, issuing zone, dates, hash; reputation tier and score if consented; consent metadata; presenting and receiving zone identifiers.

**Never included:** raw PII, original documents, verification session details, provider internals, non-consented credentials, or any field not explicitly authorized in `consent.authorizedFields`.

A `CredentialPresentation` is a signed envelope. The receiving zone MUST verify the issuing zone's signature on each credential before acting on it.

---

## 5. Cross-Zone Verification Flow

When a user presents credentials from Zone A to Zone B:

1. **Consent** - The user authorizes sharing a specific subset of credentials. Authorization is recorded with a SHA-256 `consentHash`. No field is transmitted unless listed in `consent.authorizedFields`.

2. **Delivery** - Zone B receives a `CredentialPresentation` payload (§4). Signatures are verified against the federation registry.

3. **Resolution** - Zone B chooses one of three paths:
   - **Accept** - Skip re-verification. Record the presentation with consent metadata.
   - **Delta** - Accept the base credential; request only additional checks not covered by the original.
   - **Fresh** - Reject the presentation and require new verification.

**What crosses zones:** attestations, reputation scores (if consented), compliance status, consent records, credential hashes.

**What never crosses zones:** original documents, raw PII, verification session details, provider internals, non-consented fields.

---

## 6. Reputation Schema

The protocol defines the wire format for portable reputation scores.

### 6.1 Dimensions (each 0-20, total 0-100)

| Dimension | What it measures |
|-----------|-----------------|
| `tenure` | Time holding a .si identity and active participation |
| `financial` | Payment history, escrow completion, refund rate |
| `compliance` | KYC tier achieved, cross-check results, sanctions status |
| `crossZone` | Breadth of credentials across distinct zones |
| `engagement` | Application completion rate, document quality, responsiveness |

### 6.2 Tiers

| Tier | Range |
|------|-------|
| Explorer | 0-24 |
| Pioneer | 25-49 |
| Ambassador | 50-74 |
| Sovereign | 75-100 |

---

## 7. Standards Alignment

| Standard | Role | Status |
|----------|------|--------|
| [W3C VC 2.0](https://www.w3.org/TR/vc-data-model-2.0/) | Credential data model | Implemented |
| [W3C DID Core](https://www.w3.org/TR/did-core/) | DID method framing | Implemented |
| [W3C VC Data Integrity](https://www.w3.org/TR/vc-data-integrity/) | Proof format (`eddsa-rdfc-2022`) | Implemented |
| [JCS (RFC 8785)](https://datatracker.ietf.org/doc/html/rfc8785) | Canonicalization for hashing | Implemented |
| [OID4VCI 1.0](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html) | Credential issuance | Planned — v0.2 |
| [OID4VP 1.0](https://openid.net/specs/openid-4-verifiable-presentations-1_0.html) | Credential presentation | Planned — v0.2 |
| [ISO 18013-5](https://www.iso.org/standard/69084.html) | Mobile document format (mdoc) | Planned — v0.2 |
| [EUDI Wallet](https://digital-strategy.ec.europa.eu/en/policies/eudi-wallet-toolbox) | Compatibility target | Planned — v0.2 |
| [FATF Recommendations](https://www.fatf-gafi.org/en/recommendations.html) | Framing for tiered KYC depth | Implemented |

---

## 8. On-Chain Credential Storage

On-chain storage is planned. SHA-256 hashes of credentials and reputation scores will be recorded on-chain for tamper-evident verification. Only hashes are stored on-chain; no credential contents, PII, or application data.

**Will be stored on-chain:** credential hashes, reputation score hashes, terminal hashes of audit chains.

**Will not be stored on-chain:** raw credentials, personal data, application contents, or any field that would compromise privacy if exposed.

Any verifier will be able to independently check that a presented credential matches the on-chain hash, making forgery or retroactive alteration detectable. Contract address and ABI will be published in this repository at deployment.

---

## 9. Canonicalization and Hashing

All credential hashes in Sovrn Protocol are computed as follows:

1. Serialize the credential payload (`credentialSubject` + `issuer` + `validFrom` + `validUntil` + `type`) as a JSON value.
2. Canonicalize with [JSON Canonicalization Scheme (JCS, RFC 8785)](https://datatracker.ietf.org/doc/html/rfc8785).
3. Compute SHA-256 over the UTF-8 bytes of the canonicalized output.
4. Prefix with `sha256:` and lowercase hex-encode the digest.

The `proof` block is excluded from the hashed payload (the hash must be computed before signing).

JCS is used - not URDNA2015 - to avoid RDF-processing dependencies in adapter implementations. Two conformant implementations MUST produce byte-identical hashes for the same credential payload.
