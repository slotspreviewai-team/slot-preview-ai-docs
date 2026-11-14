# Errors and limits

Canonical page: https://slotspreviewai.com/docs/errors

Source: https://slotspreviewai.com/docs/errors

### Errors and limits

Every error is JSON with one top-level key, `error`. The contract is `code` and `details`: branch on them. `message` is a human-readable hint for your logs, not a stable string and not always in English. `request_id` is present on server-side failures and ties the response to a line in our journal; quote it when you contact support.

```json
{
  "error": {
    "code": "validation_failed",
    "message": "(human-readable, not part of the contract)",
    "request_id": "c03f11d4-8f31-440a-a543-a12db987d274",
    "details": { "issues": [] }
  }
}
```

The examples below are recorded from the live API with a sandbox key. Where a code cannot be triggered from the sandbox, the table links it to the server code instead of an example.

```sh
export SLOTPREVIEW_BASE_URL=https://api.slotspreviewai.com
export SLOTPREVIEW_API_KEY=spai_test_…
```

#### 400 validation_failed

The body or the query string does not match the schema. `details.issues[]` names the path and the reason. Schemas are strict at the top level: an unknown key is a 400 with `unrecognized_keys`, not a silently ignored field, so a typo in a field name fails here rather than producing a wrong result later. The request never reaches the handler: nothing is held, no job is created.

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{}
EOF

HTTP 400
x-environment: test
x-tokens-charged: 0

{
  "error": {
    "code": "validation_failed",
    "message": "(human-readable, not part of the contract)",
    "details": {
      "issues": [
        {
          "code": "invalid_union",
          "unionErrors": [
            {
              "issues": [
                {
                  "code": "invalid_type",
                  "expected": "object",
                  "received": "undefined",
                  "path": [
                    "game"
                  ],
                  "message": "Required"
                }
              ],
              "name": "ZodError"
            },
            {
              "issues": [
                {
                  "code": "invalid_type",
                  "expected": "object",
                  "received": "undefined",
                  "path": [
                    "game"
                  ],
                  "message": "Required"
                }
              ],
              "name": "ZodError"
            }
          ],
          "path": [
            "game"
          ],
          "message": "Invalid input"
        },
        {
          "code": "invalid_type",
          "expected": "object",
          "received": "undefined",
          "path": [
            "art"
          ],
          "message": "Required"
        }
      ]
    }
  }
}
```

The caption field of an export is strict in the same way. `overlay_config_id` of `POST /v1/masters/{id}/export` and of its estimate takes an id returned by `POST /v1/renders` or the word `"none"`; `null`, an empty string or any other text is refused here, and the refusal names `"none"`. An id that does not exist or belongs to another organisation is not a 400 but the same 404 as an unknown master.

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters/00000000-0000-4000-8000-000000000000/export" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "overlay_config_id": null
}
EOF

HTTP 400
x-environment: test
x-tokens-charged: 0

{
  "error": {
    "code": "validation_failed",
    "message": "(human-readable, not part of the contract)",
    "details": {
      "issues": [
        {
          "code": "invalid_type",
          "expected": "string",
          "received": "null",
          "path": [
            "overlay_config_id"
          ],
          "message": "overlay_config_id must be an overlay_config_id returned by POST /v1/renders, or \"none\" for art without a caption; omit the field to keep the caption of an already built set (a new set gets the default caption)"
        }
      ]
    }
  }
}
```

#### 401 unauthenticated

No `Authorization` header, a malformed key, a revoked or expired key, or a key of the other environment. `details.reason`, when present, says which. The fix is always on your side: check the key, or create a new one in the account.

```
curl -s "$SLOTPREVIEW_BASE_URL/v1/me"

HTTP 401

{
  "error": {
    "code": "unauthenticated",
    "message": "Authentication failed. Check the API key or sign in again; details.reason, when present, says why."
  }
}
```

