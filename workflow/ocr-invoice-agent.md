# OCR Invoice Agent — SOP

## Objective

Receive a photo or PDF of an invoice via Telegram, OCR it with Gemini Vision,
extract structured fields and line items, and persist them to Supabase. Reply
to the sender with a confirmation summary.

## Inputs

- Telegram message containing either:
  - A photo (`message.photo`)
  - A PDF document (`message.document` with `mime_type = application/pdf`)

Anything else is rejected with a polite reply.

## Topology

1. **Telegram Trigger** — receives the message.
2. **Switch - Input Type** — routes to `photo` / `pdf` / `other`.
3. **Set - Normalize** — flattens the message metadata into a stable shape so
   downstream nodes don't care which branch we came from.
4. **Telegram - Get File** — pulls the binary from Telegram's file API.
5. **Google Drive - Upload** — archives the original file under a fixed folder
   so we have an audit trail.
6. **Switch - By File Type** — splits image vs PDF processing.
7. **Image branch**: Gemini Vision analyzes the photo and emits structured
   JSON. The **AI Agent - Vision** code node validates and reshapes the
   response.
8. **PDF branch**: `Extract from PDF` pulls text, then a LangChain agent
   (Gemini text model) extracts the same structured shape via a
   `outputParserStructured` schema.
9. **Supabase - Insert Invoice** — writes the header row.
10. **Split Out - Line Items** + **Supabase - Insert Line Item** — fans out
    one row per line item.
11. **Format Reply** + **Telegram - Reply** — sends a human-readable summary
    back to the user.

## Required outputs

- One row in `invoices` per submission, with vendor / date / total / currency.
- N rows in `invoice_line_items` for the parsed line items.
- Telegram message back to the sender summarizing what was extracted.

## Edge cases

- **Non-invoice image**: Gemini still returns JSON but with empty fields. The
  Format Reply step surfaces "No invoice detected" rather than inserting an
  empty row.
- **PDF with no extractable text** (scanned image PDF): the text branch will
  return empty. Either pre-rasterize the PDF and route through the image
  branch, or rely on Gemini Vision's PDF support if available.
- **Telegram file > 20 MB**: Telegram's bot API blocks downloads above 20 MB.
  The agent will fail at `Telegram - Get File` — surface a friendlier error
  in Format Reply.

## Operational notes

- Drive uploads live under whatever folder you configure in the Google Drive
  node (default: Drive root). Move it somewhere sensible for production.
- Supabase tables `invoices` and `invoice_line_items` must exist before the
  workflow runs. See `docs/` or design your own schema with `vendor`, `date`,
  `total_amount`, `currency`, and `notes` columns at minimum.
