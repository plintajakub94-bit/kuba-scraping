---
name: run-scrapegraphai
description: Build, install, and run ScrapeGraphAI from source in this repo. Use when asked to set up the project, verify it works, run the test suite, or drive a scraping graph (including the Chromium/Playwright fetch step) inside the Claude Code remote environment.
---

ScrapeGraphAI is a Python library — "running" it means installing from
source, exercising the public graph API, and driving a real
`SmartScraperGraph` (fetch a page with Chromium, then have an LLM extract
data). This skill captures the exact steps that work in the Claude Code
remote container, including two environment-specific patches the browser
fetch needs.

## Setup

The project ships a `uv.lock`, so use `uv` (Python 3.11):

```bash
uv sync
```

This creates a local `.venv` (git-ignored) with ~137 packages and installs
`scrapegraphai` in editable mode.

## Verify (install is usable)

```bash
uv run python -c "from scrapegraphai.graphs import SmartScraperGraph; print('OK')"
# → OK
```

## Test

Run the offline suite (integration tests need network + API keys, so
deselect them):

```bash
uv run pytest -m "not integration" -p no:cacheprovider
```

Coverage is enabled by `pytest.ini` and writes `htmlcov/` + `coverage.xml`
(both git-ignored). At import time the suite collects ~303 tests
(17 integration tests deselected).

## Run the app (drive a real graph)

A full `SmartScraperGraph.run()` does **Fetch (Chromium) → ParseNode →
GenerateAnswer (LLM)**. Two things about this environment:

1. **Chromium build mismatch.** The pinned `playwright` (1.57.0) expects a
   browser build that differs from the pre-installed one. Do **NOT** run
   `playwright install`. Point at the pre-installed binary instead:
   `/opt/pw-browsers/chromium-1194/chrome-linux/chrome`
   (forwarded into `chromium.launch(**browser_config)` via the loader's
   kwargs).
2. **Proxy.** Chromium must route through the agent proxy. Pass
   `proxy={"server": "$HTTPS_PROXY"}` (typically `http://127.0.0.1:<port>`,
   see `echo $HTTPS_PROXY`).

### No-LLM smoke (always works here)

Exercises the public API + a real HTML utility, no network or key needed:

```bash
uv run python - <<'PY'
from scrapegraphai.graphs import SmartScraperGraph
from scrapegraphai.utils import cleanup_html

sg = SmartScraperGraph(
    prompt="Extract the page title",
    source="https://example.com",
    config={"llm": {"api_key": "sk-PLACEHOLDER", "model": "openai/gpt-4o-mini"},
            "headless": True},
)
sg.graph = sg._create_graph()
print("pipeline:", [n.node_name for n in sg.graph.nodes])  # Fetch, ParseNode, GenerateAnswer

title, body, links, images, scripts = cleanup_html(
    "<html><head><title>Hi</title></head><body><h1>Hello</h1>"
    "<a href='/p'>link</a></body></html>", "https://example.com")
print("title:", title, "| links:", links)
PY
```

### Real Chromium fetch (needs the two patches above)

```bash
uv run python - <<'PY'
import os
from scrapegraphai.docloaders.chromium import ChromiumLoader

loader = ChromiumLoader(
    urls=["https://example.com"],
    backend="playwright",
    headless=True,
    executable_path="/opt/pw-browsers/chromium-1194/chrome-linux/chrome",
    proxy={"server": os.environ["HTTPS_PROXY"]},
)
docs = loader.load()
print("len:", len(docs[0].page_content))
PY
```

## Known environment limits (not code bugs)

A full end-to-end `.run()` cannot complete in the default remote
environment because:

- **Network egress policy** denies outbound CONNECT to arbitrary sites
  (the proxy returns `403`; check
  `curl -sS "$HTTPS_PROXY/__agentproxy/status"`). So Chromium launches but
  page navigation to external sites fails with
  `net::ERR_TUNNEL_CONNECTION_FAILED`.
- **No LLM key** — the `GenerateAnswer` node needs e.g. `OPENAI_API_KEY`.

To scrape for real you need an environment with web egress enabled and an
LLM provider key set.
