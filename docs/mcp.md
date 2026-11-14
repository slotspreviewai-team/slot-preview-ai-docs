# MCP server

Canonical page: https://slotspreviewai.com/docs/mcp

Source: https://slotspreviewai.com/docs/mcp

### MCP server for AI agents

The MCP server lets an AI agent make lobby tiles for you: find or register a game, price and generate a master, show you both variants, preview the caption and export the format set. It works with your balance and the same prices as the REST API, and it adds no paid operation of its own. It is a remote server over Streamable HTTP, so there is nothing to install.

There are two ways to connect. ChatGPT and claude.ai connect with sign-in: you add the server address in the app, sign in on our page and choose what the agent may do. Claude Code, Cursor, the OpenAI Responses API, the Claude API and your own agent on an MCP SDK connect with an API key in a header. The server is not listed in any connector directory: you connect it by its address.

#### Address

The server lives at the API address with the path `/mcp`, exactly, without a trailing slash. The Agents (MCP) tab of the account shows the full address with a button to copy it. For the key examples below, export the API address and a key once per shell session, the same two values as in the other guides:

```sh
export SLOTPREVIEW_BASE_URL=https://api.slotspreviewai.com
export SLOTPREVIEW_API_KEY=spai_test_…
```

The server address is then `$SLOTPREVIEW_BASE_URL/mcp`. Every request carries a credential as `Authorization: Bearer`, either the key or the token the app received at sign-in, and it is checked on every request, so a key you revoke or an agent you disconnect stops working on the next call. A credential in the address is not accepted. A request without a valid credential gets HTTP 401 from the server itself, not an error inside a tool, so the client shows the connection as broken. The server keeps no copy of the credential and never writes it to its logs.

#### Connect with sign-in

ChatGPT and claude.ai add the server as a connector by its address and send you to our sign-in page; the agent then works with the scopes you allowed there, and spends the tokens of the organization you chose.

##### ChatGPT

ChatGPT needs a Plus or higher plan, on the web, with developer mode turned on.

1. In ChatGPT settings, turn on developer mode.
2. Create an app for a remote MCP server: paste the server address and choose OAuth as the sign-in method.
3. A Slot Preview AI sign-in page opens. Sign in, choose the organization, sandbox or live, what the agent may do and its daily cap, then press Allow.
4. The agent appears on the Agents (MCP) tab of the account.

ChatGPT asks you to confirm actions that change data. Your balance is protected by the daily cap either way.

##### claude.ai

Custom connectors work on every claude.ai plan; on Free you can add one. On Team and Enterprise plans the owner of the Claude organization adds the connector, then each member signs in.

1. In claude.ai, open Connectors and add a custom connector.
2. Paste the server address and keep the sign-in settings Claude suggests.
3. A Slot Preview AI sign-in page opens. Sign in, choose the organization, sandbox or live, what the agent may do and its daily cap, then press Allow.
4. The agent appears on the Agents (MCP) tab of the account.

##### What the sign-in page asks

The page first shows the name of the app and the address it returns to, in the same size. An app that published its identity document on its own domain is marked as confirmed by that domain; an app that only named itself is marked "name given by the app, not verified", so an app that calls itself ChatGPT but returns elsewhere is visible at once. An app that returns only to this computer, such as Claude Code with sign-in, gets a warning: allow it only if you started the connection yourself just now. Then you choose:

- the organization, when you are a member of more than one;
- sandbox or live: in the sandbox no tokens are spent and the images are placeholders;
- what the agent may do: `read` is always on; `render`, `generate` and `export` only as far as the app asked for them and your role allows them, so a viewer can give `read` and `render` only;
- a daily cap in tokens, 25 by default; it is required when the agent may spend, and it stops spending at that number each day;
- how long the connection lasts: 30, 90, 180 or 365 days, 90 by default; a connection that cannot spend lasts 365 days.

Allow connects the agent and charges nothing; Deny returns you to the app without a connection. The page must be answered within 10 minutes, after that start again from the app. Your email address must be confirmed before you can allow a connection.

##### Disconnect

Every agent connected with sign-in is on the Agents (MCP) tab of the account, with its scopes, cap, spending today, last use and expiry. The owner and admins see the agents of every member; other members see their own. Disconnect stops an agent on its next call; jobs it already started finish and are charged as usual. Disconnect all stops every agent connected with sign-in and leaves API keys alone. We send no email when an agent is connected, so this tab is the place to look. Changing your password does not disconnect agents. An agent stops working when the person who connected it leaves the organization, and it can do no more than that person's current role allows.

##### Enterprise organizations

