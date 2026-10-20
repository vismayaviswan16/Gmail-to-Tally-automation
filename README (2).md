# Gmail → Tally Invoice Automation (n8n)

An [n8n](https://n8n.io) workflow that reads invoice emails from Gmail, extracts the invoice data with a **local** LLM (Ollama), and imports **Sales, Purchase or Journal vouchers into Tally**. Attachments are archived in Google Drive and every result is logged in Google Sheets.

Built for Indian GST invoices and Tally. Invoice content never leaves your machine: extraction runs on local models.

> **Status: experimental.** Test with a dummy Tally company before pointing it at real books. See [Disclaimer](#disclaimer).

---

## How it works

```mermaid
flowchart TD
    A[Gmail Trigger<br/>unread invoice mails, every minute] --> B{Subject matches<br/>invoice / bill / receipt?}
    B -->|yes| C[Upload attachment to Drive<br/>'unprocessed' folder]
    C --> D{PDF has<br/>extractable text?}
    D -->|yes| F[Ollama qwen3:14b<br/>text → invoice JSON]
    D -->|no: scan / image| E[Ghostscript → PNG<br/>qwen2.5vl:7b OCR]
    E --> F
    F --> G[Classify parties<br/>Sales / Purchase / Journal / Review]
    G --> H{Validation}
    H -->|fail| X[(Errors tab)]
    H -->|pass| I{Duplicate?}
    I -->|yes| X
    I -->|no| J{Route by direction}
    J -->|Review| X
    J -->|Sales / Purchase / Journal| K[Build ledger + voucher XML]
    K --> L[Import into Tally]
    L --> M{Tally confirmed?}
    M -->|yes| N[(Invoice + Daily report tabs)<br/>move file to 'processed' folder]
    M -->|no| X
```

1. **Gmail Trigger** polls every minute for unread mail matching a search query (subject or body containing *invoice*, *bill* or *receipt*). An `If` node then keeps mails whose **subject** contains *invoice*, *bill*, *tax invoice* or *receipt*.
2. Each attachment is uploaded to a Google Drive folder and its PDF text is extracted.
3. If no text can be extracted (scanned PDF or image), the file is converted to PNG with Ghostscript and read by the `qwen2.5vl:7b` vision model.
4. `qwen3:14b` turns the text into structured invoice JSON.
5. A Code node cleans the GSTINs and decides whether your company is the vendor or the buyer. That sets the voucher direction:

   | Your company is… | Direction |
   |---|---|
   | the vendor | **Sales** |
   | the buyer, vendor has a GSTIN | **Purchase** |
   | the buyer, vendor has no GSTIN | **Journal** |
   | not identifiable | **Review** (logged to the Errors tab, nothing posted) |

6. **Invoice Validation** checks required fields, `is_invoice`, GSTIN format, PAN-vs-GSTIN consistency, date format (`YYYY-MM-DD`) and that line items, tax and total add up (tolerance ±1.00). Failures go to the Errors tab.
7. **Duplicate check:** `invoice_key` is `<party>|<invoice number>` (whitespace removed, upper case). The party is the buyer for Sales and the vendor otherwise (GSTIN, or name if there is none). It is looked up in the `Invoice Key` column of the first sheet tab. Duplicates go to the Errors tab.
8. Ledgers are imported into Tally first, then the voucher. If Tally confirms (created/altered > 0, no exceptions), the invoice is logged to the Invoice and Daily report tabs and the Drive file moves to the processed folder. Otherwise the error is logged to the Errors tab.

---

## Requirements

| Requirement | Notes |
|---|---|
| **n8n (self-hosted)** | Uses the *Execute Command* and *Read/Write Files from Disk* nodes. Recent n8n versions may disable or restrict these by default, so enable them if the workflow errors on those nodes. |
| **Ollama** | Reachable at `127.0.0.1:11434`. Pull `qwen3:14b` and `qwen2.5vl:7b`. If n8n runs in Docker, change the two Ollama URLs (for example to `host.docker.internal`). |
| **Ghostscript** | The workflow calls `gswin64c` and writes temp files to `C:\n8n-temp\`. **Windows only as shipped**; create that folder first. |
| **Tally** | XML/HTTP server enabled (default `localhost:9000`), with your company open. |
| **Google** | Gmail, Google Drive and Google Sheets credentials in n8n. |

A machine able to run a 14B model comfortably is recommended. Extraction runs have a 10-minute timeout.

---

## Setup

1. Import the workflow JSON into n8n (**Workflows → Import from file**).
2. Create **Gmail**, **Google Drive** and **Google Sheets** credentials in n8n and select them on each Gmail, Drive and Sheets node. Credentials are not included in the file.
3. Create the Google Sheet and the two Drive folders (`unprocessed`, `processed`). See [Google Sheet layout](#google-sheet-layout).
4. Replace every placeholder listed below. In n8n, search the workflow for `YOUR_` to find most of them.
5. Start Tally with the company open, then run the workflow on **one test email** before activating it.

### Placeholders to replace

| Placeholder | Where | What to enter |
|---|---|---|
| `YOUR_GOOGLE_SHEET_ID` | Find Invoice, Invoice sheet, Daily report sheet, Review, Append row in sheet / sheet2 / sheet3 | Your spreadsheet. **Re-select the tab** on each node: *Find Invoice* and *Invoice sheet* use the first tab, *Daily report sheet* the daily-report tab, the other four the Errors tab. The `gid=` numbers in the URLs belong to a different sheet. |
| `YOUR_UNPROCESSED_FOLDER_ID` | Upload file | ID of the Drive folder for incoming attachments. |
| `YOUR_PROCESSED_FOLDER_ID` | Move file | ID of the Drive folder for processed files. |
| Tally URL `http://localhost:9000` | All six `HTTP Request [ledger]` / `[voucher]` nodes | Change on **every** one of the six nodes if Tally is not on the same machine as n8n. |
| `YOUR_TALLY_COMPANY_NAME` | Purchase, Sales and Journal code nodes | Company name exactly as shown in Tally. |
| `YOUR_COMPANY_NAME` | Purchase, Sales and Journal code nodes; Build Ollama Extraction Prompt; Build Qwen OCR Request | Your company's legal name. |
| `YOUR_COMPANY_SHORT_NAME` | Build Ollama Extraction Prompt, Build Qwen OCR Request | The short name your company goes by on invoices. |
| `YOUR_COMPANY_GSTIN`, `_2`, `_3` | `Code in JavaScript` (`MY_COMPANY_GSTINS`), Purchase, Sales, Journal and both prompt nodes | Your GSTIN(s). Unused placeholders never match anything, so extras can stay or be deleted. |
| `YOUR_COMPANY_PAN` | Build Ollama Extraction Prompt | Your PAN. |
| `YOUR_COMPANY_GSTIN_VARIANT` | Build Qwen OCR Request | An example of how OCR might misread your GSTIN, or delete that example paragraph. |
| `yourcompanyname` | `Code in JavaScript` (`MY_COMPANY_NAME_FRAGMENT`) | Lowercase letters and digits only, no spaces. A party whose normalised name contains this text is treated as your company (`"Acme Traders Pvt Ltd"` → `"acmetraders"`). |
| `YOUR_COMPANY_STATE`, `YOUR_COMPANY_PINCODE`, `YOUR_PINCODE`, `YOUR_CITY`, `YOUR_ADDRESS_LINE_1`–`3`, `YOUR_LUT_NUMBER`, `YOUR_COMPANY_EMAIL` | Purchase, Sales and Journal code nodes | State (spelled as in Tally; the Sales node compares it with the place of supply to choose intra-state vs inter-state tax), pincode, city, address lines, LUT number and e-mail. |

Also replace the "known name variations" list in **Build Ollama Extraction Prompt**, which currently repeats `YOUR_COMPANY_NAME`, with your real spelling variants. The model uses it to recognise your company on invoices.

---

## Google Sheet layout

Column names must match **exactly**.

| Tab | Used by | Columns |
|---|---|---|
| **First tab** | Find Invoice (duplicate lookup), Invoice sheet | `Invoice Key`, `Invoice Number`, `Invoice Date`, `Vendor Name`, `Vendor GSTIN`, `Buyer Name`, `Buyer GSTIN`, `Voucher Direction`, `Tally Created Date`, `Gmail`, `Drive link` |
| **Errors tab** | Review, Append row in sheet / sheet2 / sheet3 | `Invoice Date`, `Gmail`, `Error`, `Drive`, `Invoice Key`, `Invoice Number`, `Vendor Name`, `Vendor GSTIN`, `Buyer Name`, `Buyer GSTIN`, `Voucher Direction` |
| **Daily report tab** | Daily report sheet | `Invoice Key`, `Invoice Number`, `Invoice Date`, `Vendor Name`, `Vendor GSTIN`, `Buyer Name`, `Buyer GSTIN`, `Voucher Direction`, `Tally Created Date`, `Gmail`, `drive link` |

Note `Drive link` on the first tab but `drive link` (lowercase) on the daily-report tab.

---

## Behaviour to be aware of

- **Voucher dates.** The extraction prompt tells the model to set `invoice_date` to today's date, and the Purchase node dates its vouchers with today's date (IST), not the date printed on the invoice. Check this suits your accounting before use.
- **Financial year.** `applicableFrom` is hard-coded as `20260401` in the Purchase and Journal nodes. Update it for your financial year.
- **Optional vouchers.** The Sales node creates vouchers as *Optional* (`CONFIG.isOptional: true`) with a narration prefix, so a person can review them in Tally before they count.
- **Drive-link UDF.** The user-defined field index `47501` is specific to the original Tally company. Check it against yours or remove the field (the Journal node has a note about this).
- **Masters.** The cost centre defaults to `Admin`, and the Sales node creates the masters it needs (units, godowns, cost centres). Review `CONFIG` in the Sales node before the first run.
- **Ledger names.** Ledger and group names such as `Sundry Debtors`, `Sales Accounts`, `Purchase Accounts`, `CGST Input` and `Round Off` are hard-coded. Your chart of accounts must match, or the code nodes must be edited.
- **Self-billing** (same GSTIN on both sides) is not supported; the duplicated GSTIN is cleared.
- **Silent failures.** Several nodes continue on error. Watch the Errors tab, and consider adding an error-workflow notification.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Everything lands in **Review** | Your company's name fragment or GSTIN placeholders weren't replaced, so no party is recognised as yours. |
| Tally nodes return no confirmation | Tally isn't running, the company isn't open, the XML server is disabled, or only some of the six HTTP nodes were updated. |
| `Execute Command` / file nodes fail | Ghostscript isn't installed, `C:\n8n-temp\` doesn't exist, or n8n restricts these nodes. |
| Ollama connection refused | Ollama isn't running, or n8n is in Docker and `127.0.0.1` points at the container. |
| Valid invoices rejected as duplicates | Duplicate lookup reads the `Invoice Key` column of the **first tab**; make sure the right tab is selected. |
| Totals "do not match" errors | OCR or the model misread an amount. The Code node removes obvious misread total rows, but not every case. |

---

## Privacy

- Invoices are processed by local models; nothing is sent to an external AI service.
- Do **not** commit exported credentials, real invoices or your own company details back to a public repository.

---

## Disclaimer

This software is provided **as is**, without warranty of any kind. It uses LLM-based extraction, which can misread invoices. **Review every voucher before relying on it for GST filings, audits or financial statements.** You are responsible for the accuracy of your books.

---

## License

Released under the MIT License. Add a `LICENSE` file with your name and the year, or replace this section with your preferred license.

---

## Contributing

Issues and pull requests are welcome. Useful areas: Linux/macOS support for the PDF-to-image step, configurable ledger names, moving company settings into a single config node, and tests for the GSTIN and party-classification logic.
