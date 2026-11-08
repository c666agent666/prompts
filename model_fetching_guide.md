# Fetching model lists: OpenRouter, Ollama (web), NVIDIA free and non-free

Practical reference for pulling each provider's catalogue, including how to tell a free NVIDIA model from a paid one.

Verified against the live services on 2026-11-07/08. Numbers are a snapshot, not a guarantee.

## 1. Overview & Ground Rules

| Provider | Endpoint | Auth for listing | What you get |
|---|---|---|---|
| OpenRouter | `https://openrouter.ai/api/v1/models` | none (key optional) | ~460 models with pricing, context, capabilities |
| Ollama (web) | `https://ollama.com/v1/models`, `https://ollama.com/api/tags` | none | ~26 cloud models (public catalogue) |
| NVIDIA free | `build.nvidia.com/models` (scrape) + `integrate.api.nvidia.com/v1/models` | none (key for chat) | free vs paid flag, publisher, description |
| NVIDIA non-free | same as above, filtered | none (key for chat) | the rest of the catalogue |

Where the keys come from:

| Provider | Key page |
|---|---|
| OpenRouter | <https://openrouter.ai/keys> |
| Ollama (web) | <https://ollama.com/settings/keys> |
| NVIDIA | <https://org.ngc.nvidia.com/account/api-keys> |

### Foundational Rules

- **Send keys in a header, never in a URL:** `Authorization: Bearer <key>`.
- **Sort by name yourself.** Providers do not guarantee a stable order; Ollama's order changed between identical requests.
- **Treat every list as a snapshot.** NVIDIA retires models; OpenRouter adds them.
- **A listed model is not a callable model.** All three providers serve their model list publicly, and none filters it by your account. Callability has to be tested separately (see "Testing Callability" below).
- Read the response with a timeout (20-30 s) and a redirect handler.
- If `Invoke-RestMethod` fails with a TLS/schannel error, the shell is restricted (some sandboxes block it). Use `curl.exe` or the Node snippet instead — the endpoints are unchanged.
- **On Windows, write `curl.exe`, not `curl`.** PowerShell 5.1 aliases `curl` to `Invoke-WebRequest`, so `curl -s` fails there.

---

## 2. Provider Deep-Dives

### 2.1 OpenRouter

OpenRouter acts as a massive aggregator. Their `/models` endpoint is highly reliable, but the sheer volume of models requires careful pagination and capability filtering.

#### Fetching the Catalogue

> **Docs vs. Reality: Authentication**
> OpenRouter's official documentation includes the `Authorization: Bearer <key>` header in their listing examples. However, in practice, the endpoint is public and returns the exact same list whether you provide a valid key, a bogus key, or no key at all. A key is only strictly required for inference (chat).

```bash
# Works without a key
curl -s https://openrouter.ai/api/v1/models
```

```powershell
Invoke-RestMethod https://openrouter.ai/api/v1/models |
  Select-Object -ExpandProperty data |
  Select-Object id, name, context_length
```

#### Handling Pagination

The default response contains everything (460 items, `total_count: 460`), but the endpoint also accepts `limit` and `offset`. Verified: `?limit=5` returns 5 items, `?offset=450` returns the last 10. If you use `limit`, page until `offset >= total_count`:

```javascript
const all = [];
let offset = 0;

while (true) {
  const page = await getJson(`https://openrouter.ai/api/v1/models?limit=100&offset=${offset}`);
  all.push(...page.data);
  offset += page.data.length;
  if (offset >= page.total_count || page.data.length === 0) break;
}
```

#### Identifying Free Models

Two signals are needed, because the `:free` suffix is not applied to every zero-priced model.

```javascript
const isFree = (m) => {
  if (/:free$/i.test(m.id)) return true;
  // Optional chaining prevents TypeError if m.pricing is undefined
  return Number(m.pricing?.prompt) === 0 && Number(m.pricing?.completion) === 0;
};
```

#### Filtering by Capability

`supported_parameters` is not decoration — `temperature` is absent on roughly 112 of 460 models, and sending an unsupported field can be rejected.

```javascript
const payload = { model: m.id, messages };
if (m.supported_parameters.includes('temperature')) payload.temperature = 0.7;
if (m.supported_parameters.includes('top_p')) payload.top_p = 0.95;

