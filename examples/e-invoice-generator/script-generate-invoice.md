# Script: generate an electronic invoice

FileMaker script outline for `erechnungerstellung.fmp12`. Requires FileMaker Pro 16 or later. Result field names are the literal names in the example file. Header and line-item source fields belong to the host solution.

`--data @$requestData` passes the FileMaker variable contents directly. Capture `Get ( LastError )` and `Get ( LastErrorDetail )` in the same first `Set Variable` step after every `Insert from URL`.

## 1. Retrieve schema elements

```text
Set Error Capture [ On ]
Allow User Abort [ Off ]

Set Variable [ $schemaName ; Value: "Facturxen16931" ]
Set Variable [ $apiKey ; Value: Einstellungen::API_Key ]
Set Variable [ $url ; Value:
    "<API-BASE-URL>/<SCHEMA-ENDPOINT>/" & $schemaName
]
Set Variable [ $schemaResponse ; Value: "" ]
Set Variable [ $responseHeaders ; Value: "" ]

Set Variable [ $curlOptions ; Value:
    "--request GET" & ¶ &
    "--header " & Quote ( "Accept: application/json" ) & ¶ &
    If ( not IsEmpty ( $apiKey ) ;
        "--header " & Quote ( "x-api-key: " & $apiKey ) & ¶ ; "" ) &
    "--dump-header $responseHeaders" & ¶ &
    "--connect-timeout 15" & ¶ &
    "--show-error" & ¶ &
    "--max-time 120"
]

Insert from URL [
    Select ;
    With dialog: Off ;
    Target: $schemaResponse ;
    $url ;
    Verify SSL Certificates ;
    cURL options: $curlOptions
]

Set Variable [ $lastError ; Value:
    JSONSetElement ( "{}" ;
        [ "code" ; Get ( LastError ) ; JSONNumber ] ;
        [ "detail" ; Get ( LastErrorDetail ) ; JSONString ]
    )
]
Set Variable [ $fmError ; Value: JSONGetElement ( $lastError ; "code" ) ]
Set Variable [ $formattedResponse ; Value:
    JSONFormatElements ( $schemaResponse )
]

If [ $fmError ≠ 0 or Left ( $formattedResponse ; 1 ) = "?" ]
    Exit Script [ Text Result: $lastError ]
End If

If [ JSONGetElementType ( $schemaResponse ; "" ) ≠ JSONArray ]
    Exit Script [ Text Result: "The schema response is not a JSON array." ]
End If
```

## 2. Build the invoice header

```text
Set Variable [ $json ; Value: "{}" ]

Set Variable [ $json ; Value:
    JSONSetElement ( $json ;
        [ "exchanged_document_context.guideline_specified_document_context_parameter.id.value" ;
          "urn:cen.eu:en16931:2017" ; JSONString ] ;
        [ "exchanged_document.id.value" ;
          Rechnungskopfdaten::Rechnungsnummer ; JSONString ] ;
        [ "exchanged_document.type_code.value" ;
          Rechnungskopfdaten::Dokumententyp ; JSONString ] ;
        [ "exchanged_document.issue_date_time.date_time_string.format" ;
          "102" ; JSONString ] ;
        [ "exchanged_document.issue_date_time.date_time_string.value" ;
          dateformat102 ( Rechnungskopfdaten::Rechnungsdatum ) ; JSONString ] ;
        [ "supply_chain_trade_transaction.applicable_header_trade_settlement.invoice_currency_code.value" ;
          "EUR" ; JSONString ]
    )
]
```

`dateformat102` is a custom function that returns `YYYYMMDD`. Seller, buyer, tax breakdown, totals, and payment data must still be added from the retrieved schema.

## 3. Append line items

