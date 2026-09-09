# Architecture

A robust FileMaker e-invoicing implementation separates business logic from technical document processing.

```text
FileMaker business data
        ↓
Structured invoice model
        ↓
JSON interface
        ↓
Micro web service (conversion / validation / visualization)
        ↓
CII or UBL
        ↓
Validation
        ↓
XML or PDF/A-3
```

FileMaker remains the system that knows the meaning of the transaction. External technical components should not be expected to infer missing business semantics.

## Requirements

The examples require **FileMaker Pro 16 or later** (JSON functions and `Insert from URL` with cURL options).

Native FileMaker XML (XSLT 1.0) and PDF output are not sufficient for EN 16931 production documents: there is no native XSD or Schematron validation, and FileMaker cannot write PDF/A-3 with embedded XML.

## Division of responsibility

| Concern | Where it lives |
|---|---|
| Invoice data, workflow, business semantics | FileMaker |
| VAT treatment, invoice type, references, allowances, charges, payment terms, tax breakdown | FileMaker |
| Schema element lists per model | Service (schema endpoint) |
| CII / UBL serialization | Service |
| PDF/A-3 assembly | Service |
| XSD and Schematron validation | Service |
| Human-readable visualization | Service |

## Deliberately not an add-on

A FileMaker add-on would ship a data model and impose it on the host solution. These examples do not. Existing FileMaker data models stay untouched; the service sits outside the file and is updated centrally when profiles, code lists, schemas, or validation artifacts change.

Scripts from the example files can be copied as a starting point. Replace field names and calculations with those of the host solution. Do not copy the example data model.

## No data at rest

The service processes the invoice in memory and stores no invoice data. Persistence — original file, request, full API response, timestamp — is the FileMaker solution's job, and is what makes the audit trail possible.

## Schema first

Element names and structure are not hard-coded in the FileMaker file. The generator retrieves them per model:

```text
GET <API-BASE-URL>/<SCHEMA-ENDPOINT>/<MODEL-NAME>
```

The response is a JSON array of four-element arrays: XML path, JSON path, field type, XML element. An error object must not be imported as a schema list.

This keeps the FileMaker file stable across profile changes and makes the difference between profiles — for example the repeatable `applicable_trade_tax`, `defined_trade_contact`, and `specified_trade_payment_terms` in EXTENDED — visible instead of assumed.

See [json-interface.md](json-interface.md).

## Transport

All calls use `Insert from URL` with cURL options, HTTPS, and `Verify SSL Certificates` enabled:

```text
--request GET|POST
--header "Accept: application/json"
--header "Content-Type: application/json"
--header "x-api-key: <API-KEY>"
--data @$requestData
--dump-header $responseHeaders
--connect-timeout 15
--max-time 120
--show-error
```

The `x-api-key` header is only sent when an API key is present. Do not use `--fail`: HTTP 400, 403, and 500 must still deliver their JSON error body to FileMaker.