#### 403 forbidden_scope

The key is valid but lacks the scope the operation needs; `details.required_scope` names it. Create a key with that scope or switch it on for this one, see [Scopes](https://slotspreviewai.com/docs/authentication#scopes).

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

#### 403 daily_token_cap_exceeded

A key can carry a daily cap in tokens, set when the key is created; a key that spends cannot be created without one. An agent connected with sign-in over MCP carries its own cap, set on the sign-in page, and is refused the same way; `details.credential` says whose cap it is, `api_key` or `oauth_connection`. The cap is checked only by calls that spend: generation, export and batches. Reading, polling, estimates, downloads, cancelling and free repeats never stop at it. A call that would take the key over its cap is refused before anything is held, so the key never spends more than the cap, also when several calls arrive at the same moment. `details` carries `cap`, `used` and `required` in tokens and `resets_at`, the next midnight UTC; `GET /v1/me` with the same key shows `daily_token_cap`, `daily_tokens_used` and `daily_cap_resets_at` before you hit it. Tokens returned by a cancelled or failed job count back into the day they were held on. A person raises the cap in the account; topping up does not.

#### 403 spending_frozen

Spending is frozen for the whole organisation: a payment was reversed, or a top-up did not reconcile. The refusal comes before anything is held and has no `top_up_url`: topping up does not lift a freeze. Do not retry; write to support.

#### 404 not_found

An identifier that does not exist or belongs to another organisation. The two cases answer identically on purpose: the API does not confirm that a foreign resource exists. This is a generation request for a game id the API does not know:

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

HTTP 404
x-environment: test
x-tokens-charged: 0

{
  "error": {
    "code": "not_found",
    "message": "(human-readable, not part of the contract)"
  }
}
```

Lookups by identifier answer the same way: `GET /v1/games/{id}` and `GET /v1/masters/jobs/{id}` with an unknown id answer 404 with the code `not_found`.

#### 409 idempotency_key_reuse

Both calls that spend, `POST /v1/masters` and `POST /v1/masters/{id}/export`, accept an `Idempotency-Key`. Repeating the same key with the same body returns the original response and charges nothing, even if the job has since failed or been cancelled and its tokens returned. Repeating it with a different body, or for another master, is refused: the key has already been spent on another request. Pick one key per logical request and keep it for retries only.

```
IDEM_KEY=$(uuidgen)
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Idempotency-Key: $IDEM_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "game": {
    "id": "$GAME_ID"
  },
  "art": {
    "style": "dark-premium",
    "background": "rays-vignette",
    "composition": "hero_upper_60_lower_third_clear"
  }
}
EOF

HTTP 202
x-environment: test
x-tokens-charged: 0
x-tokens-would-charge: 1

{
  "id": "1b36df48-ab6d-4997-9725-19fb76d33189",
  "object": "generation_job",
  "kind": "generate",
  "status": "queued",
  "tokens_held": 0,
  "tokens_charged": 0,
  "engine": "pro",
  "anchor": "none",
  "reference": "none",
  "masters": [
    {
      "id": "a3628e10-fc74-4f97-8f8e-a578a3c89126",
      "status": "pending",
      "variant_index": 0,
      "format": "tile-2x3-2x",
      "variant_seed": 1183121174,
      "engine": "pro"
    },
    {
      "id": "d95cb8d8-8b3b-4801-b031-f707a75f7229",
      "status": "pending",
      "variant_index": 1,
      "format": "tile-2x3-2x",
      "variant_seed": 1183121173,
      "engine": "pro"
    }
  ],
  "polling_url": "/v1/masters/jobs/1b36df48-ab6d-4997-9725-19fb76d33189"
}
```

The same key, a different background:

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Idempotency-Key: $IDEM_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "game": {
    "id": "$GAME_ID"
  },
  "art": {
    "style": "dark-premium",
    "background": "radial-glow",
    "composition": "hero_upper_60_lower_third_clear"
  }
}
EOF

HTTP 409
x-environment: test
x-tokens-charged: 0

{
  "error": {
    "code": "idempotency_key_reuse",
    "message": "(human-readable, not part of the contract)"
  }
}
```

