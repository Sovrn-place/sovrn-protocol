# sovrn-protocol

Open credential schemas, verification adapters, and cross-zone portability specifications for sovereign digital identity infrastructure.

## What this is

Sovrn Protocol defines how identity credentials are issued, verified, and ported across independent economic zones. It's designed so that a KYC verification done in one zone can be recognized by another zone without re-verifying — while preserving user consent and privacy.

This repo contains the **open** components of Sovrn's identity stack. The platform that implements these specifications is proprietary.

## Specifications

### Credential Schema (W3C VC 2.0)

All credentials conform to the [W3C Verifiable Credentials Data Model 2.0](https://www.w3.org/TR/vc-data-model-2.0/) specification.

```json
{
  "@context": ["https://www.w3.org/ns/credentials/v2"],
  "type": ["VerifiableCredential", "SovrnIdentityCredential"],
  "issuer": "did:sovrn:{zone-id}",
  "credentialSubject": {
    "id": "did:sovrn:{user-uuid}",
    "type": "KYC_VERIFIED",
    "level": 2,
    "issuingZone": "itana-ng",
    "hash": "sha256:a3f8c1..."
  }
}
```

Every credential includes a SHA-256 hash computed from the credential payload. This hash serves as the bridge between the database layer (fast, Track 1) and future on-chain anchoring (trust, Track 2).

### Credential Types

| Type | Description | Tier |
|------|-------------|------|
| `KYC_VERIFIED` | Identity verified via government ID + liveness | 1 |
| `KYC_ENHANCED` | + Address proof + PEP screening | 2 |
| `KYC_FULL` | + Source of funds + enhanced due diligence | 3 |
| `RESIDENCY_ACTIVE` | Active residency in a zone | — |
| `BUSINESS_REGISTERED` | Company incorporated in a zone | — |
| `TAX_COMPLIANT` | Tax obligations met for a filing period | — |
| `PAYMENT_CONFIRMED` | Payment settled through the platform | — |

### KycAdapter Interface

The `KycAdapter` is the integration contract that any KYC provider — open or licensed — must implement to work with Sovrn.

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

### Two-Track Hybrid Model

Sovrn uses a hybrid verification model:

**Track A — Open adapters** (Tier 1 only)
- Privado ID (zero-knowledge proofs)
- Holonym / Human ID (humanity verification)
- zkPassport (passport ZK proofs)
- Anon Aadhaar (Indian identity, ZK)

**Track B — Licensed adapters** (Tier 1-3)
- Sumsub (primary)
- Persona (secondary)

Both tracks produce identical W3C VC 2.0 credentials with the same SHA-256 hash. A credential issued via Privado ID is indistinguishable from one issued via Sumsub when presented to a zone admin.

### Cross-Zone Verification Protocol

When a user applies to Zone B with credentials from Zone A:

1. User consents to sharing specific credentials
2. Zone B receives: credential type, level, issuing zone, date, hash, reputation score
3. Zone B admin can: accept (skip re-verification), request delta (additional checks only), or request fresh verification
4. Acceptance creates a `CredentialPresentation` record with consent metadata

**What crosses zones:** Attestations, reputation scores, compliance status
**What never crosses:** Original documents, raw PII, KYC session details

### Federation Configuration

Each zone configures its verification requirements:

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

## Standards Alignment

- **W3C VC 2.0** — Credential data model
- **OID4VCI / OID4VP** — Credential issuance and presentation
- **ISO 18013-5 (mdoc)** — Mobile document format
- **EUDI Wallet** — Target compatibility by December 2026
- **FATF Recommendations** — AML/CFT compliance framework

## Build Targets

| Target | Status |
|--------|--------|
| W3C VC 2.0 credentials | Implemented |
| SHA-256 hash on every credential | Implemented |
| KycAdapter interface | Defined |
| Sumsub adapter | Implemented |
| Persona adapter | Implemented |
| Cross-zone verification | Implemented |
| Open adapter stubs | Defined, not built |
| On-chain hash anchoring (Base L2) | Track 2 |
| EUDI Wallet compatibility | Target Dec 2026 |

## License

The specifications, schemas, and interface definitions in this repository are licensed under [Apache 2.0](LICENSE).

## Contributing

This protocol is maintained by [Sovrn](https://sovrn.place). If you're building identity infrastructure for economic zones and want to implement these specifications, open an issue or reach out.
