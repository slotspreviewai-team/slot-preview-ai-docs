# Slot Preview AI

Slot Preview AI generates lobby tiles (game thumbnails) for online casino slot games with AI. One master image is exported into 6 aspect ratios and 13 PNG files. Tiles are made for the buyer who generated them and are never re-issued to another buyer. Prices are in tokens, bought with USDT.

This repository holds the public documentation of the service. There is no source code here: the product runs as a hosted service, and you use it through its REST API or its MCP server.

- Website: https://slotspreviewai.com
- API reference: https://slotspreviewai.com/docs/api
- OpenAPI spec: https://api.slotspreviewai.com/v1/openapi.json
- Plain-text summary for AI tools: https://slotspreviewai.com/llms.txt

## Who it is for

iGaming operators, casino lobby and content teams, and platforms that need a tile for every game in several sizes, including games that are announced but not yet released. The product is self-service: no brief, no revision round, no queue.

## What you can do

- Generate a master image for a game from a style and a background. A pair of variants costs 1 token.
- Edit the game title and studio name as a live text layer. Editing it costs nothing.
- Export one set for 2 tokens: 13 PNG files in 6 aspect ratios (2:3, 3:4, 1:1, 9:16, 16:9, 21:9), a ZIP archive and a manifest of the overlay settings.
- Make a tile before release, from a public announcement.
- Run a whole catalogue from a CSV, one export set per game.

## Two ways to connect

**REST API.** Async jobs, webhooks, idempotent retries, a free sandbox key that spends no tokens. Start with the [quickstart](docs/quickstart.md): the first tile in five steps, every response recorded from the live API. Keys, scopes and the sandbox are in [authentication](docs/authentication.md). Every status and error code is in [errors and limits](docs/errors.md).

**MCP server.** A remote server over Streamable HTTP, nothing to install. Address: the API address with the path /mcp, that is https://api.slotspreviewai.com/mcp, without a trailing slash. ChatGPT and claude.ai connect with sign-in. Claude Code, Cursor, the OpenAI Responses API and the Claude API connect with an API key in the Authorization: Bearer header. The agent shows the price of each paid step before it spends, within a daily token cap you set. Details: [MCP server](docs/mcp.md).

## Documentation

| File | What it covers |
|---|---|
| [docs/overview.md](docs/overview.md) | The service and your first call |
| [docs/quickstart.md](docs/quickstart.md) | Register a game, generate a master, export, download |
| [docs/authentication.md](docs/authentication.md) | API keys, environments, scopes, sandbox test scenarios |
| [docs/mcp.md](docs/mcp.md) | Connect an AI agent, the twelve tools, what they cost |
| [docs/errors.md](docs/errors.md) | The error envelope, every code, what to retry |

The pages on https://slotspreviewai.com/docs are the canonical version. These files are copies for people who read documentation on GitHub.

## Contact

hello@slotspreviewai.com