A repeat that arrives while the first request with the key is still being processed gets `409 idempotency_key_in_progress`; wait a couple of seconds and repeat with the same key. A key that was used for a refused request, for example for a short balance or a price above `max_tokens`, cannot carry the corrected request: send it with a new key.

#### 409 price_above_limit

Both calls that spend accept `max_tokens`, the most you agreed to pay for this call. If the call would cost more on a production key, it is refused before anything is held, and `details` names the `price` and your `max_tokens` in tokens. A sandbox key is judged by the production price too, so a limit that works in the sandbox works in production. Free repeats cost nothing and pass under any limit. Estimate again and send the new price:

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "game": {
    "id": "$GAME_ID"
  },
  "art": {
    "style": "dark-premium",
    "background": "radial-glow",
    "composition": "hero_upper_60_lower_third_clear"
  },
  "max_tokens": 0
}
EOF

HTTP 409
x-environment: test
x-tokens-charged: 0

{
  "error": {
    "code": "price_above_limit",
    "message": "(human-readable, not part of the contract)",
    "details": {
      "price": 1,
      "max_tokens": 0,
      "unit": "tokens"
    }
  }
}
```

#### 402 insufficient_tokens

The balance does not cover the operation; nothing is held and no job is created. `details` carries `balance`, `available`, `required` and a `top_up_url`, all amounts in tokens. Compare `required` with `available`, not with `balance`: tokens held by jobs still running are part of the balance but cannot be spent. Recorded through the `insufficient_tokens` test scenario of the sandbox, which answers with the same status, code and `details` fields as production does when the balance is short:

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "X-Test-Scenario: insufficient_tokens" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "game": {
    "id": "$GAME_ID"
  },
  "art": {
    "style": "dark-premium",
    "background": "rays-vignette",
    "composition": "hero_upper_60_lower_third_clear"
  }
}
EOF

HTTP 402
x-environment: test
x-tokens-charged: 0

{
  "error": {
    "code": "insufficient_tokens",
    "message": "Not enough tokens; nothing was charged. details.required is the price, details.available what can be spent now, details.top_up_url where to top up.",
    "details": {
      "balance": 0,
      "available": 0,
      "required": 1,
      "top_up_url": "https://…/account"
    }
  }
}
```

#### 422 font_coverage

The caption cannot be drawn: the game title or the studio name has characters that none of the available fonts covers. `details.uncovered` lists them, `details.role` says which line and `details.text` quotes it. `details.operation` says which call refused, `render` or `export`, so the advice below can follow it. `POST /v1/renders` answers this for the layers you sent: pass another text or hide that line. An export answers it before anything is held when a new set would get the default caption: draw your own title with `POST /v1/renders` and export with its `overlay_config_id`, or export with `"none"` for art without a caption.

#### 429 rate_limited

Too many requests. `Retry-After` is on every 429 and is the number of seconds to wait; retry after it, with backoff if it repeats. Recorded through the `rate_limited` test scenario. The `X-RateLimit-*` headers in this example are what the sandbox scenario adds; do not rely on them being present on production responses.

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "X-Test-Scenario: rate_limited" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "game": {
    "id": "$GAME_ID"
  },
  "art": {
    "style": "dark-premium",
    "background": "rays-vignette",
    "composition": "hero_upper_60_lower_third_clear"
  }
}
EOF

HTTP 429
x-environment: test
x-tokens-charged: 0
retry-after: 12
x-ratelimit-limit: 60
x-ratelimit-remaining: 0
x-ratelimit-reset: 1791818067

