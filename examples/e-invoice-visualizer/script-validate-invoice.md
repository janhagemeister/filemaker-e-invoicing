# Script: validate and visualize an invoice

FileMaker script for `erechnungvisualisierung.fmp12`. Requires FileMaker Pro 16 or later. Field names are the literal names in the example file.

## 1. Check the input

```text
Set Error Capture [ On ]
Allow User Abort [ Off ]

If [ IsEmpty ( Rechnung::Originaldatei ) ]
    Show Custom Dialog [ "No invoice" ;
        "Please select an XML or PDF file first." ]
    Exit Script [ Text Result: "Input file missing" ]
End If
```

## 2. Build the request

```text
Set Variable [ $request ; Value:
    JSONSetElement (
        "{}" ;
        [ "data" ;
          Base64EncodeRFC ( 4648 ; Rechnung::Originaldatei ) ;
          JSONString ]
    )
]

Set Variable [ $url ; Value: "<API-ENDPOINT>" ]
Set Variable [ $apiKey ; Value: Einstellungen::API_Key ]
```

`Base64EncodeRFC ( 4648 ; … )` is used so the payload carries no added line breaks.

## 3. Build the cURL options

```text
Set Variable [ $curlOptions ; Value:
    "--request POST" & ¶ &
    "--header " & Quote ( "Content-Type: application/json" ) & ¶ &
    If (
        not IsEmpty ( $apiKey ) ;
        "--header " & Quote ( "x-api-key: " & $apiKey ) & ¶ ;
        ""
    ) &
    "--data @$request" & ¶ &
    "--dump-header $responseHeaders" & ¶ &
    "--connect-timeout 15" & ¶ &
    "--max-time 120" & ¶ &
    "--show-error"
]
```

The `x-api-key` header is only added when a key is present.

## 4. Send the request

```text
Set Variable [ $response ; Value: "" ]
Set Variable [ $responseHeaders ; Value: "" ]

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

Set Field [ Rechnung::API_Antwort ; $response ]
```

Keep `Verify SSL Certificates` on in production. Do not use `--insecure`.

## 5. Handle HTTP status and transport errors

```text
Set Variable [ $headerLines ; Value:
    Substitute ( $responseHeaders ; Char ( 13 ) ; "" )
]
Set Variable [ $headerIndex ; Value: ValueCount ( $headerLines ) ]
Set Variable [ $httpStatusLine ; Value: "" ]

Loop [ Flush: Always ]
    Exit Loop If [ $headerIndex < 1 ]
    Set Variable [ $httpStatusLine ; Value:
        GetValue ( $headerLines ; $headerIndex )
    ]
    Exit Loop If [ Left ( $httpStatusLine ; 5 ) = "HTTP/" ]
    Set Variable [ $headerIndex ; Value: $headerIndex - 1 ]
End Loop

Set Variable [ $httpStatus ; Value:
    If ( Left ( $httpStatusLine ; 5 ) = "HTTP/" ;
        GetAsNumber ( MiddleWords ( $httpStatusLine ; 2 ; 1 ) ) ; 0 )
]

If [ $fmError ≠ 0 and $httpStatus = 0 ]
    Set Variable [ $result ; Value:
        JSONSetElement ( "{}" ;
            [ "status" ; "transport_error" ; JSONString ] ;
            [ "filemaker_error" ; $fmError ; JSONNumber ] ;
            [ "detail" ; $fmErrorDetail ; JSONString ]
        )
    ]
    Exit Script [ Text Result: $result ]
End If
```

The header dump is scanned from the end so redirects do not leave a stale status line in place.

## 6. Check the JSON response

