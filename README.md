# ExploreClinicalTrials-MCP

An MCP server that answers clinical-trial questions with cited charts and trial lists from ClinicalTrials.gov.

Ask in plain words, from any MCP client (Claude, Codex, Cursor, VS Code, ...):

> *How many phase 3 Alzheimer's trials started each year since 2015?*
> *Which sponsors run the most semaglutide trials?*
> *Compare phases for Ozempic vs Mounjaro trials.*
> *Recruiting melanoma trials in Texas for a 60-year-old.*

It is built for people who need registry numbers they can defend: competitive-intelligence and
business-development analysts, clinical operations, consultants, investors and researchers.

## Why it is useful

The client's model reads the question, plans the search and picks the chart. **Code does everything
that can be checked**: it fetches every matching trial from the
[ClinicalTrials.gov API v2](https://clinicaltrials.gov/data-api/api), validates that the chart fits the
data, counts distinct trials and attaches the source trials to every value. The model never produces
a number, so it cannot invent one. The server needs no API key.

Code also corrects for registry quirks. A search for a brand name such as "Ozempic" misses most
trials, because they are indexed under the generic name (semaglutide). The server resolves names to
the registry's own terms, searches both, and says so in the chart notes.

## Features

- **Charts with citations**: bar, grouped bar, stacked bar, time series, scatter, histogram and
  network charts (Vega-Lite), each value linking to the NCT IDs behind it. Counts are of distinct trials.
- **`find_trials`**: filter by condition, intervention, free terms, sponsor, site location,
  recruitment status, phase, age and sex. Returns the exact total and one page of trials; with a
  location, each trial lists its sites there.
- **`get_trial`**: one record by NCT number (arms, outcome measures, eligibility, contacts, sites).
  It reports whether results are posted, not the results.
- **Saved answers**: every chart is stored as JSON under `~/.exploreclinicaltrials/results/` and can
  be reopened with `exploreclinicaltrials-mcp serve`; each chart offers CSV downloads of its data and
  of its trial list, and PNG/SVG export.
- **Inline or browser display**: clients that support
  [MCP Apps](https://modelcontextprotocol.io/extensions/apps) (e.g. Claude Desktop) show the chart in
  the conversation; others get a link to a local page.
- **Optional web service**: a FastAPI app (page + REST API) that runs the planning itself with
  Pydantic AI agents and an OpenAI key.

## Tech stack

Python 3.11+, [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk) (`mcp>=2.0`),
httpx, Pydantic v2, FastAPI + uvicorn, Pydantic AI (web service only), Vega-Lite (vendored in
`src/exploreclinicaltrials/web/vendor/`), managed with [uv](https://docs.astral.sh/uv/). Tests use pytest.

## Quickstart

You need [uv](https://docs.astral.sh/uv/) (`brew install uv` or
`curl -LsSf https://astral.sh/uv/install.sh | sh`). The package is not published on PyPI, so run it
from GitHub:

```bash
uvx --from git+https://github.com/pdwytr/ExploreClinicalTrials-MCP exploreclinicaltrials-mcp --help
```

Or from a clone:

```bash
git clone https://github.com/pdwytr/ExploreClinicalTrials-MCP.git
cd ExploreClinicalTrials-MCP
uv sync
uv run pytest                       # offline, no API key
uv run exploreclinicaltrials-mcp    # starts the MCP server on stdio
```

### Add it to a client

The server command is the same everywhere:
`uvx --from git+https://github.com/pdwytr/ExploreClinicalTrials-MCP exploreclinicaltrials-mcp`

**Claude Code**

```bash
claude mcp add -s user exploreclinicaltrials -- uvx --from git+https://github.com/pdwytr/ExploreClinicalTrials-MCP exploreclinicaltrials-mcp
```

Or install the plugin, which also adds a `clinical-trials` subagent that answers in its own context:

```
/plugin marketplace add pdwytr/ExploreClinicalTrials-MCP
/plugin install exploreclinicaltrials@exploreclinicaltrials
```

**Claude Desktop** (Settings -> Developer -> Edit Config, then restart; use the full path from
`which uvx` if Desktop cannot find it):

```json
{
  "mcpServers": {
    "exploreclinicaltrials": {
      "command": "uvx",
      "args": ["--from", "git+https://github.com/pdwytr/ExploreClinicalTrials-MCP", "exploreclinicaltrials-mcp"]
    }
  }
}
```

**Cursor**: the same JSON in `~/.cursor/mcp.json`.

**Codex** (`~/.codex/config.toml`; large questions can take a minute, hence the longer timeout):

```toml
[mcp_servers.exploreclinicaltrials]
command = "uvx"
args = ["--from", "git+https://github.com/pdwytr/ExploreClinicalTrials-MCP", "exploreclinicaltrials-mcp"]
tool_timeout_sec = 180
```

**VS Code (Copilot agent mode)**

```bash
code --add-mcp '{"name":"exploreclinicaltrials","command":"uvx","args":["--from","git+https://github.com/pdwytr/ExploreClinicalTrials-MCP","exploreclinicaltrials-mcp"]}'
```

**Any other client**: add a stdio server with that command.

## Usage

Ask a question in plain words. If the client answers from its own knowledge instead, say
*"use exploreclinicaltrials to ..."*.

| Tool | Purpose |
|---|---|
| `start_query` | Opens a question and returns the procedure the model follows |
| `fetch_data` | Runs a search plan (up to 5 fetches per query) |
| `describe_data` | Profiles the fetched trials so the model can pick a chart |
| `build_chart` | Validates the chart plan, counts, cites and saves the answer |
| `read_api_docs`, `read_reference` | Bundled ClinicalTrials.gov API and chart-planning references |
| `find_trials` | Lists matching trials with the exact total |
| `get_trial` | Reads one trial by NCT number |

Every tool is marked read-only except `build_chart`, which writes a new file to the results folder
and never changes or deletes an existing one. Query sessions live in server memory for 2 hours.
Neither `find_trials` nor `get_trial` says whether someone qualifies for a trial or whether a
treatment works.

Reopen saved answers without a client:

```bash
uv run exploreclinicaltrials-mcp serve --port 8765   # http://127.0.0.1:8765/
```

### Limitations

- Investigator and trial-site questions, and breakdowns by US state or city, are refused with the reason.
- Sponsor aliases are not matched ("MSD" is not "Merck Sharp & Dohme LLC"); use the registered name.
- In networks, drug names appear as sponsors typed them, so one drug can show under several names.
- Drug classes ("GLP-1 agonists") have no registry field; the client's model expands them from its own knowledge.

## Configuration

| Variable | Used by | Default | Meaning |
|---|---|---|---|
| `EXPLORECLINICALTRIALS_HOME` | MCP server | `~/.exploreclinicaltrials` | Folder for saved answers (`results/`) |
| `EXPLORECLINICALTRIALS_PORT` | MCP server | `8765` | Port of the local chart page |
| `OPENAI_API_KEY` | web service only | required | OpenAI key for the planning agents |
| `EXPLORECLINICALTRIALS_PLANNER_MODEL` | web service only | required | Model string, `openai:<model>` (e.g. `openai:gpt-5.4`) |

The web-service variables can go in a `.env` file at the repo root (copy `.env.example`; `.env` is gitignored).

## Web service (optional)

```bash
cp .env.example .env     # set OPENAI_API_KEY and EXPLORECLINICALTRIALS_PLANNER_MODEL
uv sync
uv run uvicorn exploreclinicaltrials.api.app:app --port 8765
```

`/` is the question page, `/docs` the OpenAPI UI, `GET /health` a liveness check. Submit a question
with `POST /queries` and poll `GET /queries/{task_id}`. A `Dockerfile` and `fly.toml` are included for deployment.

## Development

```bash
uv run pytest                                          # unit tests, no API key needed
EXPLORECLINICALTRIALS_LAUNCHER=$PWD/scripts/dev-mcp claude --plugin-dir ./plugin   # plugin on this checkout's source
uv run python scripts/vendor_web_assets.py             # refresh the pinned chart libraries
```

| Path | What it is |
|---|---|
| `plugin/` | Claude Code plugin: MCP config and the `clinical-trials` subagent |
| `src/exploreclinicaltrials/mcp_server/` | MCP server: tools, query sessions, saved results, local page |
| `src/exploreclinicaltrials/agents/` | Agent specs (shared by the plugin and the web service) |
| `src/exploreclinicaltrials/workflow/` | Search plans, fetching, profiling, chart checks |
| `src/exploreclinicaltrials/fetch/` | ClinicalTrials.gov client and models |
| `src/exploreclinicaltrials/visualize/` | Counts, citations and the Vega-Lite chart |
| `src/exploreclinicaltrials/api/` | Optional FastAPI web service |
| `evals/` | End-to-end cases against the live registry (need an OpenAI key) |

CI runs the test suite on every push and pull request (`.github/workflows/ci.yml`).

## License

[MIT](LICENSE)

Data comes from ClinicalTrials.gov. This project is not affiliated with the NIH or the NLM.