{
  "error": {
    "code": "rate_limited",
    "message": "Too many requests",
    "details": {
      "retry_after": 12
    }
  }
}
```

#### 500 internal_error

Any unhandled failure on our side has one shape. The body never carries the text, class or stack of the underlying error; `request_id` is always present and matches our journal, so a support ticket with it goes straight to the cause. Retry with backoff; if it persists, write to support with the identifier.

```json
{
  "error": {
    "code": "internal_error",
    "message": "Internal error",
    "request_id": "c03f11d4-8f31-440a-a543-a12db987d274"
  }
}
```

Refusals by the HTTP layer itself come in the same envelope, each with its own code: a body that is not valid JSON gets `400 invalid_json`, a body over the parser limit gets `413 payload_too_large`, and a body in an encoding or charset the API does not read gets `415 unsupported_media_type`. Any other refusal of the HTTP layer keeps its own status with the residual code `http_error`.

#### Every code

| HTTP | code | Retry | What to do |
| --- | --- | --- | --- |
| 400 | `validation_failed` | no | Read `details.issues[]`, fix the request. |
| 400 | `invalid_json` | no | The body is not valid JSON; fix the body. |
| 400 | `master_not_ready` | after the job finishes | The variant is not ready: export of a master whose generation has not finished, or `POST /v1/renders` for a variant that has an image but is not `ready`, for example one rejected by quality control; `details.status` says which. Only a ready variant has a free preview; a variant without an image answers 404. |
| 401 | `unauthenticated` | no | Check the key; `details.reason` says why it was refused. |
| 402 | `insufficient_tokens` | after topping up | `details.required` is the price, `details.available` what can be spent now, `details.top_up_url` where to top up. |
| 403 | `forbidden_scope` | no | `details.required_scope` names the missing scope. |
| 403 | `daily_token_cap_exceeded` | after `details.resets_at` | Only on calls that spend: this call would take the key or the connected agent over its daily cap. `details.cap`, `details.used` and `details.required` are in tokens, `details.credential` says whose cap it is. |
| 403 | `spending_frozen` | no | Spending is frozen for the whole organisation; topping up does not lift it. Write to support. |
| 403 | `ip_not_allowed` | no | The key has an IP allow-list and the caller is not on it. |
| 404 | `not_found` | no | Wrong identifier or a resource of another organisation. |
| 409 | `idempotency_key_reuse` | no | Same `Idempotency-Key`, different body; use a new key. |
| 409 | `idempotency_key_in_progress` | yes, in a couple of seconds | The first request with this key is still being processed. |
| 409 | `price_above_limit` | after a new estimate | The call costs more than `max_tokens`; `details.price` is the current price. |
| 409 | `format_retry_used` | no | The one free retry of this format has been used for this set. If it failed, the tokens for the lost formats are in `tokens_refunded`. |
| 409 | `format_retry_not_applicable` | no | This format cannot be retried: it is delivered, not ordered, cancelled by you, refused by the model, or the set failed. `details.why` says which. |
| 409 | `export_set_busy` | yes, after the build ends | Another build of this set is running; retry the format when it ends. |
| 422 | `batch_export_items_invalid` | no | Some chosen batch items have no ready master; `details.items` lists each with its reason. Nothing was charged. |
| 409 | `batch_export_in_progress` | yes, after the build ends | A format build of this batch is already running; `details.order_id` names it. |
| 409 | `sandbox_key_on_live_set` | no | A test key cannot buy a file container of a live set. Nothing was charged; buy it with a live key, `details.would_charge` is the price. |
| 409 | `follows_job_not_ready` | yes, after that job finishes | The job named in `follows_job_id` has no ready master yet. Nothing was charged. |
| 409 | `batch_retry_in_progress` | yes, after those items end | Some chosen batch items are still running; `details.items` lists them. Nothing was charged. |
| 422 | `batch_retry_not_applicable` | no | Some chosen batch items cannot be retried; `details.items` lists each with its reason (`unmatched`, `cancelled`, `not_failed`, `nothing_to_regenerate`, `not_in_batch`, `game_not_found`). Nothing was charged. |
| 422 | `batch_retry_of_child` | no | The batch is itself a retry; send the retry to the root batch in `details.root_batch_id`. Nothing was charged. |
| 409 | `sandbox_key_on_live_batch` | no | A test key cannot retry items of a live batch. Nothing was charged; retry with a live key, `details.would_charge` is the price. A live key on a sandbox batch retries it in the sandbox at no charge. In the sandbox the free retry counts as used once the sandbox attempt has failed; live, once a call to the image generator was made — a call the generator refused without doing any work (HTTP 429 quota, 401, 403) does not use it. |
| 413 | `payload_too_large` | no | The body is over the parser limit; send a smaller body. |
| 415 | `unsupported_media_type` | no | Send the body as UTF-8 JSON without content encoding. |
| 422 | `font_coverage` | with another caption | `details.uncovered` lists the characters no font can draw; `details.operation` is `render` or `export`. |
| 422 | `follows_with_variant_seed` | no | A pair that follows another job always gets a new seed; send `variant_seed: null`. |
| 422 | `follows_job_mismatch` | no | The game or art parameters differ from the job named in `follows_job_id`; repeat them, or generate without `follows_job_id`. |
| 429 | `rate_limited` | yes, after `Retry-After` | Slow down; back off if it repeats. |
| 500 | `internal_error` | yes, with backoff | Quote `request_id` to support if it persists. |
| 500 | `provider_unavailable` | yes, after a minute | The image model did not answer; nothing was charged. |

#### Export reasons are codes, not prose

Two fields of the export responses carry a reason, and both are machine codes from a closed set, never a sentence. Your interface chooses the wording. The sets are closed on the producing side; an unknown code, should one arrive from a newer server, is to be shown as is, not treated as an error.

##### estimate.reason

`GET /v1/masters/{id}/export/estimate` explains why a build is free. `null` means it is paid and there is nothing to explain; the price and the branch are decided by `plan`, not by this field.

| code | Meaning |
| --- | --- |
| `build_in_flight` | A build of this set is already running. |
| `format_retry_in_flight` | A free retry of one format of this set is running; the published set and its archive stay available meanwhile. |
| `resume_exhausted` | The build was lost from the queue too many times, restarts stopped, the hold is returned. |
| `resume_from_hold` | The build was lost from the queue; it continues from the tokens already held. |
| `reoverlay_after_loss` | The build was lost from the queue; the set is re-rendered for free. |
| `files_pending_render` | The art is paid for, the output files are still being rendered. |
| `overlay_changed` | A rebuild with a different overlay of already paid art. |
| `container_changed` | A rebuild into a different file container; which one is in `file_format`. |
| `topup_free` | The set is paid for in full, and the formats you asked for that it does not hold yet are added to it for free. |
| `set_ready` | The set is built; repeats and downloads are free. |
| `sandbox_fixture` | A sandbox master: the set is built from a fixture, no tokens. |
| `sandbox_test_key` | A production master exported with a sandbox key: no model call, no tokens. |

##### missing[].reason

`GET /v1/exports/{id}` lists every promised format that is not in the set under `missing`, each with a reason. The field is never empty: when no reason was recorded, it is derived from the state of the row.

| code | Meaning |
| --- | --- |
| `not_ordered` | The format was never part of this set. |
| `not_in_build` | The format was ordered but is not in this published build. |
| `not_rendered` | The art exists, the output file was not rendered. |
| `not_started` | The format has not been started yet. |
| `in_progress` | The format is still being built. |
| `not_assembled` | The format failed to build; details stayed in the server journal. |
| `qa_rejected` | The format was rejected by quality control. |
| `build_cancelled` | The build was cancelled before the format was done. |
| `text_in_art` | Quality control found text in the frame that was not ordered. |
| `artifact` | Quality control found a technical defect in the frame. |
| `content_refused` | The model declined to draw the brief. |
| `style_anchor_unavailable` | The primary model was down and the fallback does not take the uploaded style sample. |
| `sandbox_spend_blocked` | Sandbox: the spend gate refused the call, which is the intended behaviour. |
| `attempts_exhausted` | Both paid attempts at this format have been used; it will not be tried again, and its share is returned as tokens. |
| `spend_cap_reached` | Our own spending limit for the model was reached before this format was requested, so no call was made. The format keeps its second attempt and can be built by a later export once the limit lifts; if both attempts are already spent, `missing[].retry` offers the free retry instead. Formats of one set are built at the same time, so which of them hit the limit is not repeatable from run to run; the record in `missing` is what tells you. |
| `not_selected` | Not a shortfall: the format is part of the set but was not ticked for this delivery. It comes in its own `not_selected` array, never in `missing`, and a later export that names it adds it for free while `available_free` is `true`. |
| `not_delivered` | Fallback: the format was not delivered and a more precise reason is not published. |

Next to the reason, `missing[].retry` says whether the format can be retried for free with `POST /v1/exports/{id}/formats/{format}/retry`: `available`, `in_progress`, `used` (the one retry is spent and the format did not assemble again), or `null` (no retry for this format). When a drawn format is lost for good, the set returns 0.4 tokens for every drawn format it is missing, once per set; the amount is in `tokens_refunded` of the same response.

##### failure.code

A job that ended in nothing names the reason with a code from one closed set, and the same set serves both paths: `failure.code` of a single generation job (`GET /v1/masters/jobs/{id}`) and `items[].error_code` of a batch item (`GET /v1/batches/{id}`). `retryable` answers a different question from `code`: it says whether repeating the same request can succeed, not whether repeating is permitted.

Two values are not in the set and appear only where a cause was never recorded: `generation_failed` and `export_failed` on items of batches that finished before this set existed. Treat both as "failed, reason unknown".

A single generation job that failed before v0.20.0 may come without `failure` at all: before that release, a job whose attempts were all rejected by quality control or got no answer from the image model was closed without a code. Treat a failed job without `failure` the same way, as "failed, reason not recorded".

| code | Retry | What it means |
| --- | --- | --- |
| `content_refused` | no | The model declined to draw the brief. The same brief will be declined again; change the wording. |
| `style_anchor_unavailable` | yes | The job carried your uploaded style sample, the primary model was down, and the fallback does not accept uploaded samples. Retry in a few minutes, or drop the sample. |
| `processing_error` | yes | A failure on our side, not in the model. Nothing was kept, tokens were returned. |
| `qa_rejected` | no | Automatic quality control rejected every attempt. Repeating with the same settings gives the same verdict; change the style or the background. |
| `provider_error` | yes | The image model call did not go through: network, a rate limit, or a tripped breaker. |
| `item_data_invalid` | no | A batch item could not be executed. Recreate the batch; quote `request_id` to support. |
| `export_build_failed` | yes | The art was produced and paid for, the file set was not assembled. Export the master again; price it with `GET /v1/masters/{id}/export/estimate`. |

##### missing[].status

The state of the promised format, in the same `missing[]` entry. Also a code from a closed set, not a raw database value.

| code | Meaning |
| --- | --- |
| `absent` | No row for this format in the set. |
| `not_in_build` | The row is ready but the format is not in this published build. |
| `not_rendered` | The art is ready, no output file was rendered for it. |
| `pending` | Not started. |
| `queued` | Waiting in the queue. |
| `generating` | Being rendered. |
| `qa` | Under quality control, no verdict yet. |
| `failed` | Failed to build. |
| `rejected` | Rejected by quality control. |
| `cancelled` | The build was cancelled. |
| `unknown` | Fallback: the state is not known and a more precise one is not published. |
