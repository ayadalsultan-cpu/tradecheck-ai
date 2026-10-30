# TradeCheck AI

An n8n agent workflow that checks one shipment's trade documents (commercial invoice, packing list, certificate of origin) before customs declaration, and drafts HS codes for a licensed customs broker.

**TradeCheck AI never submits declarations.** Every output is a draft for a human broker.

## What it does

1. **Upload**: an n8n form takes the three PDFs and a shipment ID.
2. **Extract**: Google Gemini extracts structured data from each document.
3. **Guard**: a two-layer prompt-injection check (an LLM guardrail plus rule-based patterns).
4. **Check consistency**: deterministic checks on quantities, line totals, invoice total, net/gross weights, origin and line count.
5. **Classify**: an AI agent suggests 6-digit HS codes, grounded in a tariff vector store. It returns "NO CODE SUGGESTED" when unsure.
6. **Apply rules**: tariff grounding, a confidence threshold and a restricted-goods list.
7. **Route**: `ready_for_broker` or `human_review`.
8. **Report**: a Google Doc and an email to the broker, plus a row in a run log (Google Sheet).

A separate **error workflow** emails an alert when a production run fails.

## Repository layout

| Path | Content |
|---|---|
| `workflows/tradecheck_shipment_check.json` | Main workflow (v1.5) |
| `workflows/tradecheck_error_alert.json` | Error alert workflow |
| `eval/answer_key.csv` | Expected route, issues and HS codes per test |
| `eval/documents/` | Synthetic test documents (TEST-03 to TEST-22, TEST-27). No real transactions. |
| `docs/TradeCheck_AI_Act_GDPR_assessment.pdf` | EU AI Act and GDPR assessment |

## Setup

1. In n8n: **Workflows → Import from file**. Import both JSON files.
2. Create these credentials and select them in the nodes:
   - Google Gemini (PaLM) API: chat models and embeddings
   - Supabase: `tariff_search` vector store (table with tariff lines)
   - Google Sheets, Google Docs, Gmail (OAuth2)
3. Replace the placeholders:
   - `YOUR_GOOGLE_SHEET_ID`: the reference sheet with tabs `tariff` (`hs6`, `duty_rate_percent`, …), `restricted` (`hs6`, `reason`) and `run_log` (`timestamp`, `shipment_id`, `route`, `issues_count`, `reasons`)
   - `YOUR_DRIVE_FOLDER_ID`: the Drive folder for reports
   - `YOUR_EMAIL@example.com`: the broker / alert recipient
4. In the main workflow: **Settings → Error workflow → TradeCheck Error Alert**.
5. Activate both workflows. The error workflow only fires for production runs, not manual test runs.

## Evaluation

Run each test set through the form and compare the run log with `eval/answer_key.csv`.
- TEST-03 to TEST-10 should be `ready_for_broker`.
- TEST-11 to TEST-22 and TEST-27 (prompt injection in the certificate) should be `human_review`.

## Compliance note

Under the EU AI Act this is a non-high-risk AI system with Art. 4 (AI literacy) and Art. 50 (transparency) touchpoints. See `docs/` for the full assessment, including GDPR. That assessment is an analysis, not legal advice.