In an enterprise organization every key needs an IP allow-list, and a key with an allow-list does not work through MCP. Connect agents there with sign-in: ChatGPT or claude.ai.

#### Connect with a key

##### A key for the agent

Create a separate key for the agent on the Agents (MCP) tab of the account: the form proposes a sandbox key with a daily cap of 25 tokens, and the key then appears on the API keys tab, where you revoke it. The API holds the cap on every call that spends, before anything is charged, also when the agent fires several calls at once, so an agent stuck in a loop stops at the cap and not at your balance. Give the key only the scopes the agent needs: `read` and `render` to look and preview, `generate` to generate masters, `export` to export sets. A key without `generate` or `export` can still estimate; the paid tool then answers `forbidden_scope` and nothing is charged.

A key with an IP allow-list does not work through MCP: the server connects to the API from its own address, not from your network, so such a key is refused on every tool. Use a key without the list for the agent.

Start in the sandbox. A sandbox key or a sandbox connection goes through the whole flow, including both paid tools, without spending a token: the images are built-in placeholders marked SANDBOX, every answer of a paid tool begins with the word sandbox and names what a live run would cost, and `check_connection` names the environment first.

The examples read the address and the key from the two variables above. Keep the key out of files you share.

##### Claude Code

Put this in `.mcp.json` at the root of your project. Claude Code expands the variables from the environment it starts in:

```json
{
  "mcpServers": {
    "slotpreview": {
      "type": "http",
      "url": "${SLOTPREVIEW_BASE_URL}/mcp",
      "headers": {
        "Authorization": "Bearer ${SLOTPREVIEW_API_KEY}"
      }
    }
  }
}
```

##### Cursor

Put this in `.cursor/mcp.json` of the project, or in `~/.cursor/mcp.json` for every project:

```json
{
  "mcpServers": {
    "slotpreview": {
      "url": "${env:SLOTPREVIEW_BASE_URL}/mcp",
      "headers": {
        "Authorization": "Bearer ${env:SLOTPREVIEW_API_KEY}"
      }
    }
  }
}
```

##### OpenAI Responses API

The server is a remote MCP tool; the key goes in `headers`:

```python
import os
from openai import OpenAI

client = OpenAI()
response = client.responses.create(
    model="gpt-5.6-sol",
    tools=[{
        "type": "mcp",
        "server_label": "slotpreview",
        "server_url": os.environ["SLOTPREVIEW_BASE_URL"] + "/mcp",
        "headers": {"Authorization": "Bearer " + os.environ["SLOTPREVIEW_API_KEY"]},
        "require_approval": "always",
    }],
    input="Find the game Vault of Ember and price a tile for it.",
)
```

`require_approval` decides whether your code confirms each tool call before it runs. The protection of your balance described below does not depend on it.

##### Claude API

The MCP connector of the Messages API sends `authorization_token` as the bearer key:

```python
import os
import anthropic

client = anthropic.Anthropic()
response = client.beta.messages.create(
    model="claude-sonnet-5-5",
    max_tokens=2048,
    messages=[{"role": "user", "content": "Find the game Vault of Ember and price a tile for it."}],
    mcp_servers=[{
        "type": "url",
        "name": "slotpreview",
        "url": os.environ["SLOTPREVIEW_BASE_URL"] + "/mcp",
        "authorization_token": os.environ["SLOTPREVIEW_API_KEY"],
    }],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "slotpreview"}],
    betas=["mcp-client-2025-11-20"],
)
```

Here `max_tokens` is the length limit of the model reply, not a price.

#### Estimate first, then spend

Two tools spend tokens: `generate_tile` costs 1 for two variants and `export_tile` costs 2 for the format set, the same as the API. Neither takes parameters of its own. Each takes only a `quote_id` from its estimate tool, `estimate_tile` or `estimate_export`, so a paid call can only do what the estimate named, at the price it named:

- a quote is valid for 15 minutes and belongs to the key or connection it was issued to; an expired, altered or foreign quote is refused without a call to the API;
- on a live key or connection the paid call sends the quoted price as `max_tokens`, so if the price went up in between, it is refused with `price_above_limit` and nothing is charged: estimate again;
- calling a paid tool again with the same quote, after a dropped connection or from two places at once, returns the same job and charges once; a new generation needs a new estimate;
- a key or connection makes at most 6 paid calls and 120 other calls a minute through the server; above that the tool answers `rate_limited` with the seconds to wait;
- the daily cap of the key or connection stops spending at the cap, whatever the agent does.

