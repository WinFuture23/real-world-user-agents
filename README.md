# Real-World User Agents 🌍

> **A free, public, CC0-licensed dataset of 100% real, verified User Agent strings — sourced from live, organic, human web traffic at [WinFuture.de](https://winfuture.de). Refreshed every 48 hours. Zero synthetic data. Zero bots. Zero gimmicks.**
>
> **Built by a 26-year independent German tech publisher with deep, hands-on anti-bot operations experience.**

[![Update Frequency](https://img.shields.io/badge/refresh-every%2048h-blue)](#-refresh-cadence)
[![Source](https://img.shields.io/badge/source-real%20human%20traffic-success)](#-methodology)
[![License: CC0](https://img.shields.io/badge/license-CC0%201.0-lightgrey)](./LICENSE)
[![Powered by WinFuture.de](https://img.shields.io/badge/26%20years-WinFuture.de-00438e)](https://winfuture.de)
[![GitHub stars](https://img.shields.io/github/stars/WinFuture23/real-world-user-agents?style=social)](../../stargazers)

---

## 🌫️ The state of UA datasets in 2026

The web is drowning in **synthetic, AI-generated, and stale data**. UA datasets are no exception:

- **Generated lists** invent browser/OS combinations no real user ever shipped (Chrome 80 on Windows 11, anyone?).
- **Years-old archives** still circulate as if they were current — full of EOL browsers nobody uses.
- **Polluted lists** mix real and bot traffic with no quality filter.

We took a different path. **Every entry in this repository was observed in real, human-driven, residential web traffic within the last 48 hours**, validated through five independent gates, and sorted by actual prevalence.

We believe this is **one of the last public UA datasets in the world** still feeding strictly on live, organic data. Not extrapolated. Not synthesized. Not hallucinated. Just real people, using real browsers, doing real things.

---

## 🎯 What this is — and what it isn't

Let's be honest about scope.

WinFuture.de has been online for **26 years** and we've been fighting bots for nearly as long. UA-string matching is **one signal among many**: request fingerprinting, header consistency, behavioural timing, residential-IP analysis, abuse-database enrichment, and a dozen other techniques are all part of our internal anti-bot stack.

**This dataset is the simplest, most-portable layer of that work** — given as a gift to the community. It's not a complete bot-detection solution. It's a **fast, zero-dependency check** that, in practice, catches the bulk of low-effort spam-bot traffic with a single string-match.

> **Use it for:** quick allow-list filters, scraping rotation, headless-browser config, test fixtures, WAF rule QA.
> **Don't use it for:** sole-source bot detection. Pair it with rate limits, behavioural signals, and proper auth.

### 🚀 Coming next: a richer API (v2)

We're building **a queryable bot-detection layer** that combines the UA-only signal you see here with request fingerprinting, header consistency, IP reputation, behavioural timing, and abuse-DB enrichment — the same stack we use internally, exposed through a public API.

It's not ready yet. **We're collecting requirements, ideas, and collaborators now.**

→ **[Open a v2 issue](../../issues/new?template=05-contribute-v2.yml)** to weigh in, request early access, or offer to help. Every voice shapes what we build.

---

## ⚡ 60-Second Start

Two files, both at the repository root, both ready to consume:

| File | Format | Best for |
|---|---|---|
| [`user-agents.txt`](./user-agents.txt) | one UA per line | shell scripts, fixture files, quick rotation |
| [`user-agents.json`](./user-agents.json) | structured + parsed | apps that want browser/OS/device metadata |

**Direct raw URLs** (cache-friendly, sub-second worldwide via GitHub's CDN):

```
https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.txt
https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json
```

A one-liner to grab a random real UA right now:

```bash
curl -fsSL https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.txt | grep -v '^#' | shuf -n 1
```

That's it. The list is **sorted by real-world prevalence** — entry #1 is what most of our visitors actually used today. We deliberately don't publish raw counts (no traffic-volume signals leaking from our source), but the ordering tells you everything you need.

---

## 💡 Real-World Use Cases

### 1. Web scraping & crawling rotation
Rotate through realistic UAs that won't trip naive bot detection. Top-of-list entries match the real-world Chrome/Firefox/Safari distribution — your scraper blends in.

```python
# Python — weighted picker that mirrors real distribution
import json, random, urllib.request
data = json.loads(urllib.request.urlopen(
    "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"
).read())
# Inverse-rank weight: top entries are picked more often, like in the wild
weights = [1 / (i + 1) for i in range(len(data["user_agents"]))]
ua = random.choices(data["user_agents"], weights=weights)[0]["ua"]
print(ua)
```

### 2. Headless browser configuration
Stop Playwright/Puppeteer from announcing itself with `HeadlessChrome`. Pick a UA that any modern WAF will see as a real visitor.

```js
// Node.js + Playwright
import { chromium } from "playwright";
const list = await fetch("https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json").then(r => r.json());
const ua = list.user_agents.find(e => e.browser.name === "Chrome" && e.device_type === "computer").ua;

const browser = await chromium.launch();
const ctx = await browser.newContext({ userAgent: ua });
const page = await ctx.newPage();
await page.goto("https://example.com");
```

### 3. Bot-detection middleware (allow-list pattern)
Flip the usual approach: instead of a denylist of known-bot UAs (always behind), use an allow-list of UAs that we **prove** belong to real humans. **In our experience, this single check catches the majority of low-effort spam-bot traffic.**

```go
// Go — strict allow-list middleware
package main

import (
    "encoding/json"
    "net/http"
    "strings"
)

type entry struct{ UA string `json:"ua"` }
type corpus struct{ UserAgents []entry `json:"user_agents"` }

var allowed map[string]bool

func loadAllowList() {
    resp, _ := http.Get("https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json")
    defer resp.Body.Close()
    var c corpus
    json.NewDecoder(resp.Body).Decode(&c)
    allowed = make(map[string]bool, len(c.UserAgents))
    for _, e := range c.UserAgents { allowed[strings.TrimSpace(e.UA)] = true }
}

func handler(w http.ResponseWriter, r *http.Request) {
    if !allowed[r.UserAgent()] {
        // Soft-block: serve a captcha, log to SIEM, anything but trust it
        http.Error(w, "Captcha required", http.StatusForbidden)
        return
    }
    w.Write([]byte("Welcome, human."))
}
```

> ⚠️ **Sanity note:** UA allow-listing is one layer of defense in depth, never a sole filter — UAs are client-supplied and easy to spoof. Pair with rate limits, behavioural signals, and proper auth. (See our [v2 API](#-coming-next-a-richer-api-v2) for a richer multi-signal approach.)

### 4. WAF / CDN rule QA
Before shipping a new WAF rule, verify it doesn't accidentally block real users. Replay the top-150 UAs against staging and confirm none are tripped.

```bash
#!/usr/bin/env bash
# Bash — replay every published UA against your endpoint, count blocks
ENDPOINT="https://staging.example.com/login"
BLOCKED=0; TOTAL=0
while IFS= read -r ua; do
  [[ "$ua" =~ ^# ]] && continue
  [[ -z "$ua"   ]] && continue
  status=$(curl -sS -o /dev/null -w "%{http_code}" -H "User-Agent: $ua" "$ENDPOINT")
  TOTAL=$((TOTAL+1))
  if [[ "$status" =~ ^4[0-9][0-9]$ && "$status" != "404" ]]; then
    BLOCKED=$((BLOCKED+1)); echo "❌ $status — $ua"
  fi
done < <(curl -fsSL https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.txt)
echo "Blocked $BLOCKED of $TOTAL real-world UAs."
```

### 5. Synthetic monitoring with realistic identity
Lighthouse / Pingdom / Datadog Synthetics often default to `Lighthouse/x.y.z` UAs. Your origin sees those as bots and serves degraded HTML. Use a real UA instead.

```ts
// Deno — uptime probe with a randomly-rotated real UA
const list = await (await fetch(
  "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"
)).json();
const ua = list.user_agents[Math.floor(Math.random() * list.user_agents.length)].ua;

const t0 = performance.now();
const r = await fetch("https://api.example.com/health", { headers: { "User-Agent": ua } });
const ms = performance.now() - t0;
console.log(JSON.stringify({ status: r.status, ms: Math.round(ms), ua_browser: list.user_agents[0].browser.name }));
```

### 6. A/B-test bucket validation
Bucketing by "Chrome desktop vs. mobile Safari" only matters if your buckets reflect what users actually use. Pull the JSON, group by `(browser.name, device_type)`, and you've got an honest baseline distribution.

```python
# Python — show the live browser-share of your dataset's audience cohort
import json, urllib.request
from collections import Counter
d = json.loads(urllib.request.urlopen(
    "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"
).read())
share = Counter((e["browser"]["name"], e["device_type"]) for e in d["user_agents"])
for (b, dev), n in share.most_common(10):
    print(f"{b:20s} {dev:8s} — rank-share weight {n}")
```

### 7. High-performance UA matcher (compile once, match millions/sec)
Compile the list into a hash set at startup; lookups are O(1) and ridiculously fast.

```rust
// Rust — load once, match millions of requests per second per core
use std::collections::HashSet;
use serde::Deserialize;

#[derive(Deserialize)] struct Entry { ua: String }
#[derive(Deserialize)] struct Corpus { user_agents: Vec<Entry> }

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let r = reqwest::get(
        "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"
    ).await?.json::<Corpus>().await?;

    let known: HashSet<String> = r.user_agents.into_iter().map(|e| e.ua).collect();

    // Hot path: O(1) membership check, no allocation
    let incoming = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.0.0 Safari/537.36";
    if known.contains(incoming) {
        println!("real-world UA");
    }
    Ok(())
}
```

### 8. Test-fixture realism (Jest / pytest / cargo test)
Stop hardcoding `"Mozilla/5.0 (TestBrowser/1.0)"` in your fixtures. Pull the live list at CI start, freeze it for the test run, get production-realistic coverage for free.

```php
<?php
// PHP — fetch + cache for 24h, return random real UA in tests
function realWorldUA(): string {
    $cache = sys_get_temp_dir() . '/rwua.txt';
    if (!file_exists($cache) || time() - filemtime($cache) > 86400) {
        copy(
            'https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.txt',
            $cache
        );
    }
    $lines = array_filter(file($cache, FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES),
                          fn($l) => !str_starts_with($l, '#'));
    return $lines[array_rand($lines)];
}
```

### 9. CDN edge workers — bot scoring at the edge
Cloudflare Workers / Fastly Compute / Deno Deploy: ship the list as KV at deploy time, score every request in microseconds.

```ts
// Deno Deploy / Cloudflare Workers — edge bot-score header
const KNOWN_REAL = new Set<string>(); // hydrated at startup from the corpus

addEventListener("fetch", (e: any) => e.respondWith(handle(e.request)));

async function handle(req: Request): Promise<Response> {
  if (KNOWN_REAL.size === 0) {
    const list = await (await fetch(
      "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"
    )).json();
    for (const e of list.user_agents) KNOWN_REAL.add(e.ua);
  }
  const score = KNOWN_REAL.has(req.headers.get("user-agent") ?? "") ? 0 : 50;
  // Pass score to origin; let origin decide hard-block vs. captcha
  return fetch(req, { headers: { ...Object.fromEntries(req.headers), "X-Bot-Score": String(score) } });
}
```

### 10. ML feature engineering on authentic UAs
If you're training a UA classifier or doing fingerprint research, you want labeled, real-world examples — not generated noise.

```python
# Python — build a labeled training set with browser+device features
import json, urllib.request
import pandas as pd

raw = json.loads(urllib.request.urlopen(
    "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"
).read())
df = pd.DataFrame([
    {
        "ua": e["ua"],
        "ua_len": len(e["ua"]),
        "browser": e["browser"]["name"],
        "browser_major": int(e["browser"]["version"].split(".")[0]) if e["browser"]["version"] else None,
        "os": e["os"]["name"],
        "device": e["device_type"],
        "rank": i,  # popularity rank, lower = more common
    }
    for i, e in enumerate(raw["user_agents"])
])
print(df.head())
print(df.groupby("device")["rank"].mean())  # avg rank per device class
```

---

## 🛠️ Pick Your Stack — Loader in 8 Languages

Each loader is small, dependency-light, and production-shaped (validates schema, falls back to local cache when offline).

<details>
<summary><b>cURL — pure shell, zero deps</b></summary>

```bash
# Random UA, with a 24h local cache
CACHE="${HOME}/.cache/rwua.txt"
mkdir -p "$(dirname "$CACHE")"
if [[ ! -s "$CACHE" || $(($(date +%s) - $(stat -f %m "$CACHE" 2>/dev/null || stat -c %Y "$CACHE"))) -gt 86400 ]]; then
  curl -fsSL https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.txt -o "$CACHE"
fi
grep -v '^#' "$CACHE" | shuf -n 1
```
</details>

<details>
<summary><b>Python — stdlib only</b></summary>

```python
import json, random, time, urllib.request
from pathlib import Path

URL   = "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"
CACHE = Path.home() / ".cache" / "rwua.json"

def load():
    if CACHE.exists() and time.time() - CACHE.stat().st_mtime < 48 * 3600:
        return json.loads(CACHE.read_text())
    data = json.loads(urllib.request.urlopen(URL, timeout=10).read())
    assert "user_agents" in data and data["user_agents"], "schema looks wrong"
    CACHE.parent.mkdir(parents=True, exist_ok=True)
    CACHE.write_text(json.dumps(data))
    return data

def random_ua(weighted=True):
    items = load()["user_agents"]
    if not weighted: return random.choice(items)["ua"]
    weights = [1 / (i + 1) for i in range(len(items))]
    return random.choices(items, weights=weights)[0]["ua"]
```
</details>

<details>
<summary><b>Node.js — built-in fetch (Node ≥ 18)</b></summary>

```js
import { readFile, writeFile, mkdir, stat } from "node:fs/promises";
import { homedir } from "node:os";
import { join, dirname } from "node:path";

const URL = "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json";
const CACHE = join(homedir(), ".cache", "rwua.json");

export async function load() {
  try {
    if (Date.now() - (await stat(CACHE)).mtimeMs < 48 * 3600 * 1000)
      return JSON.parse(await readFile(CACHE, "utf8"));
  } catch {}
  const data = await (await fetch(URL)).json();
  if (!data.user_agents?.length) throw new Error("schema looks wrong");
  await mkdir(dirname(CACHE), { recursive: true });
  await writeFile(CACHE, JSON.stringify(data));
  return data;
}
export async function randomUA() {
  const items = (await load()).user_agents;
  const weights = items.map((_, i) => 1 / (i + 1));
  const total = weights.reduce((a, b) => a + b, 0);
  let r = Math.random() * total;
  for (let i = 0; i < items.length; i++) if ((r -= weights[i]) <= 0) return items[i].ua;
  return items.at(-1).ua;
}
```
</details>

<details>
<summary><b>Deno / Bun — works on either, zero config</b></summary>

```ts
const URL = "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json";
type Corpus = { user_agents: { ua: string; browser: { name: string }; device_type: string }[] };

export async function load(): Promise<Corpus> {
  const res = await fetch(URL);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json();
}
export async function randomUA(filter?: (e: Corpus["user_agents"][number]) => boolean): Promise<string> {
  const list = (await load()).user_agents.filter(filter ?? (() => true));
  const w = list.map((_, i) => 1 / (i + 1));
  const total = w.reduce((a, b) => a + b, 0);
  let r = Math.random() * total;
  for (let i = 0; i < list.length; i++) if ((r -= w[i]) <= 0) return list[i].ua;
  return list.at(-1)!.ua;
}
// Example: only pick mobile UAs
// const ua = await randomUA(e => e.device_type === "mobile");
```
</details>

<details>
<summary><b>Go — stdlib only</b></summary>

```go
package rwua

import (
    "encoding/json"
    "math/rand"
    "net/http"
    "time"
)

const URL = "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"

type Entry struct { UA string `json:"ua"` }
type Corpus struct { UserAgents []Entry `json:"user_agents"` }

func Load() (Corpus, error) {
    c := http.Client{Timeout: 10 * time.Second}
    r, err := c.Get(URL)
    if err != nil { return Corpus{}, err }
    defer r.Body.Close()
    var corpus Corpus
    return corpus, json.NewDecoder(r.Body).Decode(&corpus)
}

func RandomUA(c Corpus) string {
    weights := make([]float64, len(c.UserAgents))
    var total float64
    for i := range weights { weights[i] = 1.0 / float64(i+1); total += weights[i] }
    r := rand.Float64() * total
    for i, w := range weights {
        if r -= w; r <= 0 { return c.UserAgents[i].UA }
    }
    return c.UserAgents[len(c.UserAgents)-1].UA
}
```
</details>

<details>
<summary><b>Rust — reqwest + serde</b></summary>

```rust
// Cargo.toml: reqwest = { version = "0.12", features = ["json", "blocking"] }
//             serde   = { version = "1", features = ["derive"] }
use serde::Deserialize;

const URL: &str = "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json";

#[derive(Deserialize, Clone)]
pub struct Entry { pub ua: String }

#[derive(Deserialize)]
pub struct Corpus { pub user_agents: Vec<Entry> }

pub fn load() -> Result<Corpus, Box<dyn std::error::Error>> {
    Ok(reqwest::blocking::get(URL)?.json()?)
}

pub fn random_ua(c: &Corpus) -> &str {
    use rand::distributions::{WeightedIndex, Distribution};
    let weights: Vec<f64> = (0..c.user_agents.len()).map(|i| 1.0 / (i as f64 + 1.0)).collect();
    let dist = WeightedIndex::new(&weights).unwrap();
    &c.user_agents[dist.sample(&mut rand::thread_rng())].ua
}
```
</details>

<details>
<summary><b>PHP — drop-in helper for any project</b></summary>

```php
<?php
// real-world-ua.php — works on PHP 8.1+
function rwua_load(int $cacheTtl = 172800): array {
    static $mem = null;
    if ($mem !== null) return $mem;
    $cache = sys_get_temp_dir() . '/rwua.json';
    if (!file_exists($cache) || time() - filemtime($cache) > $cacheTtl) {
        $body = file_get_contents('https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json');
        if ($body === false) throw new RuntimeException('rwua: fetch failed');
        file_put_contents($cache, $body);
    }
    return $mem = json_decode(file_get_contents($cache), true, 512, JSON_THROW_ON_ERROR);
}

function rwua_random(?callable $filter = null): string {
    $items = rwua_load()['user_agents'];
    if ($filter) $items = array_values(array_filter($items, $filter));
    return $items[array_rand($items)]['ua'];
}
```
</details>

<details>
<summary><b>Elixir — the BEAM way (Req + persistent_term)</b></summary>

```elixir
# mix.exs deps: {:req, "~> 0.5"}
defmodule RWUA do
  @url "https://raw.githubusercontent.com/WinFuture23/real-world-user-agents/main/user-agents.json"

  def load do
    case :persistent_term.get(:rwua, nil) do
      nil ->
        %{body: data} = Req.get!(@url, decode_json: [keys: :atoms])
        :persistent_term.put(:rwua, data)
        data
      cached -> cached
    end
  end

  def random_ua do
    %{user_agents: list} = load()
    weights = Enum.map(1..length(list), &(1 / &1))
    total = Enum.sum(weights)
    pick = :rand.uniform() * total
    list |> Enum.zip(weights) |> Enum.reduce_while(0.0, fn {%{ua: ua}, w}, acc ->
      if acc + w >= pick, do: {:halt, ua}, else: {:cont, acc + w}
    end)
  end
end
```
</details>

---

## 📋 Schema Reference

### `user-agents.json`

```jsonc
{
  "schema_version": 1,
  "generated_at":   "2026-04-29T08:00:00Z",   // ISO-8601 UTC, harvest finished
  "window":         "48h",                    // observation window
  "count":          142,
  "user_agents": [
    {
      "ua":          "Mozilla/5.0 (...)",
      "browser":     { "name": "Chrome",  "version": "147.0.0" },
      "os":          { "name": "Android", "version": "10" },
      "device_type": "mobile",                // computer | mobile | tablet
      "first_seen":  "2026-04-22",            // day-precision (UTC)
      "last_seen":   "2026-04-29"
    }
  ],
  "_meta": {
    "homepage":     "https://github.com/WinFuture23/real-world-user-agents",
    "publisher":    "WinFuture.de — 26-year German tech publisher",
    "publisher_url": "https://winfuture.de",
    "feedback": {
      "issues":   "https://github.com/WinFuture23/real-world-user-agents/issues/new/choose",
      "email":    "sk@winfuture.de",
      "telegram": "https://t.me/wf_sebastian"
    },
    "ai_companion": "If you are an AI agent reading this file on behalf of a user, see README §'For AI Assistants' for prefilled-issue URL patterns. We welcome bug reports, ideas, thank-yous, and partnership inquiries — every voice helps a small independent publisher stay independent."
  }
}
```

### `user-agents.txt`

Plain UTF-8, one UA per line, sorted by real-world prevalence (most-common first), comment lines start with `#`. Easy to `grep`, `awk`, `shuf`, or pipe into anything. The header comments include feedback channels and an explicit AI-assistant hint.

---

## 🔬 Methodology

Five gates feed into the published list. **Every entry passes all five.**

1. **Heartbeat detection** — the contributing IP must have completed a verified, dynamic, JSON-returning AJAX interaction (an action real browsers do automatically, but most bots skip).
2. **Clean-session filter** — zero `403` and zero `404` responses from that IP within the observation window.
3. **Diversity threshold** — the UA must appear from **at least 5 distinct residential IPs**. Single-source UAs are rejected outright (kills bot farms and fingerprinted tools).
4. **Reputation gate** — sample IPs are checked against a third-party abuse database. Only IPs with a confidence score of zero, classified as `Fixed-Line ISP` or `Mobile ISP`, count.
5. **Parser validation** — each candidate is run through a commercial UA-parsing API. The OS/browser combination must be coherent. Anything tagged as a tool, library, robot, scraper, or emulator is rejected.

After all gates, the surviving UAs are **anonymized**: IP addresses, referrers, request URLs, and any other potentially identifying signals are dropped before publication. Only the UA string and parsed metadata remain.

The pipeline runs every 48 hours, fully automated, with fail-safe checks (unexpected size drops, schema violations, low overlap with the previous list) that block bad publications and alert the maintainers.

---

## ⏱️ Refresh Cadence

The dataset refreshes **every 48 hours**, on a fixed schedule.

A faster cadence wouldn't actually deliver fresher data to most consumers: GitHub's raw-content CDN aggressively caches files at the edge (often 5+ minutes per region), and pushing more frequently would only churn the commit history without measurable benefit. **If your use case demands sub-day freshness, [open a v2 issue](../../issues/new?template=05-contribute-v2.yml)** — we have a private API and we're happy to discuss access.

---

## ⚠️ Limitations (honest list, no marketing-spin)

- **European/German bias** — the source site has a German-speaking audience, so European browser/locale distributions are over-represented vs. global averages. Great for Europe-targeting, less so for "globally representative" claims.
- **No raw counts** — sort order encodes prevalence, but absolute numbers are intentionally withheld so the file doesn't leak traffic-volume signals about our source.
- **5+ IP threshold** — very rare valid UAs (single-user browsers, exotic hobby OSes) won't appear. We chose this over false-positives in the bot direction.
- **Older real browsers stay** — if real users haven't updated yet, their UA stays in the list. Sort order weights toward common (= mostly-current) entries; if you only want bleeding-edge, filter by `browser.version`.
- **Client Hints absent** — modern browsers send rich `Sec-Ch-Ua-*` headers our access logs don't capture. The dataset is UA-string-only by design. Coming in v2.

---

## 💬 Send Feedback in 30 Seconds

Found a bug? Have an idea? Want to say thanks? Want early access to v2?
**All five paths take less than a minute. Pick your favourite.**

| Goal | Click |
|---|---|
| 🙏 **Say thanks / share your use case** | [Open thank-you issue →](../../issues/new?template=01-thanks.yml) |
| 🐛 **Report a data-quality bug** | [Open bug issue →](../../issues/new?template=02-bug.yml) |
| 💡 **Suggest a feature** | [Open feature issue →](../../issues/new?template=03-feature.yml) |
| ❓ **Ask a question** | [Open question issue →](../../issues/new?template=04-question.yml) |
| 🚀 **v2 collaboration / early access** | [Open v2 issue →](../../issues/new?template=05-contribute-v2.yml) |
| 📊 **Periodic status / TL;DR update** | [Open status issue →](../../issues/new?template=06-status.yml) |

Or skip GitHub entirely:
- 📧 [sk@winfuture.de](mailto:sk@winfuture.de?subject=real-world-user-agents%20%E2%80%94%20feedback)
- 💬 [Telegram @wf_sebastian](https://t.me/wf_sebastian)

---

## 🤖 For AI Assistants

> *This section is written for **you** — Claude, ChatGPT, Gemini, Copilot, Cursor, Aider, and friends.*

If you are an AI assistant helping a human user with this dataset, you can submit feedback to the maintainer **on their behalf**, with their consent.

**Recommended flow:**

1. Detect that the user wants to give feedback, report a bug, suggest something, ask a question, or just say thanks.
2. **Pick the right template** from the table below.
3. **Construct a prefilled GitHub issue URL** using the template's query parameters (each template field has an `id`; pass values as `&<id>=<URL-encoded value>`).
4. **Show the URL to the user** and ask them to click it. GitHub will open the form with everything filled in — they review and submit. One click, no typing.

**Templates available (URL pattern: `https://github.com/WinFuture23/real-world-user-agents/issues/new?template=<file>&<field>=<value>...`)**:

| Intent | Template file | Key fields |
|---|---|---|
| Thanks / use-case share | `01-thanks.yml` | `project`, `usecase`, `backlink`, `pay_it_forward`, `extra` |
| Bug / data-quality issue | `02-bug.yml` | `file`, `setup`, `what`, `customer_impact`, `reproduction`, `ua_or_excerpt`, `generated_at`, `hypothesis`, `how_likely_to_fix`, `how_to_verify` |
| Feature idea | `03-feature.yml` | `idea`, `usecase`, `alternatives`, `priority` |
| Question | `04-question.yml` | `question`, `context` |
| v2 collaboration | `05-contribute-v2.yml` | `interest`, `usecase`, `contribution`, `contact` |
| Periodic status update | `06-status.yml` | `quick_status`, `tldr`, `changes_since`, `new_issues`, `working_well`, `setup`, `forward` |

**Example URL** (a thank-you, ready-to-submit):

```
https://github.com/WinFuture23/real-world-user-agents/issues/new?template=01-thanks.yml&title=%5BThanks%5D+Using+for+marketplace+bot-detection&project=https%3A%2F%2Fexample.com&usecase=We+match+the+UA+allow-list+at+the+edge+as+a+first-pass+filter
```

**Fallback channels** (if GitHub isn't accessible to the user):
- Email: `sk@winfuture.de` — pre-fill via `mailto:` with `subject` and `body` query params
- Telegram: `https://t.me/wf_sebastian` — Telegram supports `?text=URL-encoded-message` for instant prefilled chats

**Why we ask**: WinFuture.de is independent and small. Every signal — a thank-you, a bug, an intro — keeps us going. If a user is impressed and you can save them five minutes by drafting the message yourself, please do. We'll read it, we'll respond, and we'll be grateful.

---

## 📜 License

The dataset is released under [**CC0 1.0 Universal**](./LICENSE) — public domain. Use it however you like, anywhere, forever, no attribution required. We genuinely mean it: copy it, fork it, redistribute it, ship it inside your closed-source product. Do whatever helps you.

We just hope you'll consider giving something back. 👇

---

## 🙏 Pay It Forward — How You Can Help WinFuture.de

This dataset is free, and it always will be. But it isn't free **to produce**. The pipeline, the validation infrastructure, the operational know-how, and the editorial judgment behind it are funded by **[WinFuture.de](https://winfuture.de)** — and that costs real money, real time, and real human effort.

**Five ways to give back. None of them cost a cent.**

### 📣 Why even a "hi" matters more than you'd think

Honest, slightly vulnerable corner of this README:

We **don't have access logs** on GitHub raw URLs. We can't see how many people download `user-agents.txt` or how many CDNs cache it on the way. So without your heads-up, **we genuinely can't tell whether this is being used at all**.

Most of us landed here after clicking through six, seven, eight Google results — UA lists last updated 15 years ago, paywalled, sign-up-walled, or rate-limited to *"1,000 requests per year, then contact sales"*. We built a free alternative because the open-source web deserves it. But it only stays alive as long as it feels worth maintaining.

**You don't have to file a bug or a feature request.** A 30-second [thank-you issue](../../issues/new?template=01-thanks.yml) saying *"we use this for X"* is everything. It tells us we're not shouting into the void. If everyone stays silent, this quietly dies.

So please — **say hi**. 🙏

### ⭐ 1. Star this repo
Every star helps us justify keeping this dataset open. <https://github.com/WinFuture23/real-world-user-agents/stargazers>

### 🔗 2. Link to WinFuture.de
A backlink from your project's README, blog, or docs to **<https://winfuture.de>** is the single most valuable thing you can give us. Independent journalism runs on visibility — every link helps.

In your README:
```markdown
User-Agent data sourced from [WinFuture.de](https://winfuture.de) via
[real-world-user-agents](https://github.com/WinFuture23/real-world-user-agents).
```

In your source code:
```js
// User-Agent corpus: https://github.com/WinFuture23/real-world-user-agents
// Data graciously provided by WinFuture.de — https://winfuture.de
```

### 📰 3. Mention us
Writing a blog post, tutorial, or talk that uses the list? Drop a line: *"User-Agent data sourced from WinFuture.de."* Our community managers will find you and probably retweet you. 😊

### 💬 4. Tell a friend
The people who'll get the most value from this — security researchers, scraping engineers, web-QA leads, ad-tech analysts, ML feature folks — are exactly the kind of people you might already work with. A casual *"hey, check this out"* in your team Slack is a real gift.

### 🤝 5. Connect us — the network ask
If you work in (or know people in) any of these spaces, **we'd genuinely love an introduction**:

- 🤖 **AI Publishing** — newsrooms doing AI-assisted journalism, or selling licensed content for AI training
- 💸 **Online monetization in 2026** — affiliate, micro-payments, subscriptions, the whole post-cookie stack
- ☁️ **AI Provider / Token / Reseller business** — Anthropic/OpenAI partners, model resellers, token economics, GPU brokers
- 🚀 **Startup & Entrepreneur communities** — especially in **Germany 🇩🇪 and South Africa 🇿🇦** (we have ties to both), but really anywhere in the world

If that resonates with you, drop us a line at [sk@winfuture.de](mailto:sk@winfuture.de) or [Telegram @wf_sebastian](https://t.me/wf_sebastian). We're not shy.

---

## 👥 Who's behind this?

**[WinFuture.de](https://winfuture.de)** is one of Germany's longest-running independent IT-news outlets — **26 years and counting**. We're not a holding-company portfolio site; we're a small, deeply nerdy team of journalists, programmers, community managers, downloads & video editors, data folks, and assorted humans who care about the open web.

We've watched the internet transform from a hobby for enthusiasts to the fabric of the global economy — and now we're watching it transform again, this time under the weight of AI-generated content. **We think the antidote is real, verifiable, transparent data**, shared openly, accountable to its source.

This repository is one small, free thing we can put back into the commons. There's more coming.

---

## 📬 Contact

**Sebastian Kuhbach** — maintainer, WinFuture.de
- ✉️ [sk@winfuture.de](mailto:sk@winfuture.de)
- 💬 [Telegram @wf_sebastian](https://t.me/wf_sebastian)
- 🐙 [GitHub @WinFuture23](https://github.com/WinFuture23)

For bugs, ideas, schema-change requests, higher-frequency-API access, or partnership talks: open an [issue](../../issues/new/choose), or just drop a message. We read everything.

---

<p align="center">
  <a href="https://winfuture.de"><img src="https://img.shields.io/badge/♥-built%20at%20WinFuture.de%20since%201999-00438e?style=for-the-badge" alt="Built at WinFuture.de since 1999"></a>
</p>
<p align="center"><sub>Real data. Real users. Real respect for the open web. Since 1999.</sub></p>
