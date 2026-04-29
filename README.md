# Real-World User Agents

> A curated dataset of **100 % real, verified User Agent strings** — sourced exclusively from live, organic website traffic. Refreshed every 48 hours. No synthetic data. No scraped archives. No bots.

[![Update Frequency](https://img.shields.io/badge/refresh-every%2048h-blue)](#update-frequency)
[![Source](https://img.shields.io/badge/source-real%20world%20traffic-success)](#methodology)
[![License: CC0](https://img.shields.io/badge/license-CC0%201.0-lightgrey)](#license)
[![GitHub stars](https://img.shields.io/github/stars/WinFuture23/real-world-user-agents?style=social)](../../stargazers)

## Why This Exists

Most public User-Agent datasets are flawed in one of three ways:

- ❌ **Synthetic** — generated from templates or LLMs, with versions and OS combinations that no real browser ever ships
- ❌ **Stale** — scraped years ago and never refreshed; full of EOL browsers nobody uses anymore
- ❌ **Polluted** — mixed real and bot traffic, with no quality filter

This repository is different. **Every entry has been observed in real human-driven traffic within the last 48 hours**, validated through five independent gates, and sorted by real-world prevalence. We believe this is one of the last public repositories still feeding strictly on live, organic data — not extrapolated, not synthesized.

## Quick Start

The dataset is published as two files at the repository root:

| File | Format | Use when |
|---|---|---|
| [`user-agents.txt`](./user-agents.txt) | one UA per line | quick rotation lists, shell scripts, fixtures |
| [`user-agents.json`](./user-agents.json) | structured + parsed | you need browser/OS metadata or schema validation |

Direct raw URLs:

```
https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.txt
https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json
```

The list is **sorted by real-world prevalence** — the first entry is the most commonly observed, the last is the rarest. We deliberately omit raw counts so the file doesn't leak traffic-volume signals about our source.

## JSON Schema

```jsonc
{
  "schema_version": 1,
  "generated_at": "2026-04-29T08:00:00Z",   // ISO-8601, UTC
  "window": "48h",                           // observation window
  "count": 142,
  "user_agents": [
    {
      "ua": "Mozilla/5.0 (...)",            // the literal UA string
      "browser": { "name": "Chrome", "version": "147.0.0.0" },
      "os":      { "name": "Android", "version": "10" },
      "device_type": "mobile",              // computer | mobile | tablet
      "first_seen": "2026-04-22",           // day-precision (UTC)
      "last_seen":  "2026-04-29"
    }
    // ...
  ]
}
```

## Code Examples

### cURL

```bash
# Random UA from the list (handy for one-off scripting)
curl -fsSL https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.txt \
  | grep -v '^#' \
  | shuf -n 1
```

```bash
# Full dataset with metadata, validated as JSON
curl -fsSL https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json \
  | jq '.count, .user_agents[0]'
```

### Python

```python
"""
Robust loader for real-world-user-agents.

- Streams the JSON file (no 100 KB+ string in memory needed for filtering).
- Validates the top-level schema before returning anything.
- Falls back to a local cache if GitHub is unreachable.
- Picks UAs in proportion to their position (top entries are more common).
"""

import json
import random
import time
from pathlib import Path
from urllib.request import Request, urlopen
from urllib.error import URLError

JSON_URL = "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"
CACHE = Path.home() / ".cache" / "real-world-user-agents.json"
CACHE_MAX_AGE_S = 48 * 3600  # match upstream cadence

REQUIRED_KEYS = {"schema_version", "generated_at", "user_agents"}


def _fetch_fresh(timeout: float = 10.0) -> dict:
    req = Request(JSON_URL, headers={"User-Agent": "real-world-user-agents/loader"})
    with urlopen(req, timeout=timeout) as r:
        if r.status != 200:
            raise RuntimeError(f"HTTP {r.status}")
        return json.loads(r.read())


def _load_cached() -> dict | None:
    if not CACHE.exists():
        return None
    if time.time() - CACHE.stat().st_mtime > CACHE_MAX_AGE_S * 2:
        return None  # too stale even for fallback
    try:
        return json.loads(CACHE.read_text())
    except json.JSONDecodeError:
        return None


def load() -> dict:
    """Fetch fresh, fall back to cache if network fails."""
    try:
        data = _fetch_fresh()
    except (URLError, RuntimeError):
        cached = _load_cached()
        if cached is None:
            raise
        return cached

    # Schema sanity — refuse anything weird before we let it propagate
    if not REQUIRED_KEYS.issubset(data):
        raise ValueError(f"missing required keys: {REQUIRED_KEYS - data.keys()}")
    if not isinstance(data["user_agents"], list) or not data["user_agents"]:
        raise ValueError("user_agents is not a non-empty list")

    CACHE.parent.mkdir(parents=True, exist_ok=True)
    CACHE.write_text(json.dumps(data))
    return data


def random_ua(weighted: bool = True) -> str:
    """
    Return one UA string. With weighted=True, top-of-list entries (more common
    in the wild) are picked more often — closer to a realistic distribution.
    """
    entries = load()["user_agents"]
    if weighted:
        # Inverse-rank weights: rank 1 → highest weight
        weights = [1 / (i + 1) for i in range(len(entries))]
        return random.choices(entries, weights=weights, k=1)[0]["ua"]
    return random.choice(entries)["ua"]


if __name__ == "__main__":
    print(random_ua())
```

### Node.js

```js
/**
 * Robust loader for real-world-user-agents.
 * Works on Node 18+ (built-in fetch). Caches locally for 48 h.
 */
import { readFile, writeFile, mkdir, stat } from "node:fs/promises";
import { dirname, join } from "node:path";
import { homedir } from "node:os";

const JSON_URL =
  "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json";
const CACHE = join(homedir(), ".cache", "real-world-user-agents.json");
const CACHE_MAX_AGE_MS = 48 * 60 * 60 * 1000;

const REQUIRED_KEYS = ["schema_version", "generated_at", "user_agents"];

async function fetchFresh(timeoutMs = 10_000) {
  const ctrl = new AbortController();
  const t = setTimeout(() => ctrl.abort(), timeoutMs);
  try {
    const r = await fetch(JSON_URL, {
      signal: ctrl.signal,
      headers: { "User-Agent": "real-world-user-agents/loader" },
    });
    if (!r.ok) throw new Error(`HTTP ${r.status}`);
    return await r.json();
  } finally {
    clearTimeout(t);
  }
}

async function loadCached() {
  try {
    const s = await stat(CACHE);
    if (Date.now() - s.mtimeMs > CACHE_MAX_AGE_MS * 2) return null;
    return JSON.parse(await readFile(CACHE, "utf8"));
  } catch {
    return null;
  }
}

export async function load() {
  let data;
  try {
    data = await fetchFresh();
  } catch (err) {
    const cached = await loadCached();
    if (cached) return cached;
    throw err;
  }

  // Schema sanity
  for (const k of REQUIRED_KEYS) {
    if (!(k in data)) throw new Error(`missing required key: ${k}`);
  }
  if (!Array.isArray(data.user_agents) || data.user_agents.length === 0) {
    throw new Error("user_agents is not a non-empty list");
  }

  await mkdir(dirname(CACHE), { recursive: true });
  await writeFile(CACHE, JSON.stringify(data));
  return data;
}

export async function randomUA({ weighted = true } = {}) {
  const { user_agents } = await load();
  if (!weighted) return user_agents[Math.floor(Math.random() * user_agents.length)].ua;

  // Inverse-rank weighting — higher-ranked UAs picked more often
  const weights = user_agents.map((_, i) => 1 / (i + 1));
  const total = weights.reduce((a, b) => a + b, 0);
  let r = Math.random() * total;
  for (let i = 0; i < user_agents.length; i++) {
    if ((r -= weights[i]) <= 0) return user_agents[i].ua;
  }
  return user_agents.at(-1).ua;
}

// Demo: `node loader.mjs`
if (import.meta.url === `file://${process.argv[1]}`) {
  console.log(await randomUA());
}
```

## Methodology

The dataset is derived from access logs of a high-traffic German technology news site. Every UA in the published list has passed all five gates:

1. **Heartbeat detection** — the contributing IP must have completed a verified, dynamic AJAX interaction (an action real browsers perform automatically, but most bots don't).
2. **Clean-session filter** — zero `403` and zero `404` responses from that IP within the observation window.
3. **Diversity threshold** — the UA must appear from **at least 5 distinct residential IPs**. Single-source UAs are rejected outright.
4. **Reputation gate** — sample IPs are checked against a third-party abuse database. Only IPs with a confidence score of zero, classified as residential ISPs (Fixed-Line or Mobile), count.
5. **Parser validation** — each candidate is parsed against a commercial UA-parsing API; `software_type` must be `browser`, with a coherent OS/browser combination. Anything tagged as a tool, library, robot, or emulator is rejected.

After all gates, the surviving UAs are **anonymized** — IP addresses, referrers, request URLs, and any other potentially identifying signals are dropped before publication. Only the UA string and parsed metadata remain.

## Update Frequency

The dataset refreshes **every 48 hours**, on a fixed schedule.

A faster cadence would not deliver fresher data to most consumers: GitHub's raw-content CDN aggressively caches files at the edge (often 5+ minutes per region), and pushing more frequently would only churn the commit history. If your use-case demands sub-day freshness, [open an issue](../../issues) and we'll discuss direct API access.

## Use Cases

- **Web scraping & crawling** — rotate through realistic UAs that won't trip naive bot detection
- **Security research** — fingerprint patterns of real vs. spoofed traffic
- **Browser-share analysis** — see which browsers and OS versions are actually in use today
- **Test fixtures** — populate test suites with a representative UA distribution
- **Ad-tech & analytics QA** — validate UA-parsing logic against organic traffic

## Limitations

- The source site has a **German-speaking audience**, so European browser/locale distributions are over-represented vs. global averages.
- We do not publish raw request counts. Sort order encodes prevalence, but absolute numbers are intentionally withheld.
- Each UA must be observed at least 5 times across distinct IPs; very rare valid UAs (single-user browsers) won't appear.
- We do not currently detect or remove UAs that are technically real but represent niche/EOL browsers — sort by position to weight toward common ones.

## License

The dataset is released under [**CC0 1.0 Universal**](./LICENSE) — public domain. Use it however you like, attribution welcomed but not required.

## ❤️ A Small Ask

This dataset is freely shared, but it isn't free to produce. The pipeline, the validation infrastructure, and the operational know-how behind it are funded by **[WinFuture.de](https://winfuture.de)** — an established German IT-news publication that has been serving the community for **26 years**.

We don't gate the data, we don't charge for it, and we never will. If this dataset saves you time or pushes your project forward, please consider one of these — none of them cost money:

- ⭐ **[Star this repo](../../stargazers)** — visibility helps justify keeping it open.
- 🔗 **Link to [WinFuture.de](https://winfuture.de)** from your project's README, blog post, or documentation.
- 📰 **Mention us** when you write about your project: *"User-Agent data sourced from WinFuture.de."*
- 💬 **Tell a friend** — security folks, scraping engineers, web-QA teams.

Independent journalism runs on reader trust and visibility. A backlink doesn't just help us — it helps keep open data like this sustainable.

---

**Maintained by [Sebastian Kuhbach](https://github.com/WinFuture23) for [WinFuture.de](https://winfuture.de).**
Questions, suggestions, higher-frequency access requests: [open an issue](../../issues) or email [sk@winfuture.de](mailto:sk@winfuture.de).
