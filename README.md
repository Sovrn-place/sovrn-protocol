# Sovrn Protocol

**The first open standard for portable identity credentials across sovereign economic zones.**

Sovrn Protocol defines how identity credentials are issued, verified, and ported across independent special economic zones, free ports, and smart cities. A KYC verification completed in one zone can be recognized by another zone without re-verification - while preserving user consent, privacy, and regulatory sovereignty.

There is no existing standard for portable identity credentials across independent economic zones. Sovrn Protocol is the first attempt to define one.

---

## What this is

Sovrn Protocol is a specification. It defines:

- **A credential schema** - A W3C VC 2.0 compliant data model for identity attestations issued by economic zones
- **An adapter interface** - A TypeScript contract (`KycAdapter`) that any identity provider can implement to issue credentials in the Sovrn format
- **A cross-zone verification protocol** - How credentials issued by Zone A are presented to and accepted by Zone B
- **An on-chain anchoring layer** - SHA-256 hashes of every credential, anchored on Base L2, creating a tamper-proof verification layer without exposing personal data

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

### 1. Credential Schema (W3C VC 2.0)

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

### 2. Credential Types

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

### 3. KycAdapter Interface

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

### 4. Reference Implementations

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

### 5. Federation Configuration

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

### 6. Cross-Zone Verification Flow

When a user presents credentials from Zone A to Zone B:

1. The user consents to sharing specific credentials
2. Zone B receives: credential type, level, issuing zone, issue date, hash, and attached reputation score
3. Zone B's admin resolves the credential against one of three paths:
   - **Accept** - skip re-verification, create a `CredentialPresentation` record with consent metadata
   - **Delta** - accept the base credential, request only additional checks not covered by the original
   - **Fresh** - reject and require a new verification (e.g. when zone policy requires it)

**What crosses zones:** Attestations, reputation scores, compliance status, credential hash
**What never crosses:** Original documents, raw PII, KYC session details, provider internals

The protocol is explicit about what is and is not portable. Raw personal data stays with the original issuer and its KYC provider.

### 7. On-Chain Anchoring

Every credential's SHA-256 hash is written to a contract on Base L2. This creates a tamper-evident verification layer without exposing any credential contents.

- **Track 1 (database-backed)** - Credential storage and lookup happen in a standard relational database for speed. Suitable for immediate verification in the issuing zone.
- **Track 2 (on-chain anchored)** - The credential hash is mirrored to a Base L2 contract. Any verifier can independently check that a presented credential matches the on-chain hash, making forgery or retroactive alteration detectable without a trusted database.

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
| W3C VC 2.0 credential schema | Defined |
| SHA-256 hash on every credential | Defined |
| KycAdapter interface | Defined |
| Cross-zone verification protocol | Defined |
| Federation config schema | Defined |
| Sumsub reference adapter | Implemented |
| Persona reference adapter | Implemented |
| Open-track reference adapters (Privado, Holonym, zkPassport, Anon Aadhaar) | Specified, not yet open-sourced |
| On-chain anchoring (Base L2 contract) | Specified (Track 2) |
| EUDI Wallet compatibility | Target Dec 2026 |
| OID4VCI / OID4VP profile | Target 2026 |

---

## Repository structure

```
/schemas       W3C VC 2.0 credential JSON-LD schemas
/adapters      KycAdapter interface + reference implementations
/protocol      Cross-zone verification protocol specification
/examples      Example credentials and adapter implementations
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
