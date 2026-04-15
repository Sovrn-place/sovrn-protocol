# Sovrn Protocol

**Status:** Draft · **Version:** 0.1 · **License:** [Apache 2.0](./LICENSE)

Economic zones, free ports, and smart cities are the fastest-growing governance structures in the world, yet they have no shared identity infrastructure. A resident verified in one zone starts from scratch in another. A company incorporated in Estonia cannot present those credentials in Colombo. Every zone builds its own siloed onboarding, its own KYC, its own credential format.

**Sovrn Protocol is the identity and credential layer that connects them.** A KYC verification completed in one zone can be recognized by another without re-verification, while preserving user consent, privacy, and regulatory sovereignty.

> **Full specification:** [SPEC.md](./SPEC.md)

---

## What this defines

- **`did:sovrn:` method and the `.si` namespace** - a decentralized identifier purpose-built for multi-zone identity
- **Credential schemas (W3C VC 2.0)** - standardized formats for KYC verification, residency, incorporation, tax compliance, and payment history, canonicalized with [JCS (RFC 8785)](https://datatracker.ietf.org/doc/html/rfc8785) and hashed with SHA-256
- **`KycAdapter` interface** - a provider-agnostic contract any verification provider can implement
- **Cross-zone presentation format** - how credentials move between independent zones with explicit user consent
- **Reputation schema** - five-dimension portable reputation with standardized tiers
- **On-chain credential storage** (planned) - SHA-256 credential hashes stored on-chain for tamper-evident verification

## Compatible providers

Any provider implementing the [`KycAdapter`](./SPEC.md#3-kycadapter-interface) interface. Planned reference adapters for:

Privado ID · Holonym / Human ID · zkPassport · Anon Aadhaar · Sumsub · Persona

Both open-source and commercial providers emit identical W3C VC 2.0 credentials. A credential issued via any compliant adapter is indistinguishable from any other at the protocol layer.

## Repository structure

```
/SPEC.md       Full normative specification
/schemas       W3C VC 2.0 credential JSON-LD schemas             (forthcoming)
/adapters      KycAdapter interface and reference implementations  (forthcoming)
/examples      Example credentials, presentations, adapters       (forthcoming)
/docs          Additional specification documents                 (forthcoming)
```

Directories marked *forthcoming* will be populated as components are extracted and published.

## Namespace disambiguation

The `.si` namespace in this protocol is an **internal Sovrn identity namespace**, not the Slovenian ccTLD (operated by ARNES). A `.si` name such as `alex.si` is not a DNS hostname and is not resolved via DNS.

## Governance

Maintained by [Sovrn](https://sovrn.place). Schema changes go through an RFC process: open an issue with the proposed change, rationale, and backwards-compatibility analysis. Breaking changes require a major version bump.

## Security

Security issues: email **contactus@sovrnplace.com**. Do not open public issues for vulnerabilities.

## Platform

**Demo:** [Link](https://sovrn-demo.vercel.app) - access password, message @AashrayaRau on Telegram.

Sovrn Protocol is implemented by the [Sovrn platform](https://sovrn.place), which additionally provides:

- Zone administration and application management
- AI-assisted compliance review
- Programmable escrow and multi-currency settlement
- Reputation scoring across five dimensions
- Federation onboarding for new zones
- Immutable audit trail with on-chain credential storage

For platform access or zone partnership inquiries: **contactus@sovrnplace.com**

## Links

[sovrn.place](https://sovrn.place) · [AI Advisor](https://ai.sovrn.place) · [linktr.ee/joinsovrn](https://linktr.ee/joinsovrn) · [Newsletter](https://sovrn-newsletter.beehiiv.com/) · [Early Access](https://tally.so/r/3j64jY)
