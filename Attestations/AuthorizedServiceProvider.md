# Authorized Service Provider Attestation

**Version:** Draft  
**Date:** 29-04-2026  
**Lead author:** Rune Kjørlaug

## Attestation introduction

The Authorized Service Provider Attestation proves that a legal entity (the mandating business) has authorised a service provider to act on its behalf for one or more explicitly defined business or regulatory functions.

The attestation enables relying parties — including Peppol Service Providers, Access Points, tax authorities, and other intermediaries — to verify that a service provider is legitimately empowered to act in a representative capacity for a business, for purposes such as:

- sending electronic invoices to third parties,
- receiving electronic invoices,
- submitting VAT data or transaction-level reports to tax authorities,
- fulfilling regulatory or network-mandated reporting obligations.

The attestation does not assert legal identity details of the mandating business or the service provider. Legal identity SHALL be established through separate legal person attestations (e.g. EUCC). Instead, the attestation includes a reference to the mandating business, sufficient for efficient processing by relying parties, especially tax authorities.

## Attribute specification

The attestation attributes are defined in the tables below:

- The first column specifies the identifiers of the attestation attributes. The attribute identifiers in this column SHALL be used in requests and responses. There SHALL be at most one attribute with the same attribute identifier in each attestation.
- The second column describes the meaning of the attribute.
- The third column specifies whether the presence of the attribute in an attestation is mandatory (M) or optional (O).
- The fourth column indicates how the data elements SHALL be encoded, using the CDDL representation types defined in [RFC 8610].

> **NOTE:** If an attribute is indicated as mandatory, this solely means that the Issuer SHALL ensure that this element is present in the attestation. It does not imply that a Relying Party is required to request such an attribute when interacting with the Wallet Instance. Neither does it imply that the User cannot refuse to release a mandatory attribute if requested.

### Alignment with VAT in the Digital Age (ViDA)

The Authorized Service Provider Attestation (SPMA) is explicitly designed to support the VAT in the Digital Age (ViDA) framework, including:

- mandatory digital reporting of VAT-relevant transaction data,
- increased reliance on intermediaries and service providers,
- cross-border interoperability of mandates for VAT reporting.

Under ViDA, taxable persons may fulfil VAT reporting obligations directly or via authorised intermediaries. The SPMA provides a wallet-native and machine-verifiable mechanism to:

- prove that a service provider is authorised to submit VAT or transaction data on behalf of a taxable person,
- allow tax authorities to validate mandates without bilateral arrangements,
- support cross-border recognition of mandates where issued as PubEAA or QEAA.

The attestation does not assert VAT data or reporting results; it only asserts the existence, scope, and validity period of the mandate.

## Model

The Authorized Service Provider Attestation consists of a mandate object, a reference to the mandating business, a structured mandate scope, and an optional authority reference.

```
Authorized Service Provider
 ├─ Mandate attributes
 ├─ Mandating business reference
 ├─ Mandate scope
 │   └─ VAT / regulatory scope (optional)
 └─ Authority / source reference
```

Legal identity details are out of scope and SHALL be established via separate attestations (e.g. EUCC).

## Attributes

### Authorized Service Provider object

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `mandate_id` | Unique identifier of the mandate. | M | tstr |
| `mandate_type` | Type of mandate granted to the service provider. | M | tstr |
| `mandate_start_date` | Start date of the mandate. | M | tdate |
| `mandate_end_date` | End date of the mandate, if applicable. | O | tdate |
| `mandating_business` | Reference to the business that has granted the mandate. | M | Mandating business reference object |
| `mandate_scope` | Scope within which the service provider is authorised to act. | M | Mandate scope object |
| `authority_source` | Reference to the authentic source supporting the mandate. | O | Authority reference object |

### Mandating business reference object

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `vat_number` | VAT identification number of the mandating business. | M | tstr |
| `country_code` | ISO 3166-1 alpha-2 country code of VAT registration. | M | tstr |
| `business_identifier` | Additional business identifier (e.g. EUID), if applicable. | O | tstr |

> **Normative intent:** This object provides a reference to the taxable person and SHALL NOT be interpreted as a full legal identity description.

### Mandate scope object

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `scope_type` | Functional scope of the mandate. | M | tstr |
| `vat_reporting_scope` | VAT or regulatory reporting–specific constraints. | O | VAT reporting scope object |

### VAT reporting scope object

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `reporting_regime` | VAT reporting regime to which the mandate applies (e.g. EU ViDA). | M | tstr |
| `reporting_obligation_type` | Type of VAT or transaction reporting obligation covered by the mandate. | O | tstr |
| `reporting_frequency` | Frequency of reporting, where applicable. | O | tstr |

### Authority reference object

| **Attribute** | **Definition** | **M/O** | **Encoding format** |
|---|---|---|---|
| `source_type` | Type of authentic source underlying the mandate. | M | tstr |
| `source_identifier` | Identifier of the mandate record, registration, or decision. | M | tstr |
| `source_system` | System in which the mandate is registered or recognised. | O | tstr |
| `issuance_date` | Date on which the mandate was issued or registered. | O | tdate |
| `source_uri` | URI pointing to the authoritative record, if available. | O | tstr |

### Code lists

**`mandate_type`**

| Value | Description |
|---|---|
| `invoice_sending` | |
| `invoice_receiving` | |
| `vat_reporting` | |
| `transaction_reporting` | |

**`scope_type`**

| Value | Description |
|---|---|
| `einvoicing` | |
| `vat_reporting` | |
| `regulatory_reporting` | |

**`source_type`**

| Value | Description |
|---|---|
| `business_authentic_source` | |
| `peppol_authority` | |
| `tax_authority` | |
| `public_register` | |

## Integrity rules

- If `mandate_end_date` is present, it SHALL be later than `mandate_start_date`.
- If `scope_type` includes `vat_reporting`, the `vat_reporting_scope` object SHOULD be present.
- If `mandating_business.vat_number` is present, the Wallet Instance holding the attestation SHALL be bound to the same legal entity as identified by that VAT number.
