# Overview

Canonical page: https://slotspreviewai.com/docs

Source: https://slotspreviewai.com/docs

### Lobby tiles for your slots, generated before the game ships

One REST API and one flow: register a title, generate a master image, export the whole set of lobby formats from it. The files carry the art with a caption of the game title and studio name, a title of your own, or no caption at all, and you can preview the caption on the master for free before you export. Every example in these guides is recorded from the live API with a sandbox key and re-checked against it before every release.

#### Your first call

Export the address and a sandbox key, then ask the API who you are:

```sh
export SLOTPREVIEW_BASE_URL=https://api.slotspreviewai.com
export SLOTPREVIEW_API_KEY=spai_test_…
```

```
curl -s "$SLOTPREVIEW_BASE_URL/v1/me" \
  -H "Authorization: Bearer spai_test_…"

HTTP 200
x-environment: test
x-tokens-charged: 0

{
  "organization": {
    "id": "00000000-0000-0000-0000-0000000000f0",
    "name": "Acme Gaming",
    "plan": "studio",
    "token_balance": 42
  },
  "environment": "test",
  "api_key_id": "3af1a1f9-71ff-4268-9d1f-1eebc8a3fb5f",
  "scopes": [
    "read",
    "render",
    "generate",
    "export",
    "batch"
  ],
  "daily_token_cap": 25,
  "daily_tokens_used": 0,
  "daily_cap_resets_at": "2026-10-13T00:00:00.000Z"
}
```

#### Get a sandbox key

[Create an account](https://slotspreviewai.com/signup) or sign in, open the API keys tab and create a key. Sandbox is the default environment there: the key is free, never calls the image model and returns built-in fixtures, so you can wire the whole flow before spending a token. Production keys are created in the same place.

#### What next

- [Quickstart](https://slotspreviewai.com/docs/quickstart): five steps from a game name to an image address you can open in a browser.
- [Authentication and environments](https://slotspreviewai.com/docs/authentication): key format, scopes, sandbox against production, test scenarios.
- [Errors and limits](https://slotspreviewai.com/docs/errors): the envelope, every code with a recorded example, the export reason codes.
- [MCP server](https://slotspreviewai.com/docs/mcp): connect an AI agent such as Claude Code, Cursor or your own agent on the OpenAI or Claude API, and let it make tiles with your key.
- [API reference](https://slotspreviewai.com/docs/api): every operation with schemas, the OpenAPI document and a Postman collection to import.

#### Three things to know

1. **Two calls cost tokens.** Generating a master costs 1 and exporting the format set costs 2, in tokens, as the API itself reports in `X-Tokens-Would-Charge`. Everything else, from the catalogue to previews, polling and downloads, is free, and so is changing the caption of a set you already exported. Send `max_tokens` with both calls and neither will ever cost more than you agreed. Token packs are listed on the [pricing section](https://slotspreviewai.com/#pricing) of the site.
2. **Generation is asynchronous.** `POST /v1/masters` answers `202 Accepted` with a job and a `polling_url`; the image comes later. Poll the job or subscribe a webhook endpoint and let the API call you.
3. **The sandbox is isolated on every paid path.** A `spai_test_` key returns fixtures, charges nothing and reports what production would have charged. Polling, webhooks, idempotency and price limits behave as on a production key, so the integration you test is the one you ship. A short balance is played by a test scenario, and a sandbox key never reaches a daily cap.
