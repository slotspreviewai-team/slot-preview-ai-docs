# Authentication and environments

Canonical page: https://slotspreviewai.com/docs/authentication

Source: https://slotspreviewai.com/docs/authentication

### Authentication and environments

Every request carries an API key in the `Authorization` header as a bearer token. There is no other credential: no OAuth, no session, no signature on requests.

```sh
export SLOTPREVIEW_BASE_URL=https://api.slotspreviewai.com
export SLOTPREVIEW_API_KEY=spai_test_…
```

#### Key format

A key looks like `spai_test_` or `spai_live_` followed by a public prefix, an underscore and the secret. The environment is part of the key itself, so a sandbox key can never be mistaken for a production one in a log or a config file:

- `spai_test_`: the sandbox. Free, never calls the image model, returns built-in fixtures. Use it in CI and while integrating.
- `spai_live_`: production. Real generation, tokens are charged.

The secret is shown once, when the key is created, and is not stored by the server: if you lose it, create a new key. Keys are created, rotated and revoked in the API keys tab of your account, not through the API. The account issues sandbox keys by default; a production key is an explicit choice there.

#### Check a key

`GET /v1/me` is the cheapest call in the API and the first one to make with a new key. It tells you which organisation the key belongs to, which environment it is in and what it may do:

```
curl -s "$SLOTPREVIEW_BASE_URL/v1/me" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY"

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

`token_balance` is in whole tokens, like every amount in the public API. A key that can spend also shows its daily cap: `daily_token_cap`, `daily_tokens_used` today and `daily_cap_resets_at`, the next midnight UTC. `environment` is the same value the `X-Environment` header carries once the key is accepted: `test` on a sandbox key, `live` on a production key. From that point the header is on every answer of the call, error responses from the handler included: the recorded 400, 402, 404, 409 and 429 on [Errors and limits](https://slotspreviewai.com/docs/errors) all carry it. A refused key gets no header, because no environment was established: the 401 and the 403 recorded there and below have none. If you ever wonder why two runs produced the same image, look at that header first.

#### Scopes

Scopes limit what a key may do. A new key gets `read` and `render`; the spending scopes are switched on explicitly when the key is created, because each of them can take tokens from the balance, and a key with a spending scope also gets a [daily token cap](https://slotspreviewai.com/docs/errors#403-daily-token-cap-exceeded).

| Scope | What it opens |
| --- | --- |
| `read` | The catalogue, the reference lists, jobs, sets, deliveries, balance and `GET /v1/me`. |
| `render` | Issued together with `read`. `POST /v1/renders`: drawing a variant with its caption for a preview, free on any key. |
| `generate` | `POST /v1/masters` and `POST /v1/masters/estimate`: generation and its price. Spends tokens on a production key. |
| `export` | `POST /v1/masters/{id}/export`: rendering the format set. Spends tokens on a production key. |
| `batch` | `POST /v1/batches` and the other batch operations: many titles in one job. Spends tokens per item on a production key. |

A key without the scope an operation needs gets `403 forbidden_scope`, and `details.required_scope` names the missing one. This is a key with only `read` trying to generate:

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "game": {
    "id": "00000000-0000-4000-8000-000000000000"
  },
  "art": {
    "style": "dark-premium",
    "background": "rays-vignette",
    "composition": "hero_upper_60_lower_third_clear"
  }
}
EOF

HTTP 403

{
  "error": {
    "code": "forbidden_scope",
    "message": "The credential lacks the scope this call needs. details.required_scope names it.",
    "details": {
      "required_scope": "generate"
    }
  }
}
```

Add the scope to the key or create a key that has it; the request itself was fine.

#### The sandbox

A sandbox key runs the same code paths as production with two differences: the image model is never called, and nothing is charged. `POST /v1/masters` returns one of the built-in fixture images, chosen deterministically from the art parameters, so the same request gives the same picture every time. Export renders the real format set from that fixture. Every paid operation answers with `X-Tokens-Charged: 0` and adds `X-Tokens-Would-Charge`, the amount a production key would have paid for the same call.

The sandbox is meant for wiring: polling, webhooks, idempotency, error handling and the export flow all behave as in production. What it cannot show you is the quality of the art for your game; that needs a production key.

#### Test scenarios

A sandbox key can be asked to fail on purpose with the `X-Test-Scenario` header on `POST /v1/masters`. A production key ignores the header, so leaving it in by mistake breaks nothing.

| Scenario | What happens |
| --- | --- |
| `insufficient_tokens` | Immediate `402 insufficient_tokens`, no job is created. |
| `rate_limited` | Immediate `429 rate_limited` with `Retry-After`, no job is created. |
| `provider_error` | The job fails with `provider_unavailable` after the usual sandbox delay. |
| `provider_error_then_fallback` | The primary model fails, the fallback delivers; the job succeeds and is marked `degraded`. |
| `qa_reject` | One variant is rejected by quality control; the job ends `partial` with the other one. |
| `qa_reject_all` | Both variants are rejected; the job fails with `quality_budget_exhausted`. |
| `content_refused` | The model declines the brief; the job fails with `content_refused`, not retryable. |
| `slow` | The job takes about a minute instead of a few seconds, for testing timeouts. |
| `webhook_flood` | The job completes and the same event is delivered three times with one `event_id`, for testing that your receiver deduplicates. |

The first two answer before a job exists. The others create a job that reaches the named outcome, so you can test the same polling and webhook code you run in production. Examples of the immediate ones with recorded responses are on [Errors and limits](https://slotspreviewai.com/docs/errors).
