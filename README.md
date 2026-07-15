# SC5 — eInvoicing

## Overview

SC5 is the eInvoicing use case of the **WE BUILD Large-Scale Pilot**. It explores how European Business Wallets and verifiable electronic attestations can strengthen trust, authorization and interoperability in cross-border electronic invoicing processes.

The use case combines established eInvoicing infrastructure, including the **Peppol four-corner model**, with wallet-based attestations that provide machine-verifiable evidence of business relationships and authorization. The existing Peppol invoice format and transport mechanisms remain unchanged; instead, attestations are exchanged and verified at relevant decision points in the invoicing process.

SC5 covers five complementary scenarios:

1. **Supplier pre-approval**
   A buyer issues an Approved Supplier attestation to a supplier, allowing the supplier's authorization or contractual relationship to be verified before an invoice is processed.

2. **Service Provider authorization**
   A business issues an Authorized Service Provider attestation proving that a service provider is permitted to send or receive electronic invoices on its behalf.

3. **Service Provider authorization verifiable by a Tax Administration**
   The authorization of a service provider can also be verified by a Tax Administration when invoices or transaction data are submitted.

4. **Direct eInvoicing between Business Wallets**
   An eInvoice is exchanged directly between business systems using an eInvoice Attestation that cryptographically binds the invoice content to the supplier's verified business identity.

5. **Peppol trust enhancements**
   Additional trust services and attestations are combined with the Peppol infrastructure to improve the integrity, authenticity and traceability of eInvoicing exchanges.

The repository contains the SC5 scenario specifications, attestation definitions and related technical artefacts used as input for implementation, interoperability testing and pilot preparation.

## Repository structure

* [`Scenario/`](Scenario/) — Common concepts, actors, architecture and detailed specifications for the five SC5 scenarios
* [`Attestations/`](Attestations/) — Attestation schemas and related artefacts used by the SC5 scenarios

## Status

The material in this repository is developed as part of the WE BUILD specification and pilot activities. It represents the current SC5 working baseline and may evolve based on implementation feedback, interoperability testing and regulatory developments.
