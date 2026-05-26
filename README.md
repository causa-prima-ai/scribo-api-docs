# Scribo HTTP API — documentation

Public REST API for [Scribo](https://github.com/causa-prima-ai/scribo) — free EN 16931-compliant e-invoicing (XRechnung, ZUGFeRD, Factur-X, Peppol BIS UBL, Spanish Facturae, plain US PDF) for any LLM, script, or app.

> 🚧 **Status: docs in progress.** The canonical, always-current reference lives at [`scribo.causaprima.ai/docs/api`](https://scribo.causaprima.ai/docs/api). This repo will mirror it as versioned Markdown for offline use, LLM indexing, and contributions.

**Base URL:** `https://scribo.causaprima.ai`

## Endpoints (v1)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/v1/invoices` | Generate an invoice. Returns invoice metadata + 15-minute signed download URL. Requires `Idempotency-Key` header. |
| `GET` | `/api/v1/invoices/:id` | Fetch a generated invoice and a fresh signed download URL. |
| `GET` | `/api/v1/invoices/:id/download` | Stream the rendered PDF (or PDF/A-3 hybrid). |
| `GET` | `/api/v1/jurisdictions` | List supported jurisdictions and their default formats. |

JSON Schemas for each request body live in [`scribo-mcp/schemas/`](https://github.com/causa-prima-ai/scribo-mcp/tree/main/schemas). Sample payloads: [`scribo-mcp/examples/`](https://github.com/causa-prima-ai/scribo-mcp/tree/main/examples).

## Quick start

```sh
curl -X POST https://scribo.causaprima.ai/api/v1/invoices \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d @invoice.json
```

A minimal `invoice.json` example will land in [`examples/`](./examples) as this repo fills out.

## Account model

No signup. The `sender.contact_email` on your first `POST /api/v1/invoices` becomes your account. Scribo emails a magic link to that address — clicking it binds a session at [`scribo.causaprima.ai`](https://scribo.causaprima.ai) so you can return, re-download, or upgrade.

## Format coverage

| Jurisdiction | Default format | Notes |
|---|---|---|
| Germany (B2B) | ZUGFeRD COMFORT | PDF/A-3 hybrid + EN 16931 CII XML |
| Germany (B2G) | XRechnung CII | Triggered by `recipient.leitweg_id` |
| France | Factur-X | PDF/A-3 hybrid + EN 16931 CII XML |
| Spain | Facturae 3.2.2 | Spanish-specific XML syntax |
| Belgium / NL / LU / AT | Peppol BIS 3.0 UBL | Network transmission requires a Peppol AP partner |
| United States | Plain PDF | No XML |

Override the default per-invoice with `format_override`.

## Authentication

v1 is unauthenticated for end-users — the sender email is the implicit identity, and rate limits + Turnstile protect the endpoint. A forward-compatible `SCRIBO_API_KEY` header is reserved for partner integrations.

## Errors & idempotency

- All `POST`s require an `Idempotency-Key` header — repeating the same key returns the same invoice instead of generating a duplicate.
- Validator errors return `422` with the failing rule. Common ones: [`scribo-skill/references/troubleshooting.md`](https://github.com/causa-prima-ai/scribo-skill/blob/main/references/troubleshooting.md).
- Rate limits return `429` with `Retry-After`.

## Compliance

- EN 16931 conformance verified at generate-time by [Invopop](https://invopop.com)'s hosted validator.
- CI validators: [Mustang](https://github.com/ZUGFeRD/mustangproject), [KoSIT](https://www.kosit.org/) (XRechnung), [veraPDF](https://verapdf.org/) (PDF/A-3).
- Sub-processor list: [`/compliance`](https://scribo.causaprima.ai/compliance).

## Other ways to use Scribo

- **MCP server** (Claude Desktop, Cursor, Cline, ChatGPT App) → [`scribo-mcp`](https://github.com/causa-prima-ai/scribo-mcp)
- **Skill** (Claude Code, Codex CLI) → [`scribo-skill`](https://github.com/causa-prima-ai/scribo-skill)
- **CLI** (npm, planned) → [`scribo-cli`](https://github.com/causa-prima-ai/scribo-cli)
- **Web app** → [scribo.causaprima.ai](https://scribo.causaprima.ai)
- **Umbrella** → [`scribo`](https://github.com/causa-prima-ai/scribo)

## License

Docs MIT. The hosted service is operated by [Causa Prima](https://causaprima.ai).
