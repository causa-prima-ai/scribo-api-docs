<img width="3840" height="1920" alt="API _ one call _ invoice (1)" src="https://github.com/user-attachments/assets/9117c89d-6996-4d1d-a257-5a7917d10a12" />

# Scribo API
> Free, AI-native e-invoicing — over plain HTTP.

## Contents

- [What is Scribo?](#what-is-scribo)
- [What is the Scribo API?](#what-is-the-scribo-api)
  - [Demo](#demo)
- [Getting started](#getting-started)
  - [Quickstart](#quickstart)
  - [Setup & prerequisites](#setup--prerequisites)
- [Usage](#usage)
  - [How it works](#how-it-works)
  - [Examples](#examples)
- [Compliance](#compliance)
- [FAQ](#faq)
- [Resources](#resources)
- [License](#license)

## What is Scribo?

Scribo is a free, conversational e-invoicing tool. Describe an invoice in plain language — "bill Acme GmbH €2,400 for May design work" — and it drafts a structured invoice for you to review, download and send directly.

Just ask your AI agent about Scribo: it ships as an MCP server, a CLI, and a Claude/Codex skill, alongside a public HTTP API and a web app. Free forever — no credit card. You just need a sender email.

**Built by Causa Prima** — Scribo is built by [**Causa Prima**](https://causaprima.ai/), a company building agentic AI for the CFO office. It's our first freely available skill — an early glimpse of what we're building.

## What is the Scribo API?

The Scribo HTTP API is the public contract every Scribo surface is built on — the skill, MCP server, CLI, and web app all call it. `POST` an invoice payload to `/api/v1/invoices` and get back a compliant invoice (a PDF with the matching EN 16931 XML embedded) plus a durable download URL.

It's anonymous and rate-limited — no signup, no API key required (optional bearer tokens unlock higher quotas for partners). The canonical OpenAPI 3.1 spec lives at [`/api/v1/openapi.json`](https://scribo.causaprima.ai/api/v1/openapi.json).

> This repo is the public landing page for the API. The canonical, always-current reference is at [scribo.causaprima.ai/docs/api](https://scribo.causaprima.ai/docs/api).

## Getting started

### Quickstart

Generate an invoice with a single `POST` — no key required. (First invoice for a sender email? Verify it first — see [the verification step](#setup--prerequisites) below.)

```sh
curl -sS https://scribo.causaprima.ai/api/v1/invoices \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "X-Email-Verification-Token: $TOKEN" \
  -d '{
    "sender": {
      "legal_name": "Example GmbH", "country_code": "DE",
      "address_line1": "Example Allee 1", "postcode": "10115", "city": "Berlin",
      "tax_id": "DE123456788", "contact_email": "billing@example.com"
    },
    "recipient": {
      "legal_name": "Acme GmbH", "country_code": "DE",
      "address_line1": "Hauptstrasse 1", "postcode": "10117", "city": "Berlin",
      "tax_id": "DE987654321", "contact_email": "ap@acme.example"
    },
    "line_items": [
      { "description": "Senior consulting, 3 days", "quantity": "3",
        "unit_code": "DAY", "unit_price": "1200.00", "tax_rate": "19",
        "tax_category_code": "S" }
    ],
    "currency": "EUR",
    "due_date": "2026-06-01"
  }'
```

Returns `201` with an `InvoiceRecord`, including a durable `download_url`:

```json
{
  "invoice_id": "f0b3c1e2-...",
  "format": "zugferd_comfort",
  "download_url": "https://scribo.causaprima.ai/i/f0b3c1e2-.../download",
  "validator_summary": { "valid": true, "validator": "invopop", "errors": [] },
  "magic_link_sent": true
}
```

Fetch the file from `download_url` — it's durable and re-downloadable any time.

### Setup & prerequisites

- **Base URL (production):** `https://scribo.causaprima.ai`
- **Auth:** none — the `/api/v1/*` API is anonymous. Send `Authorization: Bearer <key>` only if you have a partner key for elevated quotas.
- **Verification (first invoice per sender):** the first invoice for each sender email must prove address ownership, or `POST /invoices` returns `401 email_verification_required`. Request a challenge (`POST /api/v1/scribo/email-verifications` with `{ "email": … }` — Scribo emails a 6-character code), redeem it (`POST …/{id}/redeem` with `{ "code": … }` → `{ verification_token }`), then send the token as `X-Email-Verification-Token`. One token covers ~30 minutes of invoices for that sender.
- **Format:** JSON over HTTPS. Numeric fields (`quantity`, `unit_price`, `tax_rate`) are **strings** holding decimals — the API rejects floats, to preserve precision.
- **Idempotency:** send an `Idempotency-Key` header — same key + same payload returns the original invoice, so retries are safe.
- **Spec:** the canonical OpenAPI 3.1 document is at [`/api/v1/openapi.json`](https://scribo.causaprima.ai/api/v1/openapi.json) — generate typed clients with `openapi-typescript`, `oapi-codegen`, or any OpenAPI 3.1 tool.

## Usage

### How it works

Eight endpoints under `/api/v1/*`:

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/invoices` | Generate a compliant invoice; returns an `InvoiceRecord` with a durable `download_url`. |
| `GET` | `/api/v1/invoices/{id}/download` | Stream the invoice file bytes. |
| `GET` | `/api/v1/jurisdictions` | List supported jurisdictions and their formats. |
| `GET` | `/api/v1/openapi.json` | The canonical OpenAPI 3.1 document. |
| `POST` | `/api/v1/scribo/email-verifications` | Request a sender email-ownership challenge (uniform `202`, anti-enumeration). |
| `POST` | `/api/v1/scribo/email-verifications/{id}/redeem` | Exchange the 6-character code for a `verification_token`. |
| `GET` | `/api/v1/scribo/email-verifications/{id}` | Challenge status snapshot (polling fallback). |
| `GET` | `/api/v1/scribo/email-verifications/{id}/events` | SSE stream of challenge events. |

The output format is chosen by a deterministic priority chain (e.g. a `recipient.leitweg_id` selects German XRechnung); see [jurisdictions](https://scribo.causaprima.ai/docs/jurisdictions). To re-download, use the `download_url` from the create response as-is — it's durable. (A metadata re-fetch at `GET /api/v1/invoices/{id}` is deferred in Phase 1 and currently returns `404`.)

**Errors** share one envelope:

```json
{ "error": { "code": "rate_limited", "message": "Rate limit exceeded", "details": {} } }
```

**Rate limits** (per source IP / sender tenant): 600 req/min and 200 invoices/hour per IP; 5 invoices/min and 50 per 24h per sender. `429` responses carry `retry_after_seconds`, `reset_at`, and `limit_code`. Full error reference: [troubleshooting](https://scribo.causaprima.ai/docs/troubleshooting).

### Examples

**Check supported jurisdictions**

```sh
curl -sS https://scribo.causaprima.ai/api/v1/jurisdictions | jq
```

**German B2G XRechnung** — add a `leitweg_id` to the recipient (auto-selects XRechnung) and a `payment_means` block (required by XRechnung BR-DE-1). The rest of the payload matches the [Quickstart](#quickstart):

```json
"recipient": {
  "legal_name": "Bundesamt für Beispiele", "country_code": "DE",
  "address_line1": "Wilhelmstr. 1", "postcode": "10117", "city": "Berlin",
  "leitweg_id": "991-12345-67"
},
"payment_means": {
  "type": "credit_transfer", "iban": "DE89370400440532013000", "account_name": "Acme GmbH"
}
```

`payment_means` also accepts US bank details — `account_number` + `routing_number` (plus optional `bic` / `bank`) instead of `iban` — rendered on US plain-PDF invoices. XRechnung specifically requires the IBAN.

## Compliance

Scribo emits invoices conforming to **EN 16931**, the European e-invoicing standard, with the relevant national CIUS. Every EN 16931 output (ZUGFeRD, XRechnung) is validated against the **Invopop**-hosted validator at generate-time — output that fails the rule set never reaches the user. US plain PDFs carry no EN 16931 XML and are rendered directly.

**Supported formats**

| Jurisdiction | Format | Status |
|---|---|---|
| Germany (B2B) | **ZUGFeRD** COMFORT (PDF/A-3 hybrid + CII XML) | ✅ Live |
| Germany (B2G) | **XRechnung** (UBL / CII) | ✅ Live |
| United States | Plain PDF (no XML, no e-invoice claim) | ✅ Live |
| France | **Factur-X** EN 16931 | 🔜 Coming soon |
| Spain | **Facturae** | 🔜 Coming soon |
| Belgium / NL / LU / AT | **Peppol BIS 3.0** UBL | 🔜 Coming soon |

*Disclaimer: Scribo generates and validates compliant invoice documents. It is **not tax or legal advice** — Scribo does not determine your tax obligations, VAT treatment, or filing requirements.*

## FAQ

<details>
<summary><strong>Is Scribo really free?</strong></summary>

Yes — free forever. No credit card, no subscription, no paywall before your first invoice. You just need a sender email.

</details>

<details>
<summary><strong>Do I need an account or signup?</strong></summary>

No signup form. On your first invoice, Scribo verifies the sender email — a 6-digit code (or one-click link) arrives at that address; one verification covers ~30 minutes of invoicing. The same email doubles as your magic-link login for re-downloads later.

</details>

<details>
<summary><strong>Which countries and formats are supported?</strong></summary>

Live today: **Germany** — ZUGFeRD (B2B) and XRechnung (B2G) — and the **United States** (plain PDF). Coming next: **France** (Factur-X), **Spain** (Facturae), and **Belgium / NL / LU / AT** (Peppol BIS 3.0).

</details>

<details>
<summary><strong>What does "EN 16931-compliant" actually mean here?</strong></summary>

Every EN 16931 output (ZUGFeRD, XRechnung) is validated against the **EN 16931-1:2017** rule set (via the Invopop-hosted validator) before it's returned. Output that fails validation never reaches you. US plain PDFs carry no EN 16931 XML, so no schematron validation applies.

</details>

<details>
<summary><strong>Is the US version compliant?</strong></summary>

Yes. There is no US e-invoicing mandate, so Scribo produces a fully compliant plain PDF.

</details>

<details>
<summary><strong>Does Scribo give tax or legal advice?</strong></summary>

No. Scribo generates and validates compliant invoice *documents*. It does not determine your tax obligations, VAT treatment, or filing requirements.

</details>

<details>
<summary><strong>How do AI agents / LLMs use Scribo?</strong></summary>

Scribo is built to be operated by an agent. It ships as an **MCP server**, a **CLI**, and a **Claude/Codex skill**, plus a public **HTTP API** and a **web app** — all on the same backend. An agent can discover Scribo, create an invoice, and return the file on a user's behalf.

</details>

<details>
<summary><strong>Who builds Scribo?</strong></summary>

[Causa Prima](https://causaprima.ai/) — a company building agentic AI for the CFO office. Operated by Causa Prima Germany GmbH, Munich.

</details>

## Resources

- **Web app** — [scribo.causaprima.ai](https://scribo.causaprima.ai)
- **Documentation** — [scribo.causaprima.ai/docs](https://scribo.causaprima.ai/docs)
- **Compliance & trust** — [scribo.causaprima.ai/compliance](https://scribo.causaprima.ai/compliance)
- **Causa Prima** — [causaprima.ai](https://causaprima.ai)

**Other Scribo surfaces**

- Skill (Claude / Codex) — [`scribo-skill`](https://github.com/causa-prima-ai/scribo-skill) · [docs](https://scribo.causaprima.ai/docs/skill)
- MCP server — [`scribo-mcp`](https://github.com/causa-prima-ai/scribo-mcp) · [docs](https://scribo.causaprima.ai/docs/mcp)
- CLI — [`scribo-cli`](https://github.com/causa-prima-ai/scribo-cli) · [docs](https://scribo.causaprima.ai/docs/cli)
- HTTP API — [`scribo-api-docs`](https://github.com/causa-prima-ai/scribo-api-docs) · [docs](https://scribo.causaprima.ai/docs/api)
- Brand hub — [`scribo`](https://github.com/causa-prima-ai/scribo)

## License

Proprietary — `UNLICENSED`. © Causa Prima Germany GmbH. All rights reserved. Distributed for use against the public Scribo API; not open-source. See [LICENSE](./LICENSE).