```text
Go to Record/Request/Page [ First ]

Loop [ Flush: Always ]
    Set Variable [ $i ; Value: Get ( RecordNumber ) - 1 ]
    Set Variable [ $path ; Value:
        "supply_chain_trade_transaction.included_supply_chain_trade_line_item[" & $i & "]"
    ]

    Set Variable [ $json ; Value:
        JSONSetElement ( $json ;
            [ $path & ".associated_document_line_document.line_id.value" ;
              Get ( RecordNumber ) ; JSONString ] ;
            [ $path & ".specified_trade_product.name.value" ;
              Rechnungspositionen::Artikelname ; JSONString ] ;
            [ $path & ".specified_line_trade_delivery.billed_quantity.value" ;
              Rechnungspositionen::Menge ; JSONNumber ] ;
            [ $path & ".specified_line_trade_delivery.billed_quantity.unit_code" ;
              Rechnungspositionen::EinheitenCode ; JSONString ] ;
            [ $path & ".specified_line_trade_settlement.applicable_trade_tax.rate_applicable_percent.value" ;
              Steuerschlüssel::Steuersatz ; JSONNumber ] ;
            [ $path & ".specified_line_trade_settlement.specified_trade_settlement_line_monetary_summation.line_total_amount.value" ;
              Rechnungspositionen::GesamtNetto ; JSONNumber ]
        )
    ]

    Go to Record/Request/Page [ Next ; Exit after last: On ]
End Loop

Set Variable [ $formattedJSON ; Value: JSONFormatElements ( $json ) ]
If [ Left ( $formattedJSON ; 1 ) = "?" ]
    Exit Script [ Text Result: $formattedJSON ]
End If
If [ JSONGetElementType ( $json ; "" ) ≠ JSONObject ]
    Exit Script [ Text Result: "The invoice JSON must be an object." ]
End If

Set Field [ Rechnungen::ErstellungsRequest ; $json ]
```

## 4. Generate the invoice XML

```text
Set Variable [ $model ; Value: "Facturxen16931" ]
Set Variable [ $url ; Value:
    "<API-BASE-URL>/<JSON-TO-XML-ENDPOINT>/" & $model
]
Set Variable [ $response ; Value: "" ]
Set Variable [ $responseHeaders ; Value: "" ]
Set Variable [ $requestData ; Value: $json ]

Set Variable [ $curlOptions ; Value:
    "--request POST" & ¶ &
    "--header " & Quote ( "Content-Type: application/json" ) & ¶ &
    If ( not IsEmpty ( $apiKey ) ;
        "--header " & Quote ( "x-api-key: " & $apiKey ) & ¶ ; "" ) &
    "--data @$requestData" & ¶ &
    "--dump-header $responseHeaders" & ¶ &
    "--connect-timeout 15" & ¶ &
    "--show-error" & ¶ &
    "--max-time 120"
]

Insert from URL [
    Select ;
    With dialog: Off ;
    Target: $response ;
    $url ;
    Verify SSL Certificates ;
    cURL options: $curlOptions
]

Set Variable [ $lastError ; Value:
    JSONSetElement ( "{}" ;
        [ "code" ; Get ( LastError ) ; JSONNumber ] ;
        [ "detail" ; Get ( LastErrorDetail ) ; JSONString ]
    )
]
Set Variable [ $fmError ; Value: JSONGetElement ( $lastError ; "code" ) ]
Set Variable [ $fmErrorDetail ; Value: JSONGetElement ( $lastError ; "detail" ) ]
Set Field [ Rechnungen::API_Antwort ; $response ]
```

## 5. Check HTTP status and take over the XML

Scan `$responseHeaders` from the end for the last line that starts with `HTTP/` (same block as in [../e-invoice-visualizer/script-validate-invoice.md](../e-invoice-visualizer/script-validate-invoice.md), step 5). Then:

