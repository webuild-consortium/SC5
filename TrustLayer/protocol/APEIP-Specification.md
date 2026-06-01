# APEIP — AP ↔ EUBW Integration Protocol

**Specification of the protocol between a message-handling intermediary and its companion European Business Wallet provider.**

| | |
|---|---|
| **Date** | 2026-06-01 |
| **Version** | 0.1 Draft |
| **Status** | For discussion |
| **Author** | Hans Boone — Banqup |
| **Layer** | TrustLayer Layer 2 (see [../README.md](../README.md)) |
| **Companion documents** | [../architecture/PTE-Architecture.md](../architecture/PTE-Architecture.md) (Layer 1), [../bindings/](../bindings/) (Layer 3) |

---

## Index

1. [Purpose and scope](#1-purpose-and-scope)
2. [Position in the architecture](#2-position-in-the-architecture)
3. [Transport-independence](#3-transport-independence)
4. [Endpoint inventory](#4-endpoint-inventory)
5. [Outbound: `/credentials/for-document`](#5-outbound-credentialsfor-document)
6. [Outbound: `/seal`](#6-outbound-seal)
7. [Outbound: `/presentation-request`](#7-outbound-presentation-request)
8. [Inbound: `/verify`](#8-inbound-verify)
9. [Authentication](#9-authentication)
10. [Error model](#10-error-model)
11. [Versioning and evolution](#11-versioning-and-evolution)

---

## 1. Purpose and scope

APEIP specifies how a **message-handling intermediary** (a Peppol Access Point, a mail server's outbound/inbound pipeline, a QERDS provider, a REST endpoint operator) integrates with its **companion EUBW provider** to produce and consume Portable Trust Envelope (PTE) artifacts. It is the protocol layer between the on-the-wire trust envelope (Layer 1) and the transport-binding implementation (Layer 3).

**What APEIP is:**

- A bounded, narrow integration contract between two cooperating service providers — a message handler and a wallet — both of which are operated by recognised legal entities on the EUBW Trusted List of Business Wallet Providers.
- Transport-independent: the same protocol regardless of whether the document travels via Peppol AS4, email, QERDS, REST, or any other carrier.
- Service-to-service: machine-to-machine REST/JSON over mTLS, no human-in-the-loop, no browser flows.

**What APEIP is not:**

- It is not a wallet protocol. The wallet-to-wallet trust relationship (EUBW C2 ↔ EUBW C3 over the EUBW Trusted List) is governed by EUBW specifications, not APEIP.
- It is not a transport protocol. The carriage of documents over Peppol AS4, SMTP, QERDS, etc. is governed by the respective transport specifications, not APEIP.
- It is not exposed to end users (C1, C4). The PTE in the document is what reaches them; APEIP is invisible to them.

**Who runs APEIP endpoints:**

- A wallet provider (the EUBW operator, e.g. Sphereon, Banqup's EUBW arm, Credenco) exposes APEIP endpoints to its authorised message-handler clients.
- A message handler (the Peppol AP operator, the mail server operator, the QERDS provider, the REST sender/receiver) is the APEIP client.
- Both sides are legal entities registered on the EUBW Trusted List of Business Wallet Providers (or, for the handler side, on a Peppol equivalent, or a bilateral agreement — see §9).

## 2. Position in the architecture

```
                Sender                                       Receiver
   ┌──────────────────────────┐                ┌──────────────────────────┐
   │ C1 legal entity (ERP)    │                │ C4 legal entity (ERP)    │
   └────────────┬─────────────┘                └──────────────▲───────────┘
                │                                              │
                │ document                                     │ document + PTE
                ▼                                              │
   ┌──────────────────────────┐    transport    ┌──────────────┴───────────┐
   │ Sender message handler   │═══════════════▶│ Receiver message handler │
   │ (Peppol C2 AP /          │   (AS4, mail,   │ (Peppol C3 AP /          │
   │  mail server /           │    QERDS, ...)  │  mail server /           │
   │  QERDS sender /          │                 │  QERDS receiver /        │
   │  REST sender)            │                 │  REST receiver)          │
   └────────────┬─────────────┘                 └──────────────┬───────────┘
                │                                              │
                │ APEIP outbound                               │ APEIP inbound
                │ /credentials/for-document                    │ /verify
                │ /seal                                        │
                │ /presentation-request                        │
                ▼                                              ▼
   ┌──────────────────────────┐                  ┌──────────────────────────┐
   │ Sender EUBW (EUBW C2)    │  ◀───────────▶  │ Receiver EUBW (EUBW C3)  │
   │                          │  EUBW-to-EUBW    │                          │
   │  • holds C1 credentials  │  via EUBW        │  • verifies STM creds    │
   │  • issues IntermdtMdt    │  Trusted List    │  • computes trust score  │
   │  • applies QSeal         │  (out of APEIP   │  • signs assertion       │
   │                          │   scope)         │                          │
   └──────────────────────────┘                  └──────────────────────────┘
```

APEIP is the four endpoints labelled in the diagram. Everything to the left of the sender handler and to the right of the receiver handler is APEIP-driven. Everything outside is transport (top) or EUBW-internal (bottom).

## 3. Transport-independence

APEIP makes no assumption about how the document travels between the two handlers. The protocol is invoked at four points:

| When | Endpoint | Caller | Purpose |
|---|---|---|---|
| Before send, sender side | `/credentials/for-document` | Sender handler → sender EUBW | Obtain the credentials needed to build the STM for a specific (sender, receiver, document type) tuple. |
| Before send, sender side | `/seal` | Sender handler → sender EUBW | Apply a qualified electronic seal over the canonicalised document + STM. |
| Before send, sender side (optional) | `/presentation-request` | Sender handler → sender EUBW | Request a buyer-specific credential (e.g. Approved Supplier) to push into the STM, if known at send time. |
| After receive, receiver side | `/verify` | Receiver handler → receiver EUBW | Verify the received PTE; obtain a trust score and a signed EUBW assertion. |

The handler is responsible for binding-specific carriage of the PTE artifacts:

- **Peppol binding**: STM, seal, and TVE go into sibling `<ext:UBLExtension>` elements; the sealed-document digest is computed over the canonicalised UBL.
- **Mail binding**: STM, seal, and TVE go into MIME parts of a multipart/related body; the sealed-document digest is computed over the document MIME part.
- **QERDS binding**: STM, seal, and TVE are attachments referenced by the QERDS evidence; the sealed-document digest is computed over the primary attachment.
- **REST binding**: STM, seal, and TVE are JSON properties (or base64 strings) of the request body; the sealed-document digest is computed over the document content property.

In every binding, the same APEIP calls produce the same PTE bytes, and the same APEIP `/verify` returns the same trust score. The transport-binding documents under [../bindings/](../bindings/) specify the carrier mechanics; APEIP itself does not change.

## 4. Endpoint inventory

| Endpoint | Direction | Idempotent | Streaming | Notes |
|---|---|---|---|---|
| `POST /apeip/v1/credentials/for-document` | Outbound (handler → EUBW) | Yes (same inputs → same credential set, modulo revocation events) | No | Returns credential references + digests; clients cache. |
| `POST /apeip/v1/seal` | Outbound (handler → EUBW) | No (each call produces a fresh seal with a fresh timestamp) | No | The QSeal endpoint. Key material never leaves the EUBW's QSCD. |
| `POST /apeip/v1/presentation-request` | Outbound (handler → EUBW) | Yes per (invoice, credential type, target) | No | Asks the wallet to present a specific credential out-of-band. |
| `POST /apeip/v1/verify` | Inbound (handler → EUBW) | Yes (same STM + sealed digest → same verification, modulo live revocation) | No | The verification endpoint. Returns trust score + signed assertion. |

All endpoints:

- Use HTTPS with mTLS authentication (see §9).
- Accept `Content-Type: application/json`.
- Return `Content-Type: application/json` on success and `application/problem+json` (RFC 7807) on errors.
- Are prefixed with `/apeip/v1/` to allow versioning (see §11).
- Are operated by the EUBW provider; the handler is always the client.

## 5. Outbound: `/credentials/for-document`

The sender handler calls this endpoint after C1's ERP has submitted a document and before the handler builds the STM. The EUBW returns the credential set the handler needs to construct a complete, EUBW-anchored STM.

### Request

```http
POST /apeip/v1/credentials/for-document HTTP/1.1
Host: eubw-c2.example.eu
Content-Type: application/json
Authorization: <see §9>
```

```json
{
  "senderEUID": "BEBE0123.456.789",
  "receiverParticipantId": {
    "scheme": "9925",
    "value": "BE0987654321"
  },
  "documentType": "urn:oasis:names:specification:ubl:schema:xsd:Invoice-2:Invoice",
  "documentDigest": {
    "algorithm": "SHA-256",
    "value": "f7c2a1...",
    "scope": "document body before STM insertion"
  },
  "extractedFields": {
    "ibans": ["BE68539007547034"],
    "vatNumbers": [{"jurisdiction": "BE", "number": "BE0123456789"}],
    "totalAmount": {"currency": "EUR", "value": 12500.00}
  },
  "receiverPolicyHint": {
    "requiresApprovedSupplier": "unknown",
    "policy": "urn:peppol:trust:scoring:v1"
  }
}
```

Field notes:

- `senderEUID`: the legal entity identifier of C1, used to look up C1's credentials in the EUBW.
- `receiverParticipantId`: binding-specific receiver identifier (Peppol participant ID in the Peppol binding; email address in the mail binding; etc.). The EUBW uses this to determine whether buyer-specific credentials (e.g. Approved Supplier) should be included.
- `documentType`: a URN identifying the document semantic. The Peppol binding uses the UBL `CustomizationID`-equivalent; the mail binding may use a MIME-type-based URN; etc.
- `documentDigest`: digest of the document body *before* STM insertion. The EUBW does not need the full document, only its digest, for content-binding decisions.
- `extractedFields`: the handler extracts the few fields it needs to ask the EUBW to assemble matching credentials. The EUBW does not parse the document itself.
- `receiverPolicyHint.requiresApprovedSupplier`: `"yes" | "no" | "unknown"`, used by the EUBW to decide whether to include the Approved Supplier credential proactively (push-at-send path).

### Response

```json
{
  "credentialSet": {
    "legalPersonAttestation": {
      "credentialURI": "https://eubw-c2.example.eu/vc/lpa/42",
      "digest": {"algorithm": "SHA-256", "value": "a3f2..."},
      "validUntil": "2027-01-01T00:00:00Z"
    },
    "intermediationMandate": {
      "credentialURI": "https://eubw-c2.example.eu/vc/mandate/9f2c",
      "digest": {"algorithm": "SHA-256", "value": "b41d..."},
      "mandatee": "BEBE0987.654.321",
      "scope": "e-invoicing:seal+submit",
      "validUntil": "2026-12-31T23:59:59Z"
    },
    "supportingCredentials": [
      {
        "type": "LPIDCredential",
        "credentialURI": "https://eubw-c2.example.eu/vc/lpid",
        "digest": {"algorithm": "SHA-256", "value": "c128..."}
      },
      {
        "type": "VATCredential",
        "jurisdiction": "BE",
        "vatNumber": "BE0123456789",
        "credentialURI": "https://eubw-c2.example.eu/vc/vat/BE",
        "digest": {"algorithm": "SHA-256", "value": "d937..."}
      },
      {
        "type": "IBANCredential",
        "iban": "BE68539007547034",
        "credentialURI": "https://eubw-c2.example.eu/vc/iban/01",
        "digest": {"algorithm": "SHA-256", "value": "e4a1..."}
      },
      {
        "type": "ApprovedSupplierCredential",
        "issuerEUID": "FRFR9876.543.210",
        "credentialURI": "https://eubw-c2.example.eu/vc/asc/buyer-fr-1",
        "digest": {"algorithm": "SHA-256", "value": "f0b3..."}
      }
    ]
  },
  "trustAnchorPointers": {
    "eubwTrustList": "https://ec.europa.eu/eubw/tl",
    "eIDASTrustList": "https://ec.europa.eu/tools/lotl/eu-lotl.xml",
    "eubwDirectory": "https://ec.europa.eu/eubw/directory"
  },
  "senderEUBWEndpoint": "https://eubw-c2.example.eu",
  "expiresAt": "2026-06-01T10:30:00Z"
}
```

The handler embeds these references (URIs + digests) into the STM as `Resolution mode="byReference"` elements per [PTE-Architecture §4.7](../architecture/PTE-Architecture.md#47-resolvable-credentials---the-senders-eubw-as-the-authoritative-store). The receiver EUBW resolves them at verification time, producing live status checks.

### Errors

| HTTP | Condition | `type` URI |
|---|---|---|
| 404 | Sender EUID not registered with this EUBW | `urn:apeip:error:unknown-sender` |
| 403 | Caller (the handler) is not authorised to act for this sender | `urn:apeip:error:handler-not-authorised` |
| 409 | Sender's `IntermediationMandate` to this handler is missing, revoked, or expired | `urn:apeip:error:mandate-invalid` |
| 422 | `extractedFields` reference IBAN/VAT not known to the EUBW for this sender | `urn:apeip:error:field-mismatch` |

## 6. Outbound: `/seal`

After building the STM, the sender handler calls `/seal` to obtain a qualified electronic seal over the canonicalised document + STM. The EUBW (or the QTSP behind it) applies the seal using C1's QSeal certificate; the key material never leaves the cryptographic boundary.

### Request

```json
{
  "senderEUID": "BEBE0123.456.789",
  "sealCertificateRef": "urn:eubw:cert:qtsp:7b1a...",
  "sealFormat": "XAdES-B-LT",
  "canonicalisation": "http://www.w3.org/2006/12/xml-c14n11",
  "signatureAlgorithm": "http://www.w3.org/2001/04/xmldsig-more#ecdsa-sha256",
  "digestToSign": {
    "algorithm": "SHA-256",
    "value": "..."
  },
  "xadesSignedProperties": {
    "signingTime": "2026-06-01T10:23:45Z",
    "commitmentTypeId": "http://uri.etsi.org/01903/v1.2.2#ProofOfOrigin",
    "signaturePolicy": "implied"
  },
  "augmentation": "B-LT"
}
```

The handler is responsible for canonicalising the document, computing the digest, and assembling the unsigned `<ds:Signature>` skeleton (per [PTE-Architecture §4.6.4](../architecture/PTE-Architecture.md#464-generation-procedure--step-by-step)). The EUBW only signs the digest, augments to the requested XAdES level, and returns the signature value and timestamp/validation material.

For PDF-based bindings (e.g. hybrid CII formats like ZUGFeRD, Factur-X), `sealFormat` is `PAdES-B-LT` and the canonicalisation/signature mechanics follow ETSI EN 319 142 instead of EN 319 132.

### Response

```json
{
  "signatureValue": "MEYCIQD...",
  "augmentationMaterial": {
    "timestamp": "<base64 RFC 3161 TST>",
    "certificateValues": ["<base64 DER>", "..."],
    "revocationValues": {
      "ocspResponses": ["<base64>", "..."],
      "crls": []
    }
  },
  "sealedAt": "2026-06-01T10:23:46Z",
  "sealCertificate": "<base64 DER>",
  "level": "XAdES-B-LT"
}
```

The handler assembles the final `<ds:Signature>` element (binding-specific) using the returned values.

### Errors

| HTTP | Condition | `type` URI |
|---|---|---|
| 403 | Caller not authorised to seal as this sender | `urn:apeip:error:seal-not-authorised` |
| 409 | Sender's seal certificate has expired or been revoked | `urn:apeip:error:seal-cert-invalid` |
| 503 | QSCD/QTSP temporarily unavailable | `urn:apeip:error:qscd-unavailable` |

## 7. Outbound: `/presentation-request`

Optional. Used by the sender handler at send time when it knows the receiver requires a buyer-specific credential (e.g. Approved Supplier) and wants to **push** it in the STM rather than relying on the receiver's pull-at-receive path. Symmetric semantics with the receiver-side pull request (see §8); both flow through the EUBW's standard presentation-request protocol.

### Request

```json
{
  "subjectEUID": "BEBE0123.456.789",
  "requestedCredential": {
    "type": "ApprovedSupplierCredential",
    "issuerEUID": "FRFR9876.543.210"
  },
  "purpose": "include-in-stm",
  "scope": {
    "documentDigest": {
      "algorithm": "SHA-256",
      "value": "..."
    }
  }
}
```

### Response

Either the credential (in the same shape as a `credentialSet` element from §5) or:

```json
{
  "status": "not_available",
  "reason": "no ApprovedSupplierCredential from issuer FRFR9876.543.210 for subject BEBE0123.456.789"
}
```

## 8. Inbound: `/verify`

The receiver handler calls `/verify` after receiving a document. The receiver EUBW performs cryptographic verification of the seal, resolves and verifies STM credentials against the sender EUBW (over the EUBW Trusted List), applies receiver policy, computes the trust score, and signs an EUBW assertion. The handler embeds the result into the TVE and delivers the augmented document to C4.

### Request

```json
{
  "documentBinding": "peppol-bis-billing-3.0",
  "sealedDocumentDigest": {
    "algorithm": "SHA-256",
    "value": "f7c2a1..."
  },
  "stm": {
    "format": "xml",
    "content": "<base64 of the canonicalised STM extension content>"
  },
  "document": {
    "format": "xml",
    "content": "<base64 of the canonicalised document body, for content-binding checks>"
  },
  "verifierContext": {
    "handlerIdentity": "urn:peppol:c3:provider:nl",
    "receiverParticipantId": {
      "scheme": "9988",
      "value": "NL987654321"
    },
    "receiverPolicy": "urn:peppol:trust:scoring:v1"
  },
  "options": {
    "freshness": "now"
  }
}
```

Field notes:

- `documentBinding`: identifies the transport binding so the EUBW knows how to interpret the carrier. Values: `peppol-bis-billing-3.0`, `mail-mime-multipart`, `qerds-attachment`, `rest-json`, etc.
- `stm.content`: base64 of the canonicalised STM carrier element (binding-specific canonicalisation).
- `document.content`: base64 of the canonicalised document body (without STM, without seal, without TVE — what's underneath all the trust artifacts). The EUBW needs this for content-binding cross-checks (IBAN credential ↔ payment account in document; VAT credential ↔ tax scheme in document).
- `freshness`: see [PTE-Architecture §4.7.2](../architecture/PTE-Architecture.md#472-resolution-and-freshness--eubw-c3-calls-eubw-c2) for the cache-vs-now tradeoff.

### Response

```json
{
  "validationTime": "2026-06-01T10:24:07Z",
  "verifier": "urn:eubw:euid:NLNL5555.666.777",
  "anchoredTo": {
    "sealedDocumentDigest": "f7c2a1...",
    "sealCertificate": "urn:eubw:cert:qtsp:7b1a..."
  },
  "trustScore": {
    "value": 91,
    "scale": "0-100",
    "policy": "urn:peppol:trust:scoring:v1"
  },
  "dimensionalScores": [
    {"dimension": "seal-validity",            "score": 100, "status": "verified"},
    {"dimension": "legal-person-identity",    "score": 100, "status": "verified"},
    {"dimension": "intermediation-authority", "score": 100, "status": "verified"},
    {"dimension": "vat-consistency",          "score": 100, "status": "verified"},
    {"dimension": "iban-consistency",         "score":  90, "status": "verified"},
    {"dimension": "lpid-freshness",           "score":  85, "status": "verified"},
    {"dimension": "format-compliance",        "score": 100, "status": "verified"},
    {"dimension": "supplier-approval",        "score": 100, "status": "verified",
     "note": "Not present in STM; retrieved via presentation request to sender EUBW"}
  ],
  "findings": [
    {"code": "SUPPLIER-APPROVAL-PULLED", "severity": "info",
     "message": "Receiver policy required ApprovedSupplierCredential; not delivered in STM; presentation request to sender EUBW succeeded; credential verified live."}
  ],
  "decision": {
    "action": "accept-and-book",
    "threshold": 90
  },
  "assertion": {
    "format": "JWS",
    "value": "eyJhbGciOiJFUzI1NiIsImtpZCI6IkVVQlctQzMtTkwtMDEifQ..."
  }
}
```

The dimensional scoring model is defined in [PTE-Architecture §4.8](../architecture/PTE-Architecture.md#48-trust-score--weighted-dimensional-non-boolean). The signed `assertion` is the authoritative trust artifact; the handler embeds it into the TVE.

### What the receiver EUBW does (summary)

1. Verify the XAdES (or PAdES) seal against the eIDAS Trust List.
2. Resolve each STM credential by calling sender EUBW (URI in `STM/Sender/EUBWEndpoint`), authenticated via the EUBW Trusted List.
3. For each resolved credential, verify cryptographic proof, validity period, and live status.
4. Cross-check credential content against the document body (IBAN credential ↔ document IBAN, VAT credential ↔ document VAT, etc.).
5. Apply receiver policy. If the policy requires an `ApprovedSupplierCredential` and the STM does not contain one, issue a presentation request to sender EUBW.
6. Compute trust score per receiver policy. Hard-requirement dimensions scoring 0 force the aggregate to 0.
7. Sign the result as a JWS (or JAdES) using the EUBW's seal key, registered on the EUBW Trusted List.
8. Return.

Full step-by-step verification model is in [PTE-Architecture §4.9](../architecture/PTE-Architecture.md#49-the-verification-call--apeip-verify).

### Errors

| HTTP | Condition | `type` URI |
|---|---|---|
| 400 | Malformed STM or document encoding | `urn:apeip:error:malformed-pte` |
| 422 | Seal cryptographically invalid (the response body still includes `dimensionalScores` with `seal-validity=0`) | `urn:apeip:error:seal-invalid` |
| 502 | Sender EUBW unreachable; verification could not complete | `urn:apeip:error:sender-eubw-unreachable` |
| 503 | Receiver EUBW temporarily unable to verify | `urn:apeip:error:verifier-unavailable` |

Note: a 422 with `seal-invalid` is **not** an integration failure — it's a verification result the handler should propagate (TVE shows the failure; receiver policy declines). 502/503 are integration failures.

## 9. Authentication

Three modes are defined, in increasing strength:

| Mode | Mechanism | Use when |
|---|---|---|
| **same-provider** | Internal API key or mTLS within the same legal entity | The handler and the EUBW are operated by the same provider (e.g. Banqup operating both Peppol AP and EUBW). |
| **bilateral** | mTLS with a pre-issued `IntermediationCredential` from the EUBW provider to the handler operator | Different providers with a direct contractual relationship. |
| **TL-anchored** | mTLS where both certificates chain to entries on the EUBW Trusted List of Business Wallet Providers (handler side may chain to a separate Peppol-equivalent TL where applicable) | Multi-provider ecosystems with no bilateral pre-arrangement. |

The **TL-anchored** mode is the long-term target — it makes APEIP usable between any qualified handler and any qualified EUBW without pairwise contracts. It is also what makes the underlying EUBW C3 ↔ EUBW C2 call work (same mechanism, one hop further into the wallet ecosystem).

All APEIP endpoints require mTLS in all three modes. API keys (same-provider mode) are an additional factor on top of mTLS, not a substitute.

## 10. Error model

APEIP uses RFC 7807 Problem Details for HTTP APIs. Error responses have:

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json
```

```json
{
  "type": "urn:apeip:error:mandate-invalid",
  "title": "IntermediationMandate is revoked",
  "status": 422,
  "detail": "The IntermediationMandate urn:eubw:mandate:9f2c... was revoked on 2026-05-30T08:14:00Z.",
  "instance": "urn:apeip:request:b3c4...",
  "apeip": {
    "endpoint": "/apeip/v1/credentials/for-document",
    "version": "1.0",
    "diagnosticId": "DIAG-2026-06-01-12345"
  }
}
```

The `type` URIs are stable identifiers; new types may be added in minor versions. Clients should not pattern-match on `title` or `detail`.

## 11. Versioning and evolution

The protocol path includes a major version (`/apeip/v1/...`). Major versions are non-backward-compatible breaks; minor versions add fields and never break existing clients.

| Change category | Versioning impact |
|---|---|
| New optional request field | Minor (no change to URL) |
| New optional response field | Minor (no change to URL) |
| New dimensional score in `/verify` response | Minor |
| New error `type` URI | Minor |
| New endpoint | Minor |
| Removed field, changed field semantics, changed required-ness | Major (`/v2/...`) |
| New mandatory authentication mechanism | Major |

EUBW operators MUST support the previous major version for at least 12 months after a new major is released.

---

**End of document.**
