---
name: papra
description: Read documents (invoices, letters, contracts) from the self-hosted Papra archive on this server, including their extracted text.
required_environment_variables:
  - name: PAPRA_API_KEY
    prompt: Papra API key (documents:read, organizations:read)
    help: Papra → Settings → API keys
    required_for: reading documents
---

# Papra (self-hosted document archive)

Papra runs on this server at `http://127.0.0.1:1221`. Authenticate every request with
`-H "Authorization: Bearer $PAPRA_API_KEY"`. The key is read-only; never try to change documents.

- Organizations: `GET /api/organizations` → `organizations[].id`
- Newest documents: `GET /api/organizations/<orgId>/documents?pageSize=10`
  (sorted by `createdAt` desc; supports `searchQuery=`)
- One document incl. extracted text: `GET /api/organizations/<orgId>/documents/<docId>` →
  `document.name`, `document.content`, `document.createdAt`

Use plain `curl -s` and read the JSON output yourself; the responses are small. Do not pipe curl into
`python3`, `sh` or another interpreter: Hermes flags that as a dangerous command.
Quote amounts, dates and invoice numbers exactly as they appear in `content`.