```text
Set Variable [ $formattedResponse ; Value: JSONFormatElements ( $response ) ]

If [ ( $fmError ≠ 0 and $httpStatus = 0 ) or
     Left ( $formattedResponse ; 1 ) = "?" ]
    Exit Script [ Text Result: $lastError ]
End If
If [ JSONGetElementType ( $response ; "" ) ≠ JSONObject ]
    Exit Script [ Text Result: "The API response is not a JSON object." ]
End If

Set Variable [ $responseKeys ; Value: JSONListKeys ( $response ; "" ) ]
Set Variable [ $xmlText ; Value:
    If ( not IsEmpty ( FilterValues ( $responseKeys ; "xml_output" ) ) ;
        JSONGetElement ( $response ; "xml_output" ) ; "" )
]
Set Variable [ $apiDetail ; Value:
    If ( not IsEmpty ( FilterValues ( $responseKeys ; "detail" ) ) ;
        JSONGetElement ( $response ; "detail" ) ; "" )
]

If [ $httpStatus < 200 or $httpStatus ≥ 300 or
     not IsEmpty ( $apiDetail ) or IsEmpty ( $xmlText ) ]
    Exit Script [ Text Result:
        JSONSetElement ( "{}" ;
            [ "status" ; "api_error" ; JSONString ] ;
            [ "http_status" ; $httpStatus ; JSONNumber ] ;
            [ "detail" ; $apiDetail ; JSONString ]
        )
    ]
End If

Set Field [ Rechnungen::ERechnungXML ; TextEncode ( $xmlText ; "utf-8" ; 3 ) ]
```

`xml_output` is plain text, not Base64. `\"` and `\n` belong only to the JSON encoding; `JSONGetElement` returns the XML text.

## 6. Generate the view PDF

```text
Set Variable [ $pdfContainer ; Value: "" ]

Save Records as PDF [
    Restore ;
    Save to: Target ;
    Target: $pdfContainer ;
    With dialog: Off ;
    Current record
]

Set Variable [ $pdfError ; Value: Get ( LastError ) ]
If [ $pdfError ≠ 0 or IsEmpty ( $pdfContainer ) ]
    Exit Script [ Text Result: "The view PDF could not be created." ]
End If
Set Field [ Rechnungen::SichtPDF ; $pdfContainer ]
```

Requires the *Allow printing* privilege. The invoice print layout and the correct record must be active.

## 7. Generate the PDF/A-3

```text
If [ IsEmpty ( Rechnungen::SichtPDF ) or
     IsEmpty ( Rechnungen::ERechnungXML ) ]
    Exit Script [ Text Result: "View PDF or invoice XML is missing." ]
End If

Set Variable [ $requestData ; Value:
    JSONSetElement ( "{}" ;
        [ "pdf" ; Base64EncodeRFC ( 4648 ; Rechnungen::SichtPDF ) ; JSONString ] ;
        [ "xml" ; Base64EncodeRFC ( 4648 ; Rechnungen::ERechnungXML ) ; JSONString ]
    )
]
Set Variable [ $url ; Value: "<API-BASE-URL>/<PDFA3-ENDPOINT>" ]
```

Send with the same cURL options as step 4, then evaluate HTTP status and JSON as in step 5. Read `pdf` (Base64, no `data:` prefix):

```text
Set Variable [ $fileName ; Value:
    "E-Rechnung " & Rechnungskopfdaten::Rechnungsnummer & ".pdf"
]
Set Field [ Rechnungen::PDF_A_3 ;
    Base64Decode ( $pdfBase64 ; $fileName )
]
```

The file name passed to `Base64Decode` is what makes FileMaker produce usable PDF container data.

## Resulting status

Write the summarized result to `Rechnungen::Erstellungsstatus`:

| Status | Meaning |
|---|---|
| `transport_error` | Service not reachable |
| `invalid_response` | Response is not parsable JSON / not a JSON object |
| `http_error` | HTTP status outside 200–299 |
| `xml_created` | XML generated, not yet validated |
| `pdfa3_created` | PDF/A-3 generated, not yet validated |
| `validation_failed` | Validation reported errors |
| `created_and_validated` | Generated **and** validated |

Only `created_and_validated` may release the invoice for automatic dispatch.