When the balance is short, the estimate says how many tokens are needed and how many are available, and gives no quote. The server cannot buy tokens: on a short balance the agent is told to top up in the account, with the direct `top_up_url` when the API returns one, and a person tops up there.

#### Captions

A lobby tile carries a caption: the game title and the studio name. `estimate_export` takes `caption` and draws the tile with it for free:

- `"game_title"`, the default: the game title and the studio name;
- `"none"`: the art without a caption;
- your own title, up to 40 characters, as `{"title": "Vault of Ember", "show_studio": true}`.

The estimate returns the price, the quote and the tile as an image, and the files that `export_tile` builds with that quote carry exactly the caption in that image. Drawing a caption needs the `render` scope: a key without it can estimate and export with `"none"`, and gets `forbidden_scope` naming `render` for any other caption. Exporting a variant that is already exported is free, and so is changing the caption of an exported set: estimate again with the new caption and export.

By default the set is built in one file container. Pass `all_file_formats: true` to `estimate_export` to build it in all four (WebP, PNG, JPEG, AVIF) as one purchase, one folder per container in the ZIP. The estimate names the price of that choice, and on a set that is already exported it is the price of adding the missing containers.

#### Tools

Every key and connection sees the same twelve tools in the same order; a tool whose scope the credential lacks answers with the name of the missing scope. Every tool is marked for the client: read-only or not, and destructive or not.

| Tool | Title | Read-only | Tokens | What it does |
| --- | --- | --- | --- | --- |
| `check_connection` | Check connection | yes | none | Names the organization, the environment (live or sandbox), the scopes, the tokens available and the daily cap of the key or connection. Call it first. |
| `search_games` | Search games | yes | none | Finds games in the catalogue by name, up to 20 a page; an empty query gives the first page. |
| `add_game` | Add a game | no | none | Registers a game that is not in the catalogue yet, for example before its release. Adding the same title and studio again returns the same game. |
| `list_art_options` | List art options | yes | none | Lists the styles, backgrounds and compositions that the tile estimate accepts. |
| `estimate_tile` | Estimate a tile | yes | none | Prices one generation of two variants and returns a `quote_id` valid for 15 minutes. |
| `generate_tile` | Generate a tile | no | 1 | Starts the generation of the quote and returns a job id at once. |
| `get_generation` | Get a generation | yes | none | Checks a generation; when it is done, returns the ready variants as images with links to the full-size files. |
| `cancel_generation` | Cancel a generation | no, destructive | none | Cancels a queued or running generation and says whether the held tokens came back. |
| `estimate_export` | Estimate an export | yes | none | Prices the export of a finished variant with the chosen caption, shows the tile and returns a `quote_id`. |
| `export_tile` | Export a tile | no | 2, none for a variant already exported | Builds the format set of the quote and returns an export job id at once. |
| `get_export` | Get an export | yes | none | Checks an export; when it is done, returns a link to every file and to the ZIP archive, the formats that are missing and one preview of the tile. |
| `list_my_tiles` | List my tiles | yes | none | Lists the tiles your organization already generated, newest first, so the agent can reuse one instead of paying again. |

Game titles and descriptions from the catalogue reach the agent as data, not as instructions, and they cannot change what a call spends.

#### Images, links and long jobs

A paid tool returns at once with a job id. `get_generation` and `get_export` wait for the job themselves, and no call takes longer than 25 seconds: if the job is still running, the tool says so and when to ask again. A finished generation comes back as two images of at most 600 pixels on the long side, an export as links and one preview image, so the files do not fill the context of the model. Links to images and files expire after one hour; calling the tool again gives fresh links for free.

#### Errors

A refusal comes back as a tool error that begins with its code, then a reason in English and whether a retry can help, and the model sees it. The codes are those of the API, see [Errors and limits](https://slotspreviewai.com/docs/errors), plus the refusals of a quote: `quote_expired`, `quote_bad_signature`, `quote_other_credential`, `quote_wrong_kind` and `quote_malformed` all mean that the paid tool needs a fresh estimate, and nothing was charged. A frozen organization gets `spending_frozen` with advice to write to support and no top-up link. An agent at its daily cap gets `daily_token_cap_exceeded`, and the refusal says whether the cap is that of a key or of a connection.

#### What the server does not do

- No batches: one call never starts more than one generation or one export.
- No webhooks: the agent asks with `get_generation` and `get_export`.
- No layered export with a separate foreground layer.
- No buying tokens: a person tops up in the account.
- No changing the cap of a connected agent: disconnect it and connect again with a new cap.
- No test scenarios: the `X-Test-Scenario` header of the sandbox does not pass through the server.
