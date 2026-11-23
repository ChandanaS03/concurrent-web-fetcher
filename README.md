# concurrent-web-fetcher

A typed, tested command-line tool that fetches many pages at once, retries failures, respects
robots.txt and per-host rate limits, and saves clean text as JSONL: the first stage of a RAG or
data pipeline.

Built by following [sreemantidey's "Build a production-grade web fetcher in Python"](https://github.com/alexriderspy/concurrent-web-fetcher)
tutorial, as a learning project.

## Quickstart

```bash
uv sync
uv run python scripts/make_urls.py                 # 266 Python standard library pages -> data/urls.txt
uv run fetcher fetch data/urls.txt -o data/pages.jsonl   # about 27 s at the default 10 requests/s
uv run fetcher stats data/pages.jsonl
uv run python scripts/to_chunks.py data/pages.jsonl # overlapping chunks, ready to embed
```

There's also `data/agri_urls.txt`: a curated list of agriculture-topic pages (Wikipedia, FAO,
USDA-ERS, EPA) for fetching real agricultural information instead of Python docs:

```bash
uv run fetcher fetch data/agri_urls.txt -o data/agri_pages.jsonl --rps 2 --concurrency 6
uv run fetcher stats data/agri_pages.jsonl
```

Set your own user agent (with a way to contact you) before crawling anything that isn't yours:

```bash
cp .env.example .env    # edit it
set -a; source .env; set +a
```

## What's inside

```
src/fetcher/
  models.py     Page and Status: what one fetched page looks like (Pydantic)
  config.py     FetchConfig: every knob in one frozen dataclass
  fetch.py      Fetcher: asyncio + httpx, semaphore, retries, rate limit, robots.txt
  retry.py      which failures to retry, exponential backoff with full jitter, Retry-After
  ratelimit.py  per-host rate limiter
  robots.py     robots.txt cache (RFC 9309 rules for 4xx and 5xx)
  clean.py      HTML -> readable text
  log.py        JSON logs that print above the progress bar
  output.py     JSONL writer and reader
  cli.py        Typer CLI: `fetcher fetch` and `fetcher stats`
scripts/        demos for each chapter, the flaky test server, the benchmark, RAG chunking
tests/          pytest with httpx.MockTransport (no real network)
```

## Options

```
fetcher fetch URLS_FILE [-o pages.jsonl] [--concurrency 10] [--rps 10] [--timeout 10]
                        [--retries 3] [--workers 4] [--no-robots] [--user-agent ...]
                        [--log-level info]
```

`--concurrency` caps requests in flight across all hosts; `--rps` caps requests per second to any
single host (0 turns it off); `--workers` is how many processes clean HTML (0 = one thread).

## Checks

```bash
uv run ruff check . && uv run ruff format --check .
uv run mypy
uv run pytest -v
```

The same three run in GitHub Actions on every push (`.github/workflows/ci.yml`).

## Retries and timeouts, locally

`scripts/flaky_server.py` starts a server that fails on purpose (503s, a 429 with Retry-After, a
404, a page slower than the timeout, and a robots.txt rule), so you can watch the retry logic work:

```bash
uv run python scripts/flaky_server.py 8765 &
uv run python scripts/demo_fetch.py --timeout 2 http://127.0.0.1:8765/flaky/1 http://127.0.0.1:8765/slow
```