if (m.supported_parameters.includes('max_tokens')) {
  payload.max_tokens = 512;
} else if (m.supported_parameters.includes('max_completion_tokens')) {
  payload.max_completion_tokens = 512;
}
```

#### Provider Gotchas
- Prices are **per token** strings; multiply by 1e6 for "per 1M tokens".
- `total_count` lets you check you received everything in one page.
- `:free` models exist; whether they are rate-limited differently is provider policy.

---

### 2.2 Ollama (web subscription)

This is the hosted service at `ollama.com`, not `localhost:11434`. 

> **Docs vs. Reality: Endpoints**
> Ollama's official documentation only references `localhost:11434`. Hitting the hosted `ollama.com` endpoints is an undocumented, unsupported reality. However, it uses the exact same API shape as a local install.

#### Fetching the Catalogue

Two endpoints return the same catalogue. Both return the **same 26 ids** (set-equal), differing only in shape and order.

```bash
curl -s https://ollama.com/v1/models     # OpenAI-compatible
curl -s https://ollama.com/api/tags      # native Ollama format
```

```text
# Shape of responses
/v1/models  -> { "object":"list", "data":[ { "id", "object", "created", "owned_by" } ] }
/api/tags   -> { "models":[ { "name", "model", "modified_at", "size", "digest", "details":{} } ] }
```

Listing works **without a key**, and also with a bogus key — same 26 models. It is the public cloud catalogue, not your account's model list. Do not try to derive entitlement from it. Keys are required for inference (Anthropic-compatible: `/v1/messages`, Native: `/api/chat`).

#### Provider Gotchas
- `details` (parameter size, family, quantisation) is **empty for cloud models**; do not build a UI that depends on it.
- Order varies between requests. Sort it yourself.
- Model names may contain a colon tag (`mistral-large-3:675b`). Keep it.

---

### 2.3 NVIDIA (Free and Paid)

NVIDIA splits its catalog across two places, and neither one alone is enough.

> **Docs vs. Reality: UI Badges**
> The official API has **no** free/paid flag. NVIDIA's documentation confirms access tiers are UI concepts. Scraping the website is the only way to map this, but be aware this requires maintenance if their UI framework changes.

#### Step 1: The ID List

**No key is required** for listing, though a key is required for chat (`401` without it).

```bash
curl -s https://integrate.api.nvidia.com/v1/models
```

This returns 84 entries. This is the public catalogue, **not** your account's entitlement.

#### Step 2: Scraping for Free Badges

The free/paid status lives on the website as a badge on each model card in the HTML. The loop is: request page *n*, stop when the status is not 200 **or** when the page contributes no new model cards.

*Note: Stop on a non-200 status, not just on an empty page. Past the last page, the site answers `202 Accepted`, which is not an error but means "stop and retry later".*

```javascript
const stripTags = s => String(s || '')
  .replace(/<[^>]*>/g, ' ')
  .replace(/&nbsp;/g, ' ').replace(/&amp;/g, '&').replace(/&lt;/g, '<')
  .replace(/&gt;/g, '>').replace(/&quot;/g, '"')
  .replace(/&#x27;|&#39;/g, "'").replace(/&#(\d+);/g, (_, d) => String.fromCharCode(+d))
  .replace(/\s+/g, ' ').trim();

function parseCards(html) {
  const out = [];
  for (const block of html.split(/<li\b/)) {
    if (!block.includes('nv-card-root')) continue;
    const b = block.split('</li>')[0];

    const h3 = b.match(/<h3[\s\S]*?<\/h3>/);          
    const path = (h3 ? h3[0] : b).match(/href="\/([^"/?#]+)\/([^"/?#]+)"/);
    if (!path) continue;

    const badges = [...b.matchAll(/nv-badge[^>]*>([\s\S]*?)<\//g)]
      .map(m => stripTags(m[1]))
      .filter(Boolean);                                

    out.push({
      publisher: path[1],
      slug: path[2],
      sitePath: `/${path[1]}/${path[2]}`,
      name: stripTags((b.match(/nv-text--body-bold-xl[^>]*>([\s\S]*?)<\//) || [,''])[1]),
      description: stripTags((b.match(/nv-text--label-regular-md[^>]*>([\s\S]*?)<\//) || [,''])[1]),
      free: badges.some(t => /free endpoint/i.test(t)),
      downloadable: badges.some(t => /downloadable/i.test(t)),
      deprecated: badges.some(t => /deprecat/i.test(t))
    });
  }
  return out;
}
```

#### Step 3: Join the Lists

Website slugs and API ids differ only by `_` vs `.` (`nemotron-3_5-embed-1b` vs `nemotron-3.5-embed-1b`). Normalise before matching:

```javascript
const norm = s => s.toLowerCase().replace(/_/g, '.');
const apiById = new Map(apiData.map(m => [norm(m.id), m]));   

const key  = card.sitePath.replace(/^\//, '');       
const api  = apiById.get(norm(key));                 
```

* `free === true && m.id`: Free and listed. The ideal default view.
* `free === false`: Paid models (published card without the free badge).
* `free === null`: The id is in `/v1/models` but no website card matched. **Do not guess** their status.

#### Testing Callability

A free badge does not mean your account can run the model. One 4-token request per model settles it:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  https://integrate.api.nvidia.com/v1/chat/completions \
  -H "Authorization: Bearer $NGC_API_KEY" -H "Content-Type: application/json" \
  -d '{"model":"nvidia/nemotron-3.5-bolt-e4b","messages":[{"role":"user","content":"ping"}],"max_tokens":4}'
```

| Status | Meaning |
|---|---|
| 200 | callable |
| 401 / 403 | key problem, not a model problem |
| 404 | not enabled for this account |
| 410 | retired by NVIDIA |
| 429 | rate limited — retry later, not a verdict |
| ECONNRESET | treat as unknown, not as "no" |

---

## 3. Utility Scripts & Recipes

### Minimal Node Fetcher

```javascript
const https = require('https');

function getJson(url, headers = {}) {
  return new Promise((resolve, reject) => {
    const req = https.request(url, { headers: { Accept: 'application/json', ...headers } }, res => {
      if ([301, 302, 307, 308].includes(res.statusCode) && res.headers.location) {
        return getJson(new URL(res.headers.location, url).toString(), headers).then(resolve, reject);
      }
      let body = '';
      res.on('data', c => (body += c));
      res.on('error', reject);
      res.on('end', () => {
        if (res.statusCode !== 200) return reject(new Error(`HTTP ${res.statusCode}`));
        try { resolve(JSON.parse(body)); } catch (e) { reject(e); }
      });
    });
    req.on('error', reject);
    req.setTimeout(30000, () => req.destroy(new Error('timeout')));
    req.end();
  });
}

(async () => {
  // OpenRouter (Auth optional for listing)
  const or = await getJson('https://openrouter.ai/api/v1/models');
  
  // Ollama web
  const ol = await getJson('https://ollama.com/v1/models');
  
  // NVIDIA ids (Auth optional for listing, but shown here for completeness)
  const nv = await getJson('https://integrate.api.nvidia.com/v1/models',
    { Authorization: `Bearer ${process.env.NGC_API_KEY}` });
})();
```

### Quick Recipes

```javascript
// Sort every list before rendering:
list.sort((a, b) => String(a.name || a.id).localeCompare(String(b.name || b.id)));

// NVIDIA: free and listed
const freeListed = merged.filter(m => m.free === true && m.id);

// NVIDIA: everything that is not free (paid + unknown)
const rest = merged.filter(m => m.free !== true);

// OpenRouter: tool-capable models with a large context
const capable = or.data.filter(m =>
  Array.isArray(m.supported_parameters) && m.supported_parameters.includes('tools') &&
  (m.context_length || 0) >= 128000);

// OpenRouter: cheapest non-free chat models
const cheap = or.data
  .filter(m => Number(m.pricing?.prompt) > 0 && /text->text/.test(m.architecture?.modality || ''))
  .sort((a, b) => Number(a.pricing.prompt) - Number(b.pricing.prompt))
  .slice(0, 20);

// Ollama: names ready to send as `model`
const ids = (await getJson('https://ollama.com/v1/models')).data.map(m => m.id);
```

---

## 4. Verification Status

Measured live on 2026-11-08 — every number in this document comes from these runs:

- **OpenRouter**: 460 items, `total_count` 460, 16 free, 112 without `temperature`, 454 with `max_completion_tokens`, `cache-control: max-age=120`, `?limit=5` → 5 and `?offset=450` → 10, listing 200 with a bogus key.
- **Ollama**: both endpoints 200, same 26 ids, `details` empty, order varies between calls, listing 200 with a bogus key, chat 401 without a key on both `/v1` and `/api`.
- **NVIDIA**: `/v1/models` 200 with no key, a bogus key and a valid key. Site: 96 + 9 model cards, page 3 = 202, page 4 = 200 with 0, ~1.69 MB per page. Card badges seen: `Free Endpoint`, `Downloadable`, `Deprecation in 9d`.
- **Merged**: 161 = 84 API ids + 77 site-only cards; 45 free, 60 not free, 56 unknown; 23 free + listed.
- **Callability**: 13 of 23 answered, 4 × 404, 6 × timeout.

### Snapshot Summary
```text
OpenRouter        460 models   public list, 16 free, pricing in USD/token
Ollama (web)       26 models   public catalogue, not account-scoped
NVIDIA API         84 ids      public list (no key needed), no free/paid flag
NVIDIA site       105 cards    96 + 9 model cards over two pages (+1 promo per page)
NVIDIA merged     161 entries  45 free · 60 not free · 56 unknown · 77 site-only
NVIDIA callable    13-14 of 23 free+listed models answered a ping (run-dependent)
```