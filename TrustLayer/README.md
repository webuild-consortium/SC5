# TrustLayer

A transport-independent trust layer for cross-organization business document exchange, anchored on the European Business Wallet (EUBW) ecosystem.

| | |
|---|---|
| **Date** | 2026-06-01 |
| **Version** | 0.2 Draft |
| **Status** | For discussion |
| **Author** | Hans Boone — Banqup |

---

## Why this exists

Cross-organization document exchange (invoices, orders, despatch advices, etc.) needs a portable layer of cryptographic trust that survives across whichever transport happens to carry the document. Today, every transport reinvents this layer in its own way — Peppol relies on Access Point PKI, email relies on S/MIME, EDI relies on bilateral agreements, QERDS relies on the registered-delivery cryptographic envelope. The result is a fragmented trust landscape where the originating legal entity, the authorisation of the sender's intermediary, and the live verification of supporting credentials (IBAN, VAT, approved-supplier) are either absent or expressed differently per transport.

The TrustLayer is one trust model, transport-agnostic, anchored on the EUBW Trusted List of Business Wallet Providers.

## The three layers

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 3: Transport bindings                                         │
│  How the PTE rides on a specific wire. One document per binding.     │
│  • Peppol (current focus; Oxalis-NG reference handler)               │
│  • Mail (sketch)                                                     │
│  • QERDS (sketch)                                                    │
│  • REST / OID4VP / others — future                                   │
└─────────────────────────────────────────────────────────────────────┘
                                ▲
                                │ implements
┌───────────────────────────────┴─────────────────────────────────────┐
│  Layer 2: APEIP — AP ↔ EUBW Integration Protocol                     │
│  How a message handler (Access Point, mail server, QERDS provider,   │
│  REST endpoint) integrates with its companion EUBW provider.         │
│  Outbound: /credentials/for-document, /seal, /presentation-request   │
│  Inbound:  /verify                                                   │
│  Transport-independent. Same protocol regardless of binding.         │
└─────────────────────────────────────────────────────────────────────┘
                                ▲
                                │ produces/consumes
┌───────────────────────────────┴─────────────────────────────────────┐
│  Layer 1: PTE — Portable Trust Envelope                              │
│  • STM (Sealed Trust Manifest): credentials + content binding,       │
│    covered by a qualified electronic seal                            │
│  • TVE (Trust Validation Envelope): post-verification trust score    │
│    + signed EUBW assertion                                           │
│  Transport-agnostic data structure. Carried in whichever container   │
│  the binding defines.                                                │
└─────────────────────────────────────────────────────────────────────┘
```

A transport binding picks a concrete carrier for the PTE (UBL extension for Peppol, MIME multipart for mail, an attachment in the QERDS envelope, an HTTP request body for REST), specifies what the message handler does, and uses APEIP to integrate with its EUBW. The PTE bytes are the same in all cases. The trust score returned by EUBW C3 is the same in all cases. Only the wire-level packaging differs.

## Directory layout

```
TrustLayer/
├── README.md                              ← you are here
├── architecture/
│   └── PTE-Architecture.md                ← Layer 1: the envelope itself
├── protocol/
│   └── APEIP-Specification.md             ← Layer 2: the AP ↔ EUBW protocol
├── bindings/                              ← Layer 3: transport bindings
│   ├── README.md                          ← the binding matrix
│   ├── peppol/
│   │   └── Oxalis-NG-Handler.md           ← reference implementation for Peppol
│   ├── mail/
│   │   └── Mail-Binding.md                ← sketch
│   └── qerds/
│       └── QERDS-Binding.md               ← sketch
└── sc5-binding/
    └── SC5-PTE-Technical-Binding.md       ← how SC5 attestations map onto the PTE
```

## How to read these documents

| If you want to understand... | Read... |
|---|---|
| What the trust artifact looks like, end-to-end | `architecture/PTE-Architecture.md` |
| How a message handler talks to its EUBW | `protocol/APEIP-Specification.md` |
| How to deploy this on Peppol | `bindings/peppol/Oxalis-NG-Handler.md` |
| Why transport-independence matters | `bindings/README.md` |
| How SC5 attestations encode into the PTE | `sc5-binding/SC5-PTE-Technical-Binding.md` |
| Where to start as an implementer | `bindings/README.md`, then your transport's binding doc |

## Status

All documents in this folder are **drafts for discussion**, not adopted specifications.

The Peppol binding is the most developed (concrete pilot proposed for SC5 Pilot 1: Banqup ↔ Semansys/Sphereon, scenarios 1 + 2). Other bindings are sketches that demonstrate the trust layer is genuinely transport-independent, not Peppol-coupled.

## Relationship to SC5

The TrustLayer is complementary to the WE BUILD SC5 specification suite (see [Scenario/Description.md](../Scenario/Description.md)). SC5 defines the business attestations (Approved Supplier, Authorized Service Provider) and the actor model; the TrustLayer specifies the technical encoding that closes SC5's deferred technical questions (where the attestation rides in the AS4 exchange; how attestations bind to invoice content; how C4 sees the verification result). See `sc5-binding/SC5-PTE-Technical-Binding.md` for the mapping.

## Relationship to standards bodies

| Body | Relevance |
|---|---|
| **OpenPeppol** | Governs Peppol BIS extensions; primary track for the Peppol binding (UBL extension schema, schematron rules). |
| **CEN/TC 434** | Governs EN 16931; can adopt the PTE as an EN 16931 Extension under TR 16931-5, broadening reach beyond Peppol (XRechnung, CIUS-PT, CII bindings). |
| **ETSI** | Governs XAdES (EN 319 132), JAdES, PAdES, eIDAS Trust Lists; the PTE seal is XAdES-bound today, PAdES for hybrid PDF formats. |
| **European Commission / EUBW Implementing Acts** | Will govern the EUBW Trusted List, EUID, and (we propose) APEIP. |

The Peppol binding can be piloted bilaterally today without any standardisation body's approval, because the artifacts live in extension points that the underlying standards (UBL, MIME, QERDS) explicitly reserve for foreign content. Standardisation is the path to scale, not a precondition for piloting.
