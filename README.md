# AI Invoice Processor (n8n + Gemini)

Reads invoices from an email inbox, extracts the data with Gemini, checks it with a validation layer, and saves only trustworthy results. Anything doubtful goes to a review queue with an instant Telegram alert.

**Demo video:** <link>

## The problem
Manually typing invoice data into a spreadsheet is slow and error-prone. Pure AI extraction is fast but can silently produce wrong numbers, so this pipeline never trusts the model on its own.

## How it works
1. **Gmail Trigger** watches for emails with PDF attachments (polls every minute).
2. **Gemini (Analyze document)** extracts supplier, bill-to, invoice number, date, currency, line items, subtotal, tax, total and a confidence score as strict JSON.
3. **Validator (Code node)** checks the output (see below) and sets `status` to `valid` or `needs_review`.
4. **Duplicate lookup** compares supplier + invoice number against saved invoices.
5. **IF node** routes the result:
   - `valid` -> Google Sheets `invoices` tab
   - `needs_review` -> Google Sheets `needs_review` tab + Telegram alert with the reasons
6. **Weekly summary** workflow sends totals per supplier every Monday at 08:00.
7. **Error workflow** sends a Telegram alert if any run fails.

## Validation checks
- AI call failed, blocked or returned empty/invalid JSON
- Missing or placeholder fields ("null", "N/A", "unknown")
- Non-numeric, zero or negative amounts
- Invalid, future or very old dates; day/month swap warning
- Bad or uncommon currency codes
- Quantity x unit price must equal each line amount
- Line items must sum to the subtotal; subtotal + tax must equal the total
- Supplier and bill-to swapped, or identical
- Low confidence score
- Duplicate invoice (same supplier + invoice number)

## Test cases
| File | Expected | Result |
|---|---|---|
| `invoice-clean.pdf` | valid, saved | valid, saved |
| `invoice-wrong-total.pdf` (total 1,050.00 vs 973.03) | flagged | needs_review + Telegram alert |
| `invoice-messy.pdf` (skewed, shadowed, blurred photo-style scan) | extracted or flagged | extracted correctly after prompt fix |
| Same invoice sent twice | flagged as duplicate | needs_review: duplicate invoice |

## Lessons learned
- Gemini reported 100% confidence on a degraded scan, so self-reported confidence is only a weak signal; arithmetic and party checks do the real protection.
- Early on it confused the supplier with the bill-to customer on a messy scan; fixed with explicit prompt rules plus a swapped-parties check.
- Ambiguous dates (06-07-2026) can't be verified by arithmetic; the prompt fixes the format and the validator adds a warning.
- Free-tier Gemini rate limits are handled with retry-on-fail and by continuing on error so failures reach the review queue instead of stopping the run.

## Setup
1. Import the workflow JSON files (credentials are not included).
2. Create credentials: Gmail OAuth, Google Sheets OAuth, Gemini API key, Telegram bot token.
3. Create a Google Sheet with tabs `invoices` and `needs_review` and the header row: `supplier, bill_to, invoice_number, invoice_date, currency, subtotal, tax, total, confidence, line_items, status, issues, warnings, email_from, email_subject, received_at, dedupe_key`.
4. Set `OWNER` and `MODE` at the top of the validator code.
5. Publish the workflows and set the error workflow in the main workflow's settings.

## Stack
n8n, Gemini API, Gmail, Google Sheets, Telegram Bot API, JavaScript.

## Known limits
- Scanned multi-page invoices and non-Latin scripts are untested.
- A confidently wrong read that still adds up arithmetically (for example a swapped date) needs human review.
- All sample invoices use fictional data.