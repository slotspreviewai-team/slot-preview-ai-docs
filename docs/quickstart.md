# Quickstart

Canonical page: https://slotspreviewai.com/docs/quickstart

Source: https://slotspreviewai.com/docs/quickstart

### Quickstart: the first tile in five steps

Every step below is one command against the live API with a sandbox key. The responses are recorded from a real run and re-checked against the API before every release, so what you see here is what the server sends.

#### Before you start

You need a sandbox key. Sign up, open the API keys tab of your account and create a key: sandbox is the default environment there. Then export the address and the key once per shell session. The address has no `/v1` suffix: every path in this guide starts with it.

```sh
export SLOTPREVIEW_BASE_URL=https://api.slotspreviewai.com
export SLOTPREVIEW_API_KEY=spai_test_…
```

A sandbox key charges nothing and never calls the image model: generation returns one of the built-in fixtures, export assembles the set from it. The whole quickstart is free to repeat as often as you like. Everything else, including the async model and the `X-Environment` header on every response to an accepted key, works exactly as on a production key. See [Authentication and environments](https://slotspreviewai.com/docs/authentication).

#### Step 1. Register the game

Tiles are wanted before a slot ships, so the first call registers your title. Sending the same name and studio again returns the same record instead of a duplicate, which makes the call safe to retry. Your identifiers will differ from the ones recorded here; the following steps take them from the previous response, never from this page.

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/games" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "name": "Vault of Ember",
  "provider_name": "Ember Studios",
  "theme_hint": "volcanic vault, molten gold, obsidian",
  "release_date": "2026-11-05"
}
EOF

HTTP 200
x-environment: test
x-tokens-charged: 0

{
  "id": "e9d6da95-a726-4010-96df-df39c40084a9",
  "slug": "vault-of-ember",
  "name": "Vault of Ember",
  "provider": {
    "id": "bf084b80-3dd4-4e94-9095-c0c8e01ca82b",
    "slug": "ember-studios",
    "name": "Ember Studios"
  },
  "visibility": "private",
  "art_brief": {
    "theme": "volcanic vault, molten gold, obsidian"
  },
  "brief_status": "ready",
  "brief_confidence": "verified"
}
```

The record is private to your organisation. When the call creates the studio as well, `provider` also carries `created: true`; a studio that already exists comes without it. `brief_status` tells whether the art brief the generator writes from the name and `theme_hint` is ready; you do not have to wait for it. If the game is already in the shared catalogue, find it by name and take its `id` instead:

```
curl -s "$SLOTPREVIEW_BASE_URL/v1/games?q=Vault+of+Ember" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY"

HTTP 200
x-environment: test
x-tokens-charged: 0

{
  "data": [
    {
      "id": "e9d6da95-a726-4010-96df-df39c40084a9",
      "slug": "vault-of-ember",
      "name": "Vault of Ember",
      "provider": {
        "id": "bf084b80-3dd4-4e94-9095-c0c8e01ca82b",
        "slug": "ember-studios",
        "name": "Ember Studios"
      },
      "release_date": "2026-11-05",
      "is_upcoming": true,
      "showcase_preview_url": null,
      "brief_confidence": "verified"
    }
  ],
  "next_cursor": null,
  "has_more": false
}
```

#### Step 2. Generate a master

One call, two variants, asynchronous. The `Idempotency-Key` header makes a retry safe: the same key with the same body returns the original job and charges nothing, the same key with a different body is rejected with `409 idempotency_key_reuse`. `art.style` and `art.background` accept the codes from `GET /v1/styles` and `GET /v1/backgrounds`. `max_tokens` is the most you agree to pay for this call: if it would cost more on a production key, it is refused with `409 price_above_limit` before anything is held. `POST /v1/masters/estimate` tells the price in advance.

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
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
  },
  "format_group": "tile",
  "max_tokens": 1
}
EOF

HTTP 202
x-environment: test
x-tokens-charged: 0
x-tokens-would-charge: 1

{
  "id": "ebd42491-0c58-42cd-84bd-d818eeaddadb",
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
      "id": "c7a67559-3bf7-40da-9bf9-a865d6921c9f",
      "status": "pending",
      "variant_index": 0,
      "format": "tile-2x3-2x",
      "variant_seed": 1399843477,
      "engine": "pro"
    },
    {
      "id": "90ad2017-3014-455e-848e-c7d2ca266de1",
      "status": "pending",
      "variant_index": 1,
      "format": "tile-2x3-2x",
      "variant_seed": 1399843476,
      "engine": "pro"
    }
  ],
  "polling_url": "/v1/masters/jobs/ebd42491-0c58-42cd-84bd-d818eeaddadb"
}
```

The answer is `202 Accepted` with a job, not an image. `X-Tokens-Charged` is what this call took from your balance (nothing on a sandbox key), `X-Tokens-Would-Charge` is what the same call costs on a production key.

#### Step 3. Wait for the job

Poll `polling_url` from the previous response until `status` leaves `queued` and `running`. A finished job is `succeeded`, `partial` (one variant passed quality control) or `failed`. In the sandbox this takes a few seconds; on a production key expect tens of seconds.

```
curl -s "$SLOTPREVIEW_BASE_URL/v1/masters/jobs/$JOB_ID" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY"

HTTP 200
x-environment: test
x-tokens-charged: 0

{
  "id": "ebd42491-0c58-42cd-84bd-d818eeaddadb",
  "object": "generation_job",
  "kind": "generate",
  "status": "succeeded",
  "tokens_held": 0,
  "tokens_charged": 0,
  "engine": "pro",
  "anchor": "none",
  "reference": "none",
  "started_at": "2026-10-12T15:14:00.945Z",
  "finished_at": "2026-10-12T15:14:00.952Z",
  "masters": [
    {
      "id": "c7a67559-3bf7-40da-9bf9-a865d6921c9f",
      "status": "ready",
      "variant_index": 0,
      "format": "tile-2x3-2x",
      "variant_seed": 1399843477,
      "engine": "pro",
      "quality": {
        "passed": true
      },
      "render_url": "https://…/spai-derived/derived/00000000-0000-0000-0000-0000000000f0/c7a67559-3bf7-40da-9bf9-a865d6921c9f/68b02bdfc588fa768e4287a198d5e6e8f535d9d311d3872fcd701b5ef4c58d6c.webp?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&x-amz-checksum-mode=…&x-id=…",
      "download_url": "https://…/spai-derived/fixtures/sandbox-7.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
    },
    {
      "id": "90ad2017-3014-455e-848e-c7d2ca266de1",
      "status": "ready",
      "variant_index": 1,
      "format": "tile-2x3-2x",
      "variant_seed": 1399843476,
      "engine": "pro",
      "quality": {
        "passed": true
      },
      "render_url": "https://…/spai-derived/derived/00000000-0000-0000-0000-0000000000f0/90ad2017-3014-455e-848e-c7d2ca266de1/071280510587f0a1975d9dd54f9b3a83ba3178563ae780b6e47d94af47e171b0.webp?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&x-amz-checksum-mode=…&x-id=…",
      "download_url": "https://…/spai-derived/fixtures/sandbox-8.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
    }
  ],
  "polling_url": "/v1/masters/jobs/ebd42491-0c58-42cd-84bd-d818eeaddadb"
}
```

Both variants are listed under `masters` with their own `id`. A variant whose `status` is `ready` carries two links: `render_url` is a WebP preview without a caption, at most 600 pixels on the long side, and `download_url` is the full-size file. Open `render_url` in a browser to pick the variant you like; looking costs nothing. Both links are signed and expire after one hour; poll the job again for fresh ones. A variant that is not ready has neither. The rest of the guide uses the first one. In production, prefer a webhook over polling: subscribe with `POST /v1/webhook-endpoints` and receive `master.generation.completed` instead of asking; the webhook carries no links, read the job for them.

#### Step 4. Preview the caption and export the tile set

A tile in the lobby carries a caption with the game title and the studio name. Draw it on the variant first, for free: `POST /v1/renders` with empty `layers` draws the default caption, `layers.title.text` sets your own title (up to 40 characters) and `"visible": false` hides a line. The answer holds `overlay_config_id` and the address of the picture it drew.

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/renders" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "master": "$MASTER_ID",
  "layers": {},
  "outputs": [
    {
      "name": "preview",
      "width": 400,
      "height": 600,
      "format": "webp"
    }
  ]
}
EOF

HTTP 201
x-environment: test
x-tokens-charged: 0

{
  "master": "c7a67559-3bf7-40da-9bf9-a865d6921c9f",
  "overlay_config_id": "9cde4f99-5dec-421a-8d72-0ad255973501",
  "layers_hash": "44136fa355b3678a1146ad16f7e8649e94fb4fc21fe77e8310c060f61caaff8a",
  "render_ms": 117,
  "outputs": [
    {
      "name": "preview",
      "format": "webp",
      "width": 400,
      "height": 600,
      "bytes": 25338,
      "url": "https://…/spai-derived/derived/00000000-0000-0000-0000-0000000000f0/c7a67559-3bf7-40da-9bf9-a865d6921c9f/f5a4f98f4951f32ae80de13778d07b2d4d24473f5f908467825f2a8001685778.webp?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&x-amz-checksum-mode=…&x-id=…",
      "immutable": true,
      "caption_zone": {
        "zone": [
          0,
          338,
          400,
          594
        ],
        "mean_rgb": [
          77,
          41,
          24
        ]
      },
      "caption_zone_variants": {
        "outline-heavy|auto": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            120,
            63,
            42
          ]
        },
        "outline-heavy|on": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            91,
            48,
            29
          ]
        },
        "outline-heavy|off": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            120,
            63,
            42
          ]
        },
        "outline-thin|auto": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            120,
            63,
            42
          ]
        },
        "outline-thin|on": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            91,
            48,
            29
          ]
        },
        "outline-thin|off": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            120,
            63,
            42
          ]
        },
        "shadow-only|auto": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            120,
            63,
            42
          ]
        },
        "shadow-only|on": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            91,
            48,
            29
          ]
        },
        "shadow-only|off": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            120,
            63,
            42
          ]
        },
        "plate|auto": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            46,
            24,
            19
          ]
        },
        "plate|on": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            33,
            17,
            12
          ]
        },
        "plate|off": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            46,
            24,
            19
          ]
        },
        "gradient-scrim|auto": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            77,
            41,
            24
          ]
        },
        "gradient-scrim|on": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            77,
            41,
            24
          ]
        },
        "gradient-scrim|off": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            77,
            41,
            24
          ]
        },
        "outline-tinted|auto": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            120,
            63,
            42
          ]
        },
        "outline-tinted|on": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            91,
            48,
            29
          ]
        },
        "outline-tinted|off": {
          "zone": [
            0,
            338,
            400,
            594
          ],
          "mean_rgb": [
            120,
            63,
            42
          ]
        }
      }
    }
  ],
  "cache_hits": 0
}
```

Export renders every format of the group from the master. It is the second and last call that costs tokens. `overlay_config_id` says which caption the files carry: the id from the preview gives exactly the tile you saw, `"none"` gives the art without a caption. Leave the field out and a new set gets the default caption; repeating the call without the field never changes a set that is already built. In the recorded set `has_overlay` is true on the first file. Send an `Idempotency-Key` here too: a retry with the same key returns the first answer and charges nothing, even if that build has since failed and returned its tokens.

```
curl -s -X POST "$SLOTPREVIEW_BASE_URL/v1/masters/$MASTER_ID/export" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY" \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d @- <<EOF
{
  "overlay_config_id": "$OVERLAY_CONFIG_ID",
  "hero_position": "center",
  "max_tokens": 2
}
EOF

HTTP 202
x-environment: test
x-tokens-charged: 0
x-tokens-would-charge: 2

{
  "id": "15c0dfb9-35c7-47e4-acdd-123d731b4a68",
  "object": "generation_job",
  "kind": "export",
  "status": "queued",
  "tokens_held": 0,
  "tokens_charged": 0,
  "source_master": "c7a67559-3bf7-40da-9bf9-a865d6921c9f",
  "export_set_id": "ae6c52d8-dabe-44cd-96a8-d654105c32ef",
  "formats": [
    "tile-2x3",
    "tile-2x3-2x",
    "tile-3x4",
    "tile-1x1",
    "story-9x16",
    "banner-16x9",
    "banner-21x9"
  ],
  "layers_included": true,
  "progress": [
    {
      "format": "tile-2x3-2x",
      "status": "ready"
    },
    {
      "format": "tile-3x4",
      "status": "pending"
    },
    {
      "format": "tile-1x1",
      "status": "pending"
    },
    {
      "format": "story-9x16",
      "status": "pending"
    },
    {
      "format": "banner-16x9",
      "status": "pending"
    },
    {
      "format": "banner-21x9",
      "status": "pending"
    },
    {
      "format": "tile-2x3",
      "status": "pending"
    }
  ],
  "polling_url": "/v1/exports/jobs/15c0dfb9-35c7-47e4-acdd-123d731b4a68",
  "file_format": "png",
  "file_formats": [
    "png"
  ],
  "last_build": {
    "overlay_config_id": "9cde4f99-5dec-421a-8d72-0ad255973501",
    "build_no": 1,
    "requested_at": "2026-10-12T15:14:04.655Z",
    "published_at": null,
    "outcome": null,
    "outcome_reason": null
  }
}
```

The job is asynchronous like generation. `formats` lists what will be rendered, `progress` fills in per format as the set is built.

#### Step 5. Get the files

First wait for the export job, then read the finished set: the set holds the file addresses, the job does not. A finished export is `succeeded` when every format passed quality control and `partial` when some did not; both are done, and a partial set still has files.

```
curl -s "$SLOTPREVIEW_BASE_URL/v1/exports/jobs/$EXPORT_JOB_ID" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY"

HTTP 200
x-environment: test
x-tokens-charged: 0

{
  "id": "15c0dfb9-35c7-47e4-acdd-123d731b4a68",
  "object": "generation_job",
  "kind": "export",
  "status": "partial",
  "tokens_held": 0,
  "tokens_charged": 0,
  "source_master": "c7a67559-3bf7-40da-9bf9-a865d6921c9f",
  "export_set_id": "ae6c52d8-dabe-44cd-96a8-d654105c32ef",
  "formats": [
    "tile-2x3",
    "tile-2x3-2x",
    "tile-3x4",
    "tile-1x1",
    "story-9x16",
    "banner-16x9",
    "banner-21x9"
  ],
  "layers_included": true,
  "progress": [
    {
      "format": "tile-2x3-2x",
      "status": "ready"
    },
    {
      "format": "tile-1x1",
      "status": "rejected"
    },
    {
      "format": "story-9x16",
      "status": "rejected"
    },
    {
      "format": "tile-3x4",
      "status": "ready"
    },
    {
      "format": "banner-16x9",
      "status": "rejected"
    },
    {
      "format": "tile-2x3",
      "status": "ready"
    },
    {
      "format": "banner-21x9",
      "status": "ready"
    }
  ],
  "polling_url": "/v1/exports/jobs/15c0dfb9-35c7-47e4-acdd-123d731b4a68",
  "file_format": "png",
  "file_formats": [
    "png"
  ],
  "last_build": {
    "overlay_config_id": "9cde4f99-5dec-421a-8d72-0ad255973501",
    "build_no": 1,
    "requested_at": "2026-10-12T15:14:04.655Z",
    "published_at": "2026-10-12T15:14:13.397Z",
    "outcome": "partial",
    "outcome_reason": null
  }
}
```

`export_set_id` is filled in once the set exists. Read it:

```
curl -s "$SLOTPREVIEW_BASE_URL/v1/exports/$EXPORT_SET_ID" \
  -H "Authorization: Bearer $SLOTPREVIEW_API_KEY"

HTTP 200
x-environment: test
x-tokens-charged: 0

