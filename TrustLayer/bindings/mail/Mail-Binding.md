# Mail Binding (SMTP/IMAP) — Sketch

**A sketch of the mail-transport binding for the Portable Trust Envelope (PTE).**

| | |
|---|---|
| **Date** | 2026-06-01 |
| **Version** | 0.1 Sketch |
| **Status** | Demonstrates transport-independence; not pilot-ready |
| **Author** | Hans Boone — Banqup |
| **Layer** | TrustLayer Layer 3 — transport binding (see [../../README.md](../../README.md)) |

> **This is a sketch, not a specification.** It exists to demonstrate that the Portable Trust Envelope is genuinely transport-independent — the same trust artifacts can ride on email as they ride on Peppol AS4. A production-ready mail binding would require substantial additional work on the points marked **OPEN**.

---

## 1. Why a mail binding matters

Most B2B exchanges in Europe still ride on email today. PDF invoices attached to email, ZUGFeRD/Factur-X hybrid PDFs attached to email, EDI files attached to email. Peppol coverage is concentrated in specific countries (Belgium, Norway, Italy, Netherlands) and specific document types (invoices). Outside those segments, **email is the actual transport for the bulk of B2B documents**.

A trust layer that only works on Peppol leaves this traffic uncovered. The mail binding demonstrates that the same EUBW-anchored trust verification can apply to email-borne documents — which dramatically broadens the addressable surface of the EUBW ecosystem.

Mail also matters for **B2C invoice delivery**, where consumers receive invoices by email from their utilities, telcos, and service providers. A PTE on a B2C email invoice gives the consumer (and consumer-protection authorities) cryptographic verification of the issuer's identity and content integrity — closing the gap that consumer-facing invoice fraud exploits today.

## 2. The binding choices

### 2.1 Carrier — `multipart/related`

The email is a `multipart/related` MIME message. The relevant parts:

```
From: invoicing@green-flowers.nl
To: ap@les-roses-d-or.fr
Subject: Invoice 2026-00042
MIME-Version: 1.0
Content-Type: multipart/related; type="application/xml"; boundary="boundary-A"
X-PTE-Version: 1.0
X-PTE-Binding: mail

--boundary-A
Content-Type: application/xml; name="invoice.xml"
Content-ID: <invoice.xml>
Content-Disposition: attachment; filename="invoice.xml"

<Invoice xmlns="...">
  ...
  <!-- Peppol BIS Billing 3.0 UBL invoice body -->
</Invoice>

--boundary-A
Content-Type: application/pte-stm+xml; name="trust-manifest.xml"
Content-ID: <trust-manifest.xml>
Content-Disposition: attachment; filename="trust-manifest.xml"

<SealedTrustManifest xmlns="urn:peppol:trust:pte:1.0">
  ...
</SealedTrustManifest>

--boundary-A
Content-Type: application/pkcs7-signature; name="seal.xades"
Content-ID: <seal.xades>
Content-Disposition: attachment; filename="seal.xades"

<ds:Signature ...>
  <!-- XAdES detached signature, covering invoice.xml + trust-manifest.xml -->
</ds:Signature>

--boundary-A
Content-Type: application/pte-tve+xml; name="trust-validation.xml"
Content-ID: <trust-validation.xml>
Content-Disposition: attachment; filename="trust-validation.xml"

<TrustValidationEnvelope xmlns="urn:peppol:trust:pte:1.0">
  <!-- Written by receiver's mail gateway after APEIP /verify -->
  ...
</TrustValidationEnvelope>

--boundary-A--
```

The structure parallels the Peppol binding's `<ext:UBLExtensions>` siblings: document body, STM, seal, TVE — but each lives in its own MIME part.

**For PDF documents** (Factur-X, ZUGFeRD, or pure PDF):

- The document part is `application/pdf`.
- The seal is PAdES (ETSI EN 319 142), embedded *inside the PDF* as a signature dictionary — not as a separate MIME part.
- The STM and TVE are still separate `application/pte-*+xml` MIME parts.
- The PAdES seal references the STM part via its content digest in a signed attribute.

This split (XAdES detached for XML, PAdES inside for PDF) is also what would apply for any other binding handling hybrid PDF formats.

### 2.2 Seal — XAdES detached, CAdES, or PAdES

| Document type | Seal format | Where the seal lives |
|---|---|---|
| XML (UBL, CII, plain XML) | XAdES detached (EN 319 132) | Separate MIME part (`application/pkcs7-signature` or `application/xades`) |
| PDF (Factur-X, ZUGFeRD, plain PDF) | PAdES (EN 319 142) | Inside the PDF (signature dictionary) |
| Binary (EDIFACT, X12, CSV, other) | CAdES (EN 319 122) | Separate MIME part |

The seal always covers the document body + STM, and excludes the TVE (which is added later). The mechanism for "exclude the TVE" is binding-specific:

- XAdES detached: the seal's `Reference` elements explicitly list the document part and the STM part; the TVE part is simply not referenced.
- PAdES: the PDF's `/ByteRange` defines the signed region as everything except the TVE attachment, with a placeholder reserved for the TVE's eventual addition.
- CAdES: same as XAdES detached but with `SignedData` content reference.

