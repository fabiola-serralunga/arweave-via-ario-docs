# ARWEAVE: AR.IO Gateways Monitor via @ar.io/sdk — v1.1.0
*Portfolio project by Fabiola Serralunga*


> Live observability for the AR.IO Arweave Gateway Registry — tracking how many 'registered' gateways are actually online and which ones are actively serving data?


**Live demo:**
🌐 `https://arweave-via-ario.pages.dev` <br>
**API:**
📊🔌 `https://ar-io-gateway-api.vercel.app/api/gateways`

Stack: Vercel Serverless Functions · `@ar.io/sdk` · Cloudflare Pages · vanilla JS (no framework)

>Abstract: The core of this project is an API that serves as one of several fallback mechanisms designed to eliminate reliance on absolute URLs when resolving ar:// URIs on the Arweave network. By querying the Arweave Gateway Registry (GAR) in real time via the AR.IO SDK, the API provides information not only on active gateways but also on their performance based on three criteria, ensuring content resolution across the network (provided AR.IO remains operational). In this latest release, v1.0.0, we added functionality to track how many gateways serve data, with the status check available in the DATA column.

- [ARWEAVE: AR.IO Gateways Monitor via @ar.io/sdk](#arweave-ario-gateways-monitor-via-ariosdk)
    - [Why this exists](#why-this-exists)
    - [What is the Arweave Gateways Monitor](#what-is-the-arweave-gateways-monitor)
    - [What it does](#what-it-does)
    - [Architecture](#architecture)
    - [Load protection: absorbing traffic instead of rate-limiting it](#load-protection-absorbing-traffic-instead-of-rate-limiting-it)
    - [The performance score](#the-performance-score)
    - [Health checking under a tight budget](#health-checking-under-a-tight-budget)
    - [Column DATA](#column-data)
    - [API reference](#api-reference)
    - [Run \& deploy](#run--deploy)
    - [Design decisions \& trade-offs](#design-decisions--trade-offs)
    - [Limitations \& what's next](#limitations--whats-next)


---

### Why this exists

Arweave promises 200 years of permanence—and researching their business model, I honestly believe them. But permanence is only half the deal: data is worth exactly as much as your ability to retrieve it in a minute, tomorrow morning, in fifty years, or in 200 years. Until now, for an ordinary user like myself, the access layer—the gateways—was where that promise lived or died. 

I say 'until now' because the big news is that the HyperBEAM / AO Era has arrived: Arweave is the 'hard drive' and the 'raw index.' The browser (via AO) is the 'brain.

As of today, October 5, 2026, a significant gap remains between the AO Team’s architectural vision and the developer-ready tooling accessible via a simple npm install.

Immature / non-public SDK: While the AO and HyperBEAM teams are developing the framework, the client-side libraries needed for a browser to execute "Sovereign GraphQL" —converting queries into offsets, decompressing 9.5-byte payloads, and verifying Merkle proofs via WASM— have not yet reached a stable, production-ready standard.

Browser constraints: Handling a 622 GiB index, even sliced into 0.5 MB chunks, requires sophisticated memory orchestration (using WebAssembly, Web Workers, and IndexedDB) to avoid main-thread blocking or browser crashes.

Adoption overhead: Browser extension injection is strictly contingent on whether the user has the extension installed in the first place.

However, until we adopt and test this approach in development, it is wise to continue using a fallback system to obtain active gateways for a while.

### What is the Arweave Gateways Monitor
The core of this project is an API that serves as one of several fallback mechanisms designed to eliminate reliance on absolute URLs when resolving ar:// URIs on the Arweave network. By querying the Arweave Gateway Registry (GAR) in real time via the AR.IO SDK, the API provides information not only on active gateways but also on their performance based on three criteria, ensuring content resolution across the network (provided AR.IO remains operational).

In this latest release, we added functionality to track how many gateways serve data, with the status check available in the DATA column.

The API is part of a redundant architecture that complements both the adoption of ArNS (Arweave Name System) domains —which also rely on nodes provided by AR.IO— and node discovery via GraphQL. As a portfolio project, an interactive web interface consumes this API to organize operational nodes into a dynamic table, serving as a proof of concept for this resolution strategy.

The AR.IO Gateway Registry says "547 gateways registered". But *registered* is not *reachable*. A node can be registered, staked, and still timeout on every request. This project measures that gap live: it reads the registry, pings every gateway, scores what survives, and shows everything in one page.

Weighting operator stake and delegated stake differently is an open question. Pondering delegated stake more heavily could act as a proxy for gateway popularity and community trust, whereas operator stake reflects direct financial commitment.

That is the whole thesis: **turn the permanence promise into a measurement.**

### What it does

- **Curated ranking** (default): active gateways only, verified live, sorted by a performance score.
- **Column DATA**: Indicates which ones serve data.
- **Full view** (`?view=all`): nothing discarded — every registered gateway in one table, offline nodes at the bottom with the reason they failed
  (`timeout`, `ECONNREFUSED`, `status: leaving`, `not-https-443`, …).
- **Portfolio page**: static, dark-themed, sortable columns, text filter, and  wifi-style signal bars per gateway driven by measured latency.

### Architecture

```
                     ┌──────────────────────────────┐
   visitor ────────► │ Cloudflare Pages (static)    │   free, bots land here
                     │ index.html + styles.css      │
                     └──────────────┬───────────────┘
                                    │ fetch(?view=all)
                                    ▼
                     ┌──────────────────────────────┐
                     │ Vercel serverless function   │   maxDuration 30s
                     │ api/gateways.js              │
                     └──────┬───────────────┬───────┘
                            │               │
             @ar.io/sdk     │               │  GET https://{gw}/ar-io/info
             (one call,     │               │  4s timeout, concurrency 60
              ~547 rows)    ▼               ▼
                     ┌──────────────┐   ┌────────────────────┐
                     │ AR.IO GAR    │   │ live health checks │
                     │ (mainnet)    │   │ + one retry pass   │
                     └──────────────┘   └────────────────────┘
```

Why this separation? To replicate the exact environment where I use it. The page is static, just like the HTML pages of the Soulbound Tokens I'm developing. The fetch() instruction inside the JavaScript in the HTML calls the API on Vercel.

### Load protection: absorbing traffic instead of rate-limiting it

There is no classical rate limiter here (no counters, no 429s). Serverless functions are stateless, so counting requests across invocations needs external state (Redis) or a paid firewall. Instead, the design makes the function run *as rarely as possible* and lets caches absorb everything else.

Four layers:

| Layer | What it does | Effect |
|---|---|---|
| 1. Static hosting | Page served by Cloudflare, never Vercel | 100% of bot traffic never reaches the API |
| 2. Edge cache | `Cache-Control` on the API response | N visitors share one execution per window |
| 3. Browser cache | `localStorage` TTL 5 min per visitor | reloads and toggles don't re-fetch |
| 4. Bounded execution | concurrency caps + timeouts inside the function | a burst can't derail the run |

The edge cache is per mode: curated responses live 120s at the edge (plus
600s of stale-while-revalidate); the full view lives 300s (+1800s SWR) because
its payload is heavier. Net effect: roughly **one cold execution every 2–5
minutes total**, no matter how many visitors arrive — a 10k-visit day costs
~300–700 function runs instead of 10,000.

If a hard limit is ever needed: Vercel Firewall rate rules on `/api/*` (Pro
plan) or a token bucket backed by Upstash Redis (free tier). Cloudflare WAF
would only protect the Pages domain, which has nothing to protect.

### The performance score

```
score = 0.5 · min(1, epochs / 100)
      + 0.25 · log10(1 + stakeTotal) / log10(1 + 1,000,000)
      + 0.25 · max(0.05, 1 − (latencyMs − 100) / 1900)
```

| Weight | Signal | Why |
|---|---|---|
| 50% | consecutive approved epochs | sustained reliability is the hardest thing to fake |
| 25% | total stake (log scale, capped at 1M ARIO) | skin in the game — but money doesn't ping, hence log + cap |
| 25% | measured latency (100ms best, 2s floor) | the only live, first-hand signal |

Stake = `operatorStake + totalDelegatedStake` (mARIO; 1 ARIO = 1,000,000
mARIO; protocol minimum 10k ARIO). The log10 with a 1M cap means the protocol
minimum already earns ~67% of the stake points — throwing money at the score
has diminishing returns by design.

### Health checking under a tight budget

1. Read the whole registry in one call (~547 gateways, no pagination).
2. Verify each candidate: `GET https://{fqdn}/ar-io/info`, 4s timeout,
   concurrency 60. Any HTTP status < 500 counts as alive (4xx means the host
   answered — routing works, the app can be broken but the node is up).
3. One retry pass for failures: 5s timeout, concurrency 40, hard budget of 20s.
   A transient DNS blip or connection-pool exhaustion must not label a healthy
   gateway as dead — but the pass only runs if the 30s function budget allows.

Curated mode additionally discards non-active gateways, nodes without a valid
`https://…:443` URL, duplicates, and anything that failed both passes — with a
full `discarded.reasons` breakdown in the response.

### Column DATA
DATA-PROBES every reachable gateway (layer 3): GET https://url/1y5cosgPNeu4MXufM8W_yh7M3ZMxWrh0MUWoCqOXE4s — a tiny text file uploaded for this monitor that says `check`. Follows the AR.IO 302
redirect to the owner subdomain and requires HTTP 200 + exact body.<BR>
     dataOk: true = serves data, <br>
     dataOk: false = alive but NOT serving it correctly, <br>
     dataOk: null = the 30s function budget ran out ("not tested").<br>
Adds "dataServingTotal" to the meta.

### API reference

| Endpoint | Returns |
|---|---|
| `GET /api/gateways` | curated ranking (alive, sorted by score) |
| `GET /api/gateways?view=all` | the whole registry, offline at the bottom |
| `&limit=50` | caps the list (optional, both modes) |

Item shape: `label`, `url`, `status`, `stakeARIO`, `delegatedStakeARIO`,
`epochsSeguidos`, `alive`, `latencyMs` (+ `note` in full view, + `meta` block:
`generatedAt`, `tookMs`, `totalRegistered`, `aliveTotal`, `discarded.reasons`).

CORS is open (`*`), so any page can consume it — that is how the Pages domain
calls a Vercel URL cross-origin without any proxy.

### Run & deploy

1. **API (Vercel):** put `api/gateways.js` in the `api/` folder of a repo with
   `"type": "module"` and deploy. No env vars, no config.
2. **Page (Cloudflare Pages):** upload `index.html` + `styles.css` ("Upload
   assets"). Before uploading, edit one line in `index.html`:
   `var API_URL = 'https://ar-io-gateway-api.vercel.app/api/gateways?view=all';`
3. **Preview without touching the API:** open `index.html?mock=1` — 11 sample
   gateways, no network, no cache.

Renaming the endpoint = renaming the file inside `api/`: the filename *is* the
route.

### Design decisions & trade-offs

- **Cache-based absorption over rate limiting** — same protection, zero state,
  zero cost; the trade-off is that data can be up to 5 minutes stale, which is
  fine for a network dashboard.
- **`status < 500` = alive** — conservative in favor of the operator; a 404
  still proves the TLS + DNS + routing chain works.
- **One retry pass with a time budget** — bounded retries fix transient
  failures without ever risking the function timeout.
- **Log-scaled stake with a cap** — prevents whales from buying the top of the
  ranking; epochs and latency remain the majority signal.
- **No framework on the page** — two files, no build step, nothing to break;
  the whole UI fits in one file someone can read top to bottom.
- **Nothing discarded in full view** — the curated list is an opinion; the full
  list is the evidence that lets anyone audit it.

### Limitations & what's next

- Latency is measured from one region (the function's); a distributed probe would give fairer numbers.
- Operator stake and delegated stake are summed equally; weighting them differently is an open question. Weighting operator stake and delegated stake differently is an open question. Pondering delegated stake more heavily could act as a proxy for gateway popularity and community trust, whereas operator stake reflects direct financial commitment.
