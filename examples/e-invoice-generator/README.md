# E-Invoice Generator for FileMaker

**File:** `erechnungerstellung.fmp12`

**Requires:** FileMaker Pro 16 or later

This FileMaker example demonstrates how an electronic invoice is generated from a FileMaker solution, following [E-Rechnung in FileMaker erstellen](https://www.hagemeister-conception.de/blog/e-rechnung-in-filemaker-erstellen).

The process is:

```text
Invoice data → generate XML → generate PDF/A-3 if applicable → validate → only then send
```

The invoice is processed by the service in memory only; no invoice data is stored there.

The full script is in [script-generate-invoice.md](script-generate-invoice.md).

## Fields (table `Rechnungen`)

| Field | Type | Purpose |
|---|---|---|
| `ErstellungsRequest` | Text | The request JSON that was sent |
| `ERechnungXML` | Container | Generated invoice XML |
| `SichtPDF` | Container | Human-readable PDF from FileMaker |
| `PDF_A_3` | Container | Hybrid PDF/A-3 with embedded XML |
| `API_Antwort` | Text | Full response, for diagnostics |
| `Erstellungsstatus` | Text | Summarized status |
| `Validierungsbericht` | Text/Container | Validation report |

The API key lives in a separate settings table: `Einstellungen::API_Key`. Header and line-item source fields are those of the host solution.

## Models

| Model | Format |
|---|---|
| `Facturxen16931` | Factur-X / ZUGFeRD, EN 16931 |
| `Facturxextended` | Factur-X / ZUGFeRD, EXTENDED |
| `Ubl21` | UBL 2.1 Invoice |
| `Ubl21cn` | UBL 2.1 Credit Note |
| `Cii16buncoupled` | UN/CEFACT CII D16B, uncoupled |
| `Cii16bcoupled` | UN/CEFACT CII D16B, coupled |

`EXTENDED` is not `EN16931` with a few extra fields. Repeatable elements such as `applicable_trade_tax`, `defined_trade_contact`, and `specified_trade_payment_terms` behave differently between the profiles, which is why EXTENDED is addressed through its own endpoint.

## Endpoints

```text
GET  <API-BASE-URL>/<SCHEMA-ENDPOINT>/<MODEL-NAME>
POST <API-BASE-URL>/<JSON-TO-XML-ENDPOINT>/<MODEL-NAME>
POST <API-BASE-URL>/<PDFA3-ENDPOINT>
POST <API-BASE-URL>/Facturxextended
```

Headers:

```text
Accept: application/json
Content-Type: application/json
x-api-key: <API-KEY>
```

Send `x-api-key` only when a key is present.

## Steps

### 1. Retrieve schema elements

`GET <API-BASE-URL>/<SCHEMA-ENDPOINT>/<MODEL-NAME>` returns a JSON array of four-element arrays (XML path, JSON path, field type, XML element). Check that the response is a JSON array before using it; handle error shapes separately. The schema — not the FileMaker file — is authoritative for element names.

### 2. Build the invoice JSON

Use `JSONSetElement` on the paths from the schema. Header excerpt:

```text
exchanged_document_context.guideline_specified_document_context_parameter.id.value
exchanged_document.id.value
exchanged_document.type_code.value
exchanged_document.issue_date_time.date_time_string.format
exchanged_document.issue_date_time.date_time_string.value
supply_chain_trade_transaction.applicable_header_trade_settlement.invoice_currency_code.value
```

Line items, looping with a zero-based index:

```text
supply_chain_trade_transaction.included_supply_chain_trade_line_item[$i]
    associated_document_line_document.line_id.value
    specified_trade_product.name.value
    specified_line_trade_delivery.billed_quantity.value
    specified_line_trade_delivery.billed_quantity.unit_code
    specified_line_trade_settlement.applicable_trade_tax.rate_applicable_percent.value
    specified_line_trade_settlement.specified_trade_settlement_line_monetary_summation.line_total_amount.value
```

Use the FileMaker JSON type constants: `JSONString`, `JSONNumber`, `JSONBoolean`, `JSONObject`, `JSONArray`. Do not send formatted number strings such as `"1.234,56"`. Store the finished request in `Rechnungen::ErstellungsRequest`.

See [../../sample-data/invoice.json](../../sample-data/invoice.json) and [../../docs/json-interface.md](../../docs/json-interface.md).

### 3. Generate the invoice XML

`POST <API-BASE-URL>/<JSON-TO-XML-ENDPOINT>/<MODEL-NAME>` with the invoice JSON as the body.

The response contains `message` and `xml_output`. `xml_output` is **plain text, not Base64**. Convert it to container data:

```text
TextEncode ( $xmlText ; "utf-8" ; 3 )
```

and store it in `Rechnungen::ERechnungXML`.

### 4. Generate the view PDF and the PDF/A-3

Produce the readable PDF with `Save Records as PDF` into `Rechnungen::SichtPDF`. This requires the *Allow printing* privilege. The invoice print layout and the correct record must be active.

Then `POST <API-BASE-URL>/<PDFA3-ENDPOINT>` with both parts Base64-encoded:

```text
"pdf" : Base64EncodeRFC ( 4648 ; Rechnungen::SichtPDF )
"xml" : Base64EncodeRFC ( 4648 ; Rechnungen::ERechnungXML )
```

Do not prefix the Base64 with `data:application/pdf;base64,`.

The response contains `message` and `pdf` (Base64). Decode it with a file name — the `.pdf` extension is what makes FileMaker treat the result as real container data:

```text
Base64Decode ( $pdfBase64 ; $fileName )
```

Store the result in `Rechnungen::PDF_A_3`.

### 5. Evaluate responses completely

Capture `Get ( LastError )` and `Get ( LastErrorDetail )` in the same first `Set Variable` step after `Insert from URL`. Then:

- Extract the last `HTTP/` status line from the dumped headers (scan backwards so redirects do not leave a stale status).
- Accept only HTTP 200–299.
- Verify the JSON root is a `JSONObject` (schema endpoint: `JSONArray`).
- Check for a `detail` key before treating a response as success.
- Write the full response to `Rechnungen::API_Antwort`.

### 6. Validate, then set the overall status

```text
transport_error, invalid_response, http_error,
xml_created, pdfa3_created,
validation_failed, created_and_validated
```

Only `created_and_validated` may release the invoice for automatic dispatch. A technically successful XML or PDF/A-3 file is not automatically a valid electronic invoice.

## Container and Base64 handling

| Direction | Handling |
|---|---|
| XML from response → container | plain text, then `TextEncode ( … ; "utf-8" ; 3 )` |
| PDF/A-3 from response → container | `Base64Decode ( $pdfBase64 ; $fileName )` with `.pdf` extension |
| Container → request | `Base64EncodeRFC ( 4648 ; <Container> )` |

## Acceptance test

1. A valid invoice per model
2. Missing required field, wrong data type, invalid model name
3. Empty or invalid JSON
4. Correct, missing, and invalid API key
5. TLS error, network error, forced timeout
6. Valid and corrupted view PDF / XML
7. HTTP error with a JSON body, and a non-JSON response
8. A business-level invalid invoice that the validator rejects
9. Execution in FileMaker Pro and as a FileMaker Server script

## Security

Store the API key in a protected settings table or in secrets management. Enable `Verify SSL Certificates` on every HTTPS call. Do not use `--fail` (error bodies would be discarded) or `--insecure`. All URLs, model names, and credentials shown here are placeholders.

## Technical reference

- [`Insert from URL`](https://help.claris.com/en/pro-help/content/insert-from-url.html)
- [Supported cURL options](https://help.claris.com/en/pro-help/content/curl-options.html)
- [`Save Records as PDF`](https://help.claris.com/en/pro-help/content/save-records-as-pdf.html)
- [`TextEncode`](https://help.claris.com/en/pro-help/content/textencode.html)
- [`Base64Decode`](https://help.claris.com/en/pro-help/content/base64decode.html)
- [Working with the JSON functions](https://help.claris.com/en/pro-help/content/json-functions.html)

See [DISCLAIMER.md](../../DISCLAIMER.md).

Further information: https://www.hagemeister-conception.de/