{
  "export_set_id": "ae6c52d8-dabe-44cd-96a8-d654105c32ef",
  "status": "partial",
  "files": [
    {
      "format": "tile-2x3",
      "filename": "tile-2x3.png",
      "width": 512,
      "height": 768,
      "bytes": 456525,
      "checksum": "a294e0fba2b69b2f9afe914ddbda94556e423c11beea833a317119c64a558a7a",
      "mime": "image/png",
      "has_overlay": true,
      "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/3438944d-ed16-4db1-b658-23e4e5507938/80c0fcfe06a3d8ea49cd90b81d6f8310528359039e67d12fb7b74d781945c022.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
    },
    {
      "format": "tile-2x3-2x",
      "filename": "tile-2x3-2x.png",
      "width": 1024,
      "height": 1536,
      "bytes": 1689564,
      "checksum": "a160f4bf39ee63a24677bbe8a5ed6bbef05abaabb1504ac332b7d21cafe08783",
      "mime": "image/png",
      "has_overlay": true,
      "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/c7a67559-3bf7-40da-9bf9-a865d6921c9f/da199e21fd0338b43923c5b2ea5643d07957bb995533dc8d463fe2c85c83ecf4.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
    },
    {
      "format": "tile-3x4",
      "filename": "tile-3x4.png",
      "width": 768,
      "height": 1024,
      "bytes": 897406,
      "checksum": "60035fedaa9e63ee3d912153c4865414a86291e3c5a907cca6f188a436e7df8c",
      "mime": "image/png",
      "has_overlay": true,
      "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/78337044-3e9a-4713-ad57-795a5bbcac57/84439b5d003facef8a7650c214e5dd7384c9f36b5240a740f618744e33f0ae38.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…",
      "sizes": [
        {
          "filename": "tile-3x4-427x575.png",
          "width": 427,
          "height": 575,
          "bytes": 318858,
          "checksum": "a255cf531631ede28b3beaa6eb91f79374494fb2d0fb127fb81dc68e29ab8f0d",
          "mime": "image/png",
          "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/78337044-3e9a-4713-ad57-795a5bbcac57/4e4eed0190c485af7d1a670418ad8918a72409b9c4cd71728ee50c9bc879ae9d.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
        },
        {
          "filename": "tile-3x4-316x425.png",
          "width": 316,
          "height": 425,
          "bytes": 185731,
          "checksum": "1dbed11d7ed74a701187fdcac27b194b5d4c67baad0e324f99777d3293279376",
          "mime": "image/png",
          "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/78337044-3e9a-4713-ad57-795a5bbcac57/68f6801d7fc14ac584e80c5d1108147e9381215648c96cfe02125989a1cfffc4.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
        }
      ]
    },
    {
      "format": "banner-21x9",
      "filename": "banner-21x9.png",
      "width": 1540,
      "height": 660,
      "bytes": 1249639,
      "checksum": "c25dd2c3b5bc3dd018cad5490dd28655d46874ae1cc867e849562eb8964959f7",
      "mime": "image/png",
      "has_overlay": true,
      "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/58188316-52ce-4c1c-aba7-0139921e6049/920c1199c141e42774b582538456d7d29f0adf8da3af671d32e9153a369dfd82.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…",
      "sizes": [
        {
          "filename": "banner-21x9-236x110.png",
          "width": 236,
          "height": 110,
          "bytes": 48330,
          "checksum": "c932c5e7b12f4f3fdb8f36093dbecc686fda092e8be4a8aa22125f61fa014491",
          "mime": "image/png",
          "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/58188316-52ce-4c1c-aba7-0139921e6049/bb8a925f7bdaef25cb5fec9c4b965b8b44ccd56fdda84404b4a68138fc1bf674.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
        }
      ]
    }
  ],
  "file_format": "png",
  "containers": [
    {
      "file_format": "png",
      "files": [
        {
          "format": "tile-2x3",
          "filename": "tile-2x3.png",
          "width": 512,
          "height": 768,
          "bytes": 456525,
          "checksum": "a294e0fba2b69b2f9afe914ddbda94556e423c11beea833a317119c64a558a7a",
          "mime": "image/png",
          "has_overlay": true,
          "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/3438944d-ed16-4db1-b658-23e4e5507938/80c0fcfe06a3d8ea49cd90b81d6f8310528359039e67d12fb7b74d781945c022.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
        },
        {
          "format": "tile-2x3-2x",
          "filename": "tile-2x3-2x.png",
          "width": 1024,
          "height": 1536,
          "bytes": 1689564,
          "checksum": "a160f4bf39ee63a24677bbe8a5ed6bbef05abaabb1504ac332b7d21cafe08783",
          "mime": "image/png",
          "has_overlay": true,
          "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/c7a67559-3bf7-40da-9bf9-a865d6921c9f/da199e21fd0338b43923c5b2ea5643d07957bb995533dc8d463fe2c85c83ecf4.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
        },
        {
          "format": "tile-3x4",
          "filename": "tile-3x4.png",
          "width": 768,
          "height": 1024,
          "bytes": 897406,
          "checksum": "60035fedaa9e63ee3d912153c4865414a86291e3c5a907cca6f188a436e7df8c",
          "mime": "image/png",
          "has_overlay": true,
          "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/78337044-3e9a-4713-ad57-795a5bbcac57/84439b5d003facef8a7650c214e5dd7384c9f36b5240a740f618744e33f0ae38.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…",
          "sizes": [
            {
              "filename": "tile-3x4-427x575.png",
              "width": 427,
              "height": 575,
              "bytes": 318858,
              "checksum": "a255cf531631ede28b3beaa6eb91f79374494fb2d0fb127fb81dc68e29ab8f0d",
              "mime": "image/png",
              "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/78337044-3e9a-4713-ad57-795a5bbcac57/4e4eed0190c485af7d1a670418ad8918a72409b9c4cd71728ee50c9bc879ae9d.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
            },
            {
              "filename": "tile-3x4-316x425.png",
              "width": 316,
              "height": 425,
              "bytes": 185731,
              "checksum": "1dbed11d7ed74a701187fdcac27b194b5d4c67baad0e324f99777d3293279376",
              "mime": "image/png",
              "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/78337044-3e9a-4713-ad57-795a5bbcac57/68f6801d7fc14ac584e80c5d1108147e9381215648c96cfe02125989a1cfffc4.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
            }
          ]
        },
        {
          "format": "banner-21x9",
          "filename": "banner-21x9.png",
          "width": 1540,
          "height": 660,
          "bytes": 1249639,
          "checksum": "c25dd2c3b5bc3dd018cad5490dd28655d46874ae1cc867e849562eb8964959f7",
          "mime": "image/png",
          "has_overlay": true,
          "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/58188316-52ce-4c1c-aba7-0139921e6049/920c1199c141e42774b582538456d7d29f0adf8da3af671d32e9153a369dfd82.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…",
          "sizes": [
            {
              "filename": "banner-21x9-236x110.png",
              "width": 236,
              "height": 110,
              "bytes": 48330,
              "checksum": "c932c5e7b12f4f3fdb8f36093dbecc686fda092e8be4a8aa22125f61fa014491",
              "mime": "image/png",
              "url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/58188316-52ce-4c1c-aba7-0139921e6049/bb8a925f7bdaef25cb5fec9c4b965b8b44ccd56fdda84404b4a68138fc1bf674.png?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…"
            }
          ]
        }
      ]
    }
  ],
  "zip_url": "https://…/spai-derived/export/00000000-0000-0000-0000-0000000000f0/ae6c52d8-dabe-44cd-96a8-d654105c32ef/1/bundle.zip?X-Amz-Algorithm=…&X-Amz-Content-Sha256=…&X-Amz-Credential=…&X-Amz-Date=…&X-Amz-Expires=…&X-Amz-Signature=…&X-Amz-SignedHeaders=…&response-content-disposition=…&x-amz-checksum-mode=…&x-id=…",
  "tokens_charged": 0,
  "tokens_refunded": 0,
  "paid_file_formats": [
    "png"
  ],
  "formats_expected": 7,
  "formats_ready": 4,
  "files_ready": 7,
  "missing": [
    {
      "format": "tile-1x1",
      "status": "rejected",
      "reason": "artifact",
      "retry": "available",
      "available_free": true
    },
    {
      "format": "story-9x16",
      "status": "rejected",
      "reason": "text_in_art",
      "retry": "available",
      "available_free": true
    },
    {
      "format": "banner-16x9",
      "status": "rejected",
      "reason": "artifact",
      "retry": "available",
      "available_free": true
    }
  ],
  "not_selected": [],
  "overlay_config_id": "9cde4f99-5dec-421a-8d72-0ad255973501"
}
```

Open `files[0].url` in a browser: it is a signed address of a real image, not a placeholder. Signed addresses expire after one hour; ask for the set again to get fresh ones. `missing` lists any format that is not in the set with a reason code, see [Errors and limits](https://slotspreviewai.com/docs/errors#export-reasons-are-codes-not-prose).

To change the caption of a built set, export the same master again with another `overlay_config_id` or with `"none"`: the set is rebuilt with the new caption for free. `GET /v1/exports/{exportSetId}/overlay-history` lists every caption the set has been built with, newest first, and exporting with one of those ids again goes back to it, also for free.

#### What next

- [Authentication and environments](https://slotspreviewai.com/docs/authentication): key format, scopes, the sandbox and its test scenarios.
- [Errors and limits](https://slotspreviewai.com/docs/errors): the error envelope, every code you can receive and what to do with it.
- [API reference](https://slotspreviewai.com/docs/api): every operation with request and response schemas, plus the Postman collection.