```text
Set Variable [ $formatted ; Value: JSONFormatElements ( $response ) ]

If [ Left ( $formatted ; 1 ) = "?" ]
    Exit Script [ Text Result:
        JSONSetElement ( "{}" ;
            [ "status" ; "invalid_response" ; JSONString ] ;
            [ "http_status" ; $httpStatus ; JSONNumber ] ;
            [ "response" ; $response ; JSONString ]
        )
    ]
End If

If [ JSONGetElementType ( $response ; "" ) ≠ JSONObject ]
    Exit Script [ Text Result: "The API response is not a JSON object." ]
End If

Set Variable [ $responseKeys ; Value: JSONListKeys ( $response ; "" ) ]
Set Variable [ $detail ; Value:
    If ( not IsEmpty ( FilterValues ( $responseKeys ; "detail" ) ) ;
        JSONGetElement ( $response ; "detail" ) ; "" )
]
Set Variable [ $message ; Value:
    If ( not IsEmpty ( FilterValues ( $responseKeys ; "Message" ) ) ;
        JSONGetElement ( $response ; "Message" ) ; "" )
]
Set Variable [ $error ; Value:
    If ( not IsEmpty ( FilterValues ( $responseKeys ; "error" ) ) ;
        JSONGetElement ( $response ; "error" ) ; "" )
]

If [ $httpStatus < 200 or $httpStatus ≥ 300 ]
    Exit Script [ Text Result:
        JSONSetElement ( "{}" ;
            [ "status" ; "http_error" ; JSONString ] ;
            [ "http_status" ; $httpStatus ; JSONNumber ] ;
            [ "detail" ; $detail ; JSONString ] ;
            [ "message" ; $message ; JSONString ] ;
            [ "error" ; $error ; JSONString ]
        )
    ]
End If
```

Keys are checked with `JSONListKeys` and `FilterValues` before they are read, so a missing key is not confused with an empty value.

## 7. Take over the validations

```text
Set Variable [ $validations ; Value:
    If ( not IsEmpty ( FilterValues ( $responseKeys ; "validation" ) ) ;
        JSONListValues ( $response ; "validation" ) ; "" )
]

Set Field [ Rechnung::Validierungen ; $validations ]
```

This is the list of checks that actually ran — the basis for telling `valid` apart from `validation_incomplete`.

## 8. Count and store the SVRL reports

```text
Set Variable [ $reportCount ; Value:
    If ( not IsEmpty ( FilterValues ( $responseKeys ; "reports" ) ) ;
        ValueCount ( JSONListKeys ( $response ; "reports" ) ) ; 0 )
]
Set Variable [ $i ; Value: 0 ]
Set Variable [ $failedAssertCount ; Value: 0 ]

Loop [ Flush: Always ]
    Exit Loop If [ $i ≥ $reportCount ]

    Set Variable [ $report ; Value:
        JSONGetElement ( $response ; "reports[" & $i & "]" )
    ]
    Set Variable [ $failedAssertCount ; Value:
        $failedAssertCount +
        PatternCount ( $report ; "<svrl:failed-assert" )
    ]

    Set Variable [ $i ; Value: $i + 1 ]
End Loop

Set Field [ Rechnung::Pruefberichte ;
    If ( $reportCount > 0 ;
        JSONFormatElements ( JSONGetElement ( $response ; "reports" ) ) ; "" )
]
```

An empty `reports` array means neither "valid" nor "invalid".

## 9. Store the visualization

```text
Set Variable [ $pdfBase64 ; Value:
    If ( not IsEmpty ( FilterValues ( $responseKeys ; "pdf" ) ) ;
        JSONGetElement ( $response ; "pdf" ) ; "" )
]

If [ not IsEmpty ( $pdfBase64 ) ]
    Set Variable [ $pdfContainer ; Value:
        Base64Decode (
            $pdfBase64 ;
            "Rechnungsvisualisierung.pdf"
        )
    ]
    Set Field [ Rechnung::Visualisierung ; $pdfContainer ]
End If
```

The file name passed to `Base64Decode` is essential — without it the container does not receive usable PDF data.

## 10. Take over the structured data

```text
Set Variable [ $content ; Value:
    If ( not IsEmpty ( FilterValues ( $responseKeys ; "content" ) ) ;
        JSONGetElement ( $response ; "content" ) ; "" )
]

If [ not IsEmpty ( $content ) ]
    Set Field [ Rechnung::StrukturierteDaten ;
        JSONFormatElements ( $content )
    ]
End If
```

## Resulting status

Write the summarized result to `Rechnung::Validierungsstatus`:

| Status | Condition |
|---|---|
| `transport_error` | FileMaker error, no HTTP status |
| `invalid_response` | Response is not parsable JSON |
| `http_error` | HTTP status outside 200–299 |
| `processing_error` | `detail`, `Message`, or `error` present |
| `validation_incomplete` | No XSD check in `validation`, or no evaluable report |
| `validation_failed` | `$failedAssertCount` > 0 for non-accepted rules |
| `valid` | XSD check present, no non-accepted `failed-assert` |