**OPEN: PAdES inside-PDF with separate STM MIME part.** This requires the PAdES seal to incorporate the STM's digest via a signed attribute (e.g., `signing-certificate-v2` plus a custom `pte-stm-digest` attribute). The exact attribute definition is not yet specified.

### 2.3 Handler — the mail gateway or mail-receiving server

The mail binding's "AP equivalent" is a mail-handling component on each side:

| Side | Component | APEIP role |
|---|---|---|
| **Sender** | Outbound mail gateway (the SMTP relay through which the sender's mail leaves the organisation) | Calls APEIP `/credentials/for-document`, `/seal`, `/presentation-request` |
| **Receiver** | Inbound mail server, mail-receiving gateway, or a milter-style filter on the receiving MTA | Calls APEIP `/verify`, writes TVE part |

The mail gateway is the natural integration point because it already touches every message and has the operational privileges needed to add/inspect MIME parts. Major mail platforms (Postfix, Exchange, Microsoft 365, Google Workspace) provide milter, transport-agent, or connector hooks for this kind of integration.

For organisations without their own mail gateway, a third-party "PTE-aware mail relay" service could be used — analogous to how some businesses use third-party Peppol APs rather than running their own.

### 2.4 Discovery — DNS TXT records

**OPEN: discovery mechanism for mail.** The Peppol binding uses the SMP. Mail has no equivalent universal directory. Three candidate mechanisms:

1. **DNS TXT records.** Analogous to SPF, DKIM, and DMARC: a TXT record at `_pte.<receiver-domain>` advertises PTE support and policy. Example:
   ```
   _pte.les-roses-d-or.fr.  IN  TXT  "v=PTE1; requires=approved-supplier:hard; policy=urn:peppol:trust:scoring:v1"
   ```
2. **EUBW directory.** The EUBW operates a directory of registered legal persons. The directory could expose PTE capability information by EUID + email-domain mapping.
3. **Out-of-band.** Bilateral arrangement between sender and receiver — works for known trading partners, fails for new ones.

For the sketch, **DNS TXT records** is the most natural mechanism because it mirrors mail's existing trust-discovery patterns (SPF/DKIM/DMARC) and uses the same DNS infrastructure mail servers already query for every message.

## 3. Flow

### 3.1 Send side

1. Sender's accounting system composes an invoice (XML or PDF).
2. Sender's mail gateway intercepts the outbound message.
3. Mail gateway extracts the document, looks up the receiver's PTE capability via DNS TXT.
4. If receiver supports PTE: gateway calls APEIP `/credentials/for-document`, builds STM, calls APEIP `/seal`, assembles the multipart MIME structure.
5. Gateway sends the augmented MIME message via SMTP.

If the receiver does **not** support PTE, the gateway sends a plain email with just the document attached — gracefully degrading to today's mail-invoice pattern.

### 3.2 Receive side

1. Receiver's mail server accepts the message via SMTP.
2. Inbound gateway / milter detects the `X-PTE-Version` header and the STM MIME part.
3. Gateway calls APEIP `/verify` with the STM, the document, and the sealed-document digest.
4. Gateway constructs the TVE MIME part from the response and appends it to the MIME structure (replacing any existing TVE part).
5. Gateway delivers the message to the receiver's mailbox (or to a back-office system, depending on configuration).

The receiver's ERP / back-office reads the TVE part to obtain the trust score, identically to how Peppol C4 reads the TVE UBL extension.

## 4. Open questions

| # | Question |
|---|---|
| O1 | **PAdES + separate-STM-MIME-part coordination.** How exactly does the PAdES seal incorporate the STM's digest as a signed attribute? Custom XAdES profile, or use existing OASIS DSS structures? |
| O2 | **DNS TXT record format.** Are SPF/DKIM/DMARC formats reusable, or does PTE need a fresh format? RFC pathway? |
| O3 | **Mail loop hardening.** Mail is more forgeable than Peppol AS4 (no transport PKI by default). Is DKIM on the sender side a prerequisite for PTE-mail, or does the PTE's own qualified seal subsume DKIM? |
| O4 | **Multipart preservation.** Some mail clients re-encode `multipart/related` messages. A standardised forwarding behaviour is required. |
| O5 | **EUBW visibility into mail addresses.** APEIP currently uses `receiverParticipantId` keyed by Peppol scheme. The mail binding would need a separate scheme (RFC 5322 address; possibly hashed). |
| O6 | **Spam filters and the PTE.** Many corporate spam filters strip or quarantine messages with multiple unusual MIME parts. Coordination with filter vendors needed. |
| O7 | **B2C deliverability.** Consumer mail providers (Gmail, Outlook.com, Apple Mail) need to support displaying / acting on PTE messages. Display contract design? |

## 5. Status

This document is a **sketch** to demonstrate feasibility. The architecture (multipart structure, seal placement, APEIP integration, DNS TXT discovery) is internally consistent and aligns with how mail's existing trust mechanisms work (SPF/DKIM/DMARC patterns). Completing the binding to a pilot-ready state requires resolving the open questions in §4, agreeing the discovery format, and producing reference implementations for at least Postfix and Microsoft 365.

A reasonable next step would be a Banqup-led pilot of the mail binding between two domains (one acting as sender, one as receiver) using DNS TXT for discovery and a Postfix milter for the gateway integration. This could be done bilaterally without standards-body approval, similar to the Peppol pilot.
