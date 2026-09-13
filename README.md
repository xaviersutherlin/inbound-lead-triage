# Inbound Lead Triage

An n8n workflow that takes a web form enquiry, cleans and validates it, uses
Claude to classify how urgent it is and what kind of job it is, writes it to a
Google Sheet, and emails an alert for anything urgent.

## What it does

Form submission
  -> validate and normalise the input
  -> Claude classifies service type and urgency, and writes a one-line summary
  -> row created or updated in Google Sheets
  -> urgent leads trigger an email alert
  -> any failure triggers a separate alert to the operator

## Demo

- 90-second walkthrough: https://drive.google.com/file/d/1UdYW-OYmvOS8_ZFLUx6IxmGxspdPIFx7/view?usp=sharing
- Screenshots: see /screenshots




## How it handles failure

The happy path is the easy part. This is what happens when things go wrong.

**Bad or missing input.** Every submission is checked for a usable name,
email, phone and message. Anything incomplete is written to the sheet with
status `needs_review` rather than dropped, or silently written as though it
were fine. A lost enquiry is a lost customer; a flagged one is a two-minute job.

**Messy input.** Phone numbers are stripped to digits and a leading US country
code is removed, so `(970) 555-0142` and `1-970-555-0142` both store as
`9705550142`. Emails are lowercased. Whitespace is collapsed.

**The model returning something unexpected.** The classification step asks for
JSON. Models occasionally wrap it in prose or in code fences, and a naive
`JSON.parse()` would throw and kill the run. The parser strips fences, extracts
the object, and if it still cannot parse, keeps the lead, flags it
`needs_review`, and records what the model actually said.

**The model returning an invalid value.** `service` and `urgency` are both
checked against their allowed lists. Anything outside the list falls back to a
safe default rather than reaching the sheet.

**Transient API failures.** The classification node retries 3 times with a
3-second wait. On Error is deliberately left as Stop Workflow, because enabling
a Continue option silently disables retries in n8n.

**Anything else.** A separate error workflow catches failures and emails the
workflow name, the node that failed, the error message, and a direct link to
the failed run.

**Duplicate submissions.** Rows are matched on email, so the same person
submitting twice updates their row instead of creating a second one.

## Setup

1. Import `workflow.json` into n8n.
2. Connect three credentials:
   - Google Sheets (OAuth2)
   - Gmail (OAuth2), for the alerts
   - An Anthropic chat model - gateway credits on n8n Cloud, or your own API key
3. Create a Google Sheet whose first row is exactly these nine headers, lowercase:
   `timestamp` `name` `email` `phone` `message` `service` `urgency` `summary` `status`
4. Point the Google Sheets node at that document and sheet. Operation is
   Append or Update Row, matching on `email`.
5. Import `error-workflow.json` as a second workflow. In the main workflow,
   open Settings and set Error workflow to it.
6. Change the To address on both Gmail nodes to wherever alerts should go.

## Known limits

Things this does not do, and what I would change for real volume.

- **Repeat enquiries overwrite.** Matching on email means a second submission
  replaces the first. For a lead sheet, current state beats history - but if a
  client needed the full trail, this should append and mark superseded rows
  instead.
- **No spam protection.** The form is a public URL with no rate limiting or
  captcha. Fine for a low-traffic site, not for anything that gets found.
- **US phone numbers only.** Normalisation assumes a 10-digit number.
  International numbers are flagged as invalid.
- **No retry on the Sheets node.** Only the AI step retries. A Google outage
  mid-run fails the execution rather than queuing it.
- **Alerts go to one hardcoded address.** Real use would route by service type
  or by an on-call rota.
- **Classification quality depends on the prompt.** The category list is fixed
  and tuned for a general service business. A specific industry would need its
  own categories and examples.
- **Single sheet, no archiving.** Comfortable in the hundreds of rows. At tens
  of thousands this should be a database.

## Built by

Xavier Sutherlin - Xaviersutherlin2027@gmail.com
Built as a portfolio piece. Happy to walk through any part of it.
