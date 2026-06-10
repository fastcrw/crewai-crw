# crewai-crw

[![PyPI version](https://img.shields.io/pypi/v/crewai-crw)](https://pypi.org/project/crewai-crw/)
[![Python](https://img.shields.io/pypi/pyversions/crewai-crw)](https://pypi.org/project/crewai-crw/)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)

CRW web scraping tools for [CrewAI](https://github.com/crewAIInc/crewAI) — scrape, crawl, map, and search the web with AI agents.

[CRW](https://github.com/us/crw) is an open-source web scraper built for AI agents. Single Rust binary, ~6 MB idle RAM, Firecrawl-compatible API.

## Installation

```bash
pip install crewai crewai-crw
# or
uv add crewai crewai-crw
```

**Requirements:** Python 3.11–3.13. The upper bound currently tracks `crewai`'s own `python = ">=3.10,<3.14"` constraint — on Python 3.14 the dependency resolver fails at `crewai` resolution, not `crewai-crw`.

## Quick Start — Cloud (default)

CRW is cloud-first. [Sign up at fastcrw.com](https://fastcrw.com/dashboard) for **500 free credits** — no payment, no monthly reset (GitHub/Google, ~10s) — then set `CRW_API_KEY`:

```python
from crewai_crw import CrwScrapeWebsiteTool

# Uses the managed cloud (api.fastcrw.com) — reads CRW_API_KEY from the env
scrape_tool = CrwScrapeWebsiteTool()

# ...or pass the key explicitly
scrape_tool = CrwScrapeWebsiteTool(api_key="crw_live_...")
```

## Self-hosting

Prefer to run the engine yourself? Two options.

**Local zero-config engine** — set `CRW_LOCAL=1` (no server, no key; the `crw` SDK manages the binary):

```python
# CRW_LOCAL=1 in the environment
scrape_tool = CrwScrapeWebsiteTool()
```

**Persistent server** (shared across services):

```bash
curl -fsSL https://raw.githubusercontent.com/us/crw/main/install.sh | sh
crw serve  # http://localhost:3000
# or: docker run -d -p 3000:3000 ghcr.io/us/crw:latest
```

```python
scrape_tool = CrwScrapeWebsiteTool(api_url="http://localhost:3000")
```

## Tools

| Tool | Description |
|------|-------------|
| `CrwScrapeWebsiteTool` | Scrape a single URL and get clean markdown |
| `CrwCrawlWebsiteTool` | BFS crawl a website, collect content from multiple pages |
| `CrwMapWebsiteTool` | Discover all URLs on a website |
| `CrwSearchWebTool` | Search the web and get results (cloud only) |

## CrewAI Example

```python
from crewai import Agent, Task, Crew
from crewai_crw import CrwScrapeWebsiteTool

# Zero config — just works out of the box
scrape_tool = CrwScrapeWebsiteTool()

researcher = Agent(
    role="Web Researcher",
    goal="Research and summarize information from websites",
    backstory="Expert at extracting key information from web pages",
    tools=[scrape_tool],
)

task = Task(
    description="Scrape https://example.com and summarize the content",
    expected_output="A summary of the page content",
    agent=researcher,
)

crew = Crew(agents=[researcher], tasks=[task])
result = crew.kickoff()
```

### Crawl an entire site

```python
from crewai_crw import CrwCrawlWebsiteTool

crawl_tool = CrwCrawlWebsiteTool(
    config={
        "maxDepth": 3,
        "maxPages": 50,
        "formats": ["markdown"],
        "onlyMainContent": True,
    }
)

# Use in an agent
researcher = Agent(
    role="Deep Researcher",
    goal="Crawl documentation sites and extract comprehensive information",
    backstory="Expert at gathering information across multiple pages",
    tools=[crawl_tool],
)
```

### Discover all URLs on a site

```python
from crewai_crw import CrwMapWebsiteTool

map_tool = CrwMapWebsiteTool()

# Use in an agent
mapper = Agent(
    role="Site Mapper",
    goal="Discover and catalog all pages on a website",
    backstory="Expert at understanding website structure",
    tools=[map_tool],
)
```

### Search the web (Cloud Only)

> **Note:** Web search is a cloud-only feature. It requires `api_url` pointing to a CRW cloud instance (e.g. fastcrw.com). Subprocess mode is not supported for search.

```python
from crewai_crw import CrwSearchWebTool

search_tool = CrwSearchWebTool(
    api_url="https://fastcrw.com/api",
    api_key="YOUR_KEY",
)

# Use in an agent
researcher = Agent(
    role="Web Researcher",
    goal="Find the latest information on any topic",
    backstory="Expert at searching the web for relevant information",
    tools=[search_tool],
)
```

## Configuration

### Constructor Arguments

| Argument | Type | Default | Description |
|----------|------|---------|-------------|
| `api_url` | `str \| None` | `None` | CRW server URL. If unset, uses subprocess mode (no server needed) |
| `api_key` | `str \| None` | `None` | API key (required for fastcrw.com) |
| `config` | `dict` | varies per tool | Tool-specific configuration |

### Environment Variables

Pass `api_url` and `api_key` explicitly when creating tools:

```bash
export CRW_API_URL=https://fastcrw.com/api  # or http://localhost:3000
export CRW_API_KEY=your_api_key              # required for cloud, optional for self-hosted
```

```python
import os

tool = CrwScrapeWebsiteTool(
    api_url=os.getenv("CRW_API_URL"),
    api_key=os.getenv("CRW_API_KEY"),
)
```

### Scrape Config

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `formats` | `list[str]` | `["markdown"]` | Output formats: markdown, html, rawHtml, plainText, links, json |
| `onlyMainContent` | `bool` | `true` | Strip nav/footer/sidebar |
| `renderJs` | `bool\|null` | `null` | null=auto, true=force JS, false=HTTP only |
| `waitFor` | `int` | — | ms to wait after JS rendering |
| `includeTags` | `list[str]` | `[]` | CSS selectors to include |
| `excludeTags` | `list[str]` | `[]` | CSS selectors to exclude |

### Crawl Config

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `maxDepth` | `int` | `2` | Maximum link-follow depth |
| `maxPages` | `int` | `10` | Maximum pages to scrape |
| `formats` | `list[str]` | `["markdown"]` | Output formats per page |
| `onlyMainContent` | `bool` | `true` | Strip boilerplate |

### Map Config

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `maxDepth` | `int` | `2` | Maximum discovery depth |
| `useSitemap` | `bool` | `true` | Also read sitemap.xml |

### Search Config (Cloud Only)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `limit` | `int` | `5` | Maximum number of results to return |

## Compared to Firecrawl Tools

| Feature | crewai-crw | Firecrawl Tools |
|---------|-----------|-----------------|
| Requires SDK package | No (uses `crw` SDK, auto-manages binary) | Yes (`firecrawl-py`) |
| Requires API key | No (subprocess or self-hosted) | Yes (always) |
| Server required | No (`pip install` is all you need) | Yes (always) |
| Self-hosted option | Yes (single binary, auto-managed) | Complex (5+ containers) |
| Cloud option | Yes (fastcrw.com) | Yes (firecrawl.dev) |
| Idle RAM | ~6 MB | ~500 MB+ |

## License

MIT
