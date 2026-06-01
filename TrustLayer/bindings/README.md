# Transport bindings

A **transport binding** specifies how the Portable Trust Envelope (PTE) is carried on a specific wire — which carrier element holds the STM, the qualified seal, and the TVE; which software role acts as the message handler; how a sender discovers whether a receiver supports the PTE.

The PTE itself, and the APEIP protocol between handler and EUBW, are **transport-independent**. The binding only specifies the carriage mechanics.

## Why multiple bindings matter

A trust layer that only works on Peppol is a Peppol enhancement, not a trust layer. The PTE is designed to ride on any transport that can carry a structured document with extension points or attachments. Three reasons this matters in practice:

| Reason | Concrete example |
|---|---|
| **Reach beyond Peppol** | B2C invoice delivery, B2B exchanges that still use email (which is most of them), bilateral REST integrations, EDI residue. The trust layer must reach these to be useful at scale. |
| **Regulatory diversification** | QERDS (eIDAS Art. 44) is the appropriate transport for formal notices, contractual deliveries, and regulator submissions. The trust layer should anchor those equally well. |
| **Standardisation independence** | Pinning the trust layer to one transport pins its governance to that transport's standards body (OpenPeppol in the Peppol case). Multiple bindings let the trust layer be standardised on its own track (CEN/TC 434, EUBW Implementing Acts). |

The Peppol binding is the most developed because it is closest to production deployment (Belgium B2B 2026, SC5 pilots, ViDA reporting). Other bindings are sketched here to demonstrate that the trust layer is genuinely transport-independent. They are not full specifications yet.

## Binding matrix

| Binding | Document | Carrier | Seal format | Handler role | Status |
|---|---|---|---|---|---|
| **Peppol BIS UBL over AS4** | [peppol/Oxalis-NG-Handler.md](peppol/Oxalis-NG-Handler.md) | UBL `<ext:UBLExtensions>` | XAdES via UBL DSig enveloped profile | Peppol Access Point (Oxalis-NG reference implementation) | Pilot-ready (SC5 Pilot 1: Banqup ↔ Semansys/Sphereon) |
| **Mail (SMTP/IMAP)** | [mail/Mail-Binding.md](mail/Mail-Binding.md) | `multipart/related` MIME body; PTE parts adjacent to the document part | XAdES detached for XML payloads; CAdES for non-XML; PAdES for PDF payloads | Mail server / mail gateway with PTE-aware module | Sketch |
| **QERDS (eIDAS Art. 44)** | [qerds/QERDS-Binding.md](qerds/QERDS-Binding.md) | QERDS message payload; PTE in the message body or as referenced attachments inside the QERDS evidence | XAdES/CAdES/PAdES depending on payload type | QERDS provider (which is, by definition, a QTSP) | Sketch |
| **REST direct B2B** | (Future) | JSON request body properties; PTE encoded as base64 or referenced by URI | XAdES/JAdES depending on document format | HTTP server with PTE-aware middleware | Future |
| **OID4VP / DCQL** | (Out of scope) | Verifiable Presentation; buyer wallet pulls from supplier wallet | N/A — this is a wallet-to-wallet protocol, not a document-with-PTE pattern | Buyer's wallet; supplier's wallet | SC5 Scenario 4 territory; not modelled here |

## What every binding answers

Each binding document specifies four things:

1. **The carrier.** Which container element holds the STM, the seal, and the TVE on the wire. This is the only thing that genuinely differs between bindings.
2. **The handler.** Which software role acts as the AP-equivalent — the message handler that calls APEIP outbound at send, calls APEIP `/verify` at receive, and writes the TVE back into the document.
3. **The seal format.** XAdES (XML), PAdES (PDF), CAdES (binary), or JAdES (JSON) — driven by what the document body is, not by the transport.
4. **Discovery.** How a sender determines whether a receiver supports the PTE before sending. Peppol uses SMP capability identifiers; mail uses (proposed) DNS TXT records; QERDS uses provider capability advertisements.

What every binding does **not** redefine:

- The STM and TVE data model — identical across bindings.
- The APEIP protocol — identical across bindings.
- The trust score calculation — identical across bindings.
- The cryptographic seal's role — always covers the document body + STM, never the TVE.

## Adoption strategy

The pragmatic path is: start with the Peppol binding (highest concentration of relevant traffic; SC5 pilots provide a concrete vehicle), then add the mail binding (largest installed base of B2B exchanges that *aren't* on Peppol), then QERDS (regulator-grade scenarios). Each new binding adds reach without re-litigating the trust model.

## Status of the documents in this folder

| Document | What "ready" would mean | Current state |
|---|---|---|
| `peppol/Oxalis-NG-Handler.md` | An open-source reference implementation deployable by any Peppol AP. | Architecture spec, no implementation yet. SC5 Pilot 1 is the first deployment target. |
| `mail/Mail-Binding.md` | A complete MIME structure spec + handler integration model + discovery mechanism. | Sketch — demonstrates feasibility, identifies design questions, not a deployable spec. |
| `qerds/QERDS-Binding.md` | Same as mail, plus alignment with ETSI EN 319 522 (ERDS) and eIDAS Art. 44 requirements. | Sketch — same level of completeness as the mail binding. |
| `rest/` | (Folder not created) — would specify the REST request/response shape and a discovery mechanism for arbitrary HTTP endpoints. | Not started. |

The mail and QERDS sketches exist to prove the trust layer is transport-independent. They are not pilot-ready and should not be implemented without further specification work.
