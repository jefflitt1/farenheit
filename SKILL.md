# farenheit

**Geographic price transparency for digital products.**

Find out how much a product costs when searched from different countries — and how much you could save.

---

## Usage

```
/farenheit [any product name]                   — geo-pricing for any product (catalog or live discovery)
/farenheit --category [category]                — scan a full catalog category
/farenheit --all                                — scan everything in the catalog
/farenheit [product] --markets IN,TR,AR         — specific markets only
/farenheit goodrx [drug]                        — v2a: intra-US zip comparison (auto-selects zips)
/farenheit amazon [ASIN]                        — v2a: Amazon price by delivery zip (auto zips)
/farenheit --category domestic                  — v2a: scan all domestic products (auto zips)
/farenheit --portfolio [p1],[p2],[p3]           — aggregate savings across your subscription stack
/farenheit --add [url]                          — cache a product entry for faster future lookups
/farenheit --cached [product]                   — skip live scrape, show reference data only (fast, no proxy credits)
```

**Categories:** `streaming` · `saas` · `software` · `cloud` · `gaming` · `hardware` · `domestic`

**Examples:**
```
/farenheit spotify
/farenheit cursor
/farenheit perplexity pro
/farenheit chatgpt plus
/farenheit adobe creative cloud
/farenheit --category gaming
/farenheit --portfolio spotify,adobe creative cloud,notion,grammarly
/farenheit --add https://linear.app/pricing
```

---

## How to Execute This Skill

### Step 1 — Parse the request and route

Read the user's input and determine:
- **Mode:** single product, category scan, full catalog, portfolio, or --add
- **Catalog check:** fuzzy match against `catalog.json` by `name`, `id`, or `vendor` — **fast path if found, not required**
- **Markets:** default to all markets in `markets.json`; override if user specifies `--markets`

**If no catalog match:** Do NOT stop. Proceed directly to **Step 2.5 — Live Discovery**. The catalog is a cache of known URL patterns, not a gate on what products can be queried.

**Then apply routing logic** — determine which variance type applies before fetching anything:

| If the input is... | Route to | How |
|---|---|---|
| `--portfolio [products]` | **Portfolio aggregation (v3)** | Step 4d below |
| `--add [url]` | **Cache this entry** | Step 4e below — scrape, classify, write to catalog |
| Catalog hit — `live: true` | **URL-based scrape** | Steps 3-5 (Firecrawl + FX) |
| Catalog hit — `live: false` | **Firecrawl geo-routing (v2b)** | Step 4c — proxy always on for IP-detected products |
| **No catalog hit** | **Live Discovery** | Step 2.5 → Step 4f |
| `domestic` category products | **Intra-US zip (v2a)** | Step 4b — uses auto-zips from catalog |
| Insurance, Home services, Healthcare | **Intra-US zip (v2a)** | Step 4b — uses auto-zips from catalog |
| E-commerce (Amazon), Gig economy | **Both** | International first, then domestic |
| Hardware (Apple, Samsung) | **International + note sales tax** | Steps 3-5, add import duty note |
| Cloud/infra (AWS, Vercel, DO) | **Control** | Reference data only, flag as low-variance |
| `--cached` flag present | **Reference data** | Skip scrape, show cached reference + date |

**If `variance_type: domestic` with no `--zip` flag:** Prompt the user:
> "⚠ [Product] pricing is primarily domestic (US zip-code level). Add `--zip` with comma-separated zip codes to compare. Example: `--zip 10001,60601,90210,78701,98101`"

**If `variance_type: both` (Apple hardware, e-commerce):** Run the international comparison first (Steps 3-5). After the international table, note: "💡 This product also varies by US delivery zip. Add `--zip 10001,60601,...` for an intra-US comparison." If the user provided `--zip` in the same invocation, run Step 4b immediately after the international output and show both tables.

### Step 2 — Load data files

Read both data files:
- `~/.claude/skills/farenheit/markets.json`
- `~/.claude/skills/farenheit/catalog.json`

### Step 2.5 — Live Discovery (no catalog match)

**Trigger:** Run this step when Step 1 finds no catalog match for the requested product.

The catalog is a speed cache. Any product can be queried without a catalog entry — this step finds the pricing page and detects geo-variance live.

**2.5a — Find the pricing page:**

Use Firecrawl search to locate the canonical pricing URL:
```json
{ "query": "[product name] pricing", "limit": 3 }
```
Pick the most authoritative result (official domain, not resellers or review sites). If ambiguous, pick the first official-domain result.

**2.5b — Detect URL-based geo-variance:**

Test whether the product uses URL-addressable regional pricing by probing common patterns:
1. Try appending `/in/`, `/tr/`, `/br/` to the base pricing URL — or try `?country=IN`, `?region=IN`
2. Call `mcp__firecrawl__firecrawl_scrape` on the US URL and one variant (e.g. `/in/`)
3. **If the variant returns different prices → `live: true` path.** Use Step 4 (URL scraping across all markets).
4. **If the variant returns the same price or 404 → `live: false` path.** Use Step 4f (geo-proxy detection).

**2.5c — Classify the product:**

From the pricing page content, determine:
- `category` — streaming · saas · software · gaming · hardware · cloud · domestic
- `plan` — the most common paid individual tier
- Expected variance type — use the routing table from Step 1 to infer (e.g. SaaS → international)

Proceed directly to Step 3 (FX rates) then Step 4 or 4f as appropriate. No need to write a catalog entry — just run the comparison and show results.

### Step 3 — Get current exchange rates

Fetch exchange rates from Frankfurter. **Use the v1 endpoint directly** — the legacy domain redirects and WebFetch may not follow it:

```
GET https://api.frankfurter.dev/v1/latest?from=USD&to=INR,BRL,MXN,TRY,PLN,THB,PHP,NGN,GBP,EUR
```

Note: **ARS (Argentine Peso) is intentionally excluded** — Frankfurter does not carry ARS due to Argentina's exchange rate controls. Handle separately (see below).

Parse the `rates` object. Store as your FX table for this session.

**FX conversion formula:** `usd_price = local_price / rate[currency]`

**ARS handling:** When Argentina price is scraped, display local price only and append:
> `* ARS — Frankfurter doesn't carry this rate (exchange controls). At ~[current official rate] ARS/USD: ≈$[estimate]. Verify at xe.com.`
Use ~1,100 ARS/USD as a rough estimate if no better source available; flag it clearly as approximate.

**MXN ambiguity guard:** Mexico uses `$` as its peso symbol. When scraping Mexican pages, the extracted price `$139` means MX$139 (pesos), not USD. Always check the page's locale metadata (`og:locale: es-Latn-MX`) to confirm currency before converting. Convert MXN using the MXN rate, never treat as USD.

If Frankfurter is unavailable, proceed with a warning: "⚠ Live FX unavailable — showing approximate USD conversions."

### Step 4 — Fetch prices (per product)

**For `live: true` products** — use Firecrawl to scrape each regional URL:

For each market in the product's `markets` array:
1. Resolve the URL using `url_pattern` or `override_urls`
   - Replace `{country_path}` with `markets.json → country_path` for that market
   - Replace `{country_path_lower}` with the lowercase version
2. Call `mcp__firecrawl__firecrawl_scrape` with:
   ```json
   { "url": "[resolved_url]", "formats": ["markdown"] }
   ```
3. From the returned markdown, extract the price for the target plan using this prompt logic:
   > "Find the monthly or annual price for [plan name] on this pricing page. Return ONLY: currency symbol + numeric amount + billing period (e.g. '$12.99/mo' or '₹159/mo'). If multiple plans exist, use [plan]. If not found, return NULL."
4. Convert to USD using your FX table.

**Rate limit:** Add a brief pause between Firecrawl calls when scanning multiple products to avoid hammering.

**For `live: false` products** — these are IP-detected; route directly to Step 4c (geo-proxy). Reference data is never shown by default.

**If `--cached` flag is present:** Skip Step 4c. Read `reference` object from catalog entry and display with `(ref: [date])` label. No scraping.

**For `control: true` products** — display with a 🔒 label indicating no geo-pricing detected.

---

### Step 4b — v2a: Intra-US zip mode (`--zip` flag or `domestic` category)

Use this path when: `variance_type: domestic`, `--zip` flag is present, or user asks about GoodRx/Amazon/Instacart/DoorDash/insurance/home services by zip.

**No FX conversion needed** — all prices are USD. Skip Step 3 (FX fetch) for domestic-only lookups.

**Zip selection — always automatic, never ask the user:**

Each domestic product in `catalog.json` has a `default_zips` array — a curated set chosen to maximize variance for that product's specific `zip_variance_axis`. Always use these. The user never needs to provide zip codes.

If the user provides `--zip` override, use theirs instead. Otherwise: use `default_zips` from the catalog entry, no prompt.

**Method A — `url_param_zip` products (GoodRx):**

GoodRx embeds zip as a URL parameter — no browser automation required:
1. Read `default_zips` from the catalog entry (or user `--zip` override)
2. For each zip: build URL from `url_pattern` with `{zip}` and `{drug_name}` replaced
3. Call `mcp__firecrawl__firecrawl_scrape` on each URL with `{ "formats": ["markdown"] }`
4. Extract: lowest cash price shown + pharmacy name
5. Build comparison table sorted cheapest to most expensive

**Method B — `browser_zip` products (Amazon, Instacart, DoorDash, insurance, home services):**

These require form interaction. Use `unified-browser` MCP (Playwright):

For each zip in `default_zips` (or user override):
1. `mcp__unified-browser__browser_navigate` → product URL
2. Follow the `browser_steps` string from the catalog entry
3. Use `mcp__unified-browser__browser_type` to enter zip, `browser_click` to submit
4. `mcp__unified-browser__browser_content` or `browser_snapshot` to read the displayed price
5. Record price + city label (from `default_zips[].city`), move to next

**Output format for zip mode:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  farenheit ── GoodRx: lisinopril (10mg, 30 tabs)
  Comparing: pharmacy chain density + competition
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  City                ZIP    Lowest price   Pharmacy
  ─────────────────────────────────────────────────
  Austin, TX          78701  $4.00          Walmart   ✓
  Chicago, IL         60601  $6.50          CVS
  Des Moines, IA      50301  $7.20          Walgreens
  Miami, FL           33101  $9.80          Navarro
  New York, NY        10001  $11.40         Duane Reade
  Starkville, MS      39759  $14.20         independent  ↑
  ─────────────────────────────────────────────────
  💡 Cheapest: Austin — $10.20 less than Starkville
  📡 Source: live scrape · All prices USD
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Note: include the `zip_variance_axis` from the catalog as the subtitle line ("Comparing: ...") so users understand why these specific cities were chosen.

---

### Step 4c — Firecrawl geo-routing (default for `live: false` products)

Use this path when: product is `live: false` (IP-detected pricing) and `--cached` flag is NOT present.

**What this does:** IP-detected products (Notion, Grammarly, Canva, Zoom, Netflix, Hulu, etc.) serve prices based on the requester's IP, not the URL path. Plain scraping always returns US prices. Firecrawl's built-in `location` parameter routes the request through a geo-specific residential IP — no external proxy account needed.

For each market in the product's `markets` array:
1. Look up the `country` code from `markets.json` (use `code` field, e.g. `"IN"`)
2. Look up the `locale` from `markets.json` (e.g. `"en-IN"`)
3. Call `mcp__firecrawl__firecrawl_scrape` with geo-routing parameters:
   ```json
   {
     "url": "[product's main pricing URL]",
     "formats": ["markdown"],
     "location": {
       "country": "IN",
       "languages": ["en-IN"]
     },
     "proxy": "stealth"
   }
   ```
4. Extract price from returned markdown using the same prompt logic as Step 4
5. Convert to USD using FX table

**Note on `proxy: "stealth"`:** This uses Firecrawl's residential proxy pool to make the request appear to originate from the target country. No Smartproxy, Bright Data, or external proxy subscription required — it's built into the Firecrawl MCP.

**Output:** Same format as international output (Step 6), but mark source as `📡 Source: live scrape (geo-proxy) · FX: [date]` instead of `(ref: [date])`.

**Which products use this path by default:** All `live: false` products in `saas`, `streaming` categories — Notion, Grammarly, Canva, Zoom, Hulu, Peacock, Paramount+. These are IP-detected services; Step 4c runs automatically for them.

### Step 4d — v3: Portfolio mode (`--portfolio`)

Use this path when: user invokes `/farenheit --portfolio [product1],[product2],...`

Fetch live FX rates once (Step 3), then run each product through Steps 4–5, reusing the same FX table. Collect results across all products and produce a single aggregated dashboard.

**Execution:**
1. Parse comma-separated product list; fuzzy-match each against `catalog.json`
2. Note any product not found — include in output as "not in catalog"
3. Skip `domestic` / `control` products from the savings total — note them separately
4. For each matched product: fetch prices (Step 4 / 4b / 4c as appropriate), calculate savings vs US baseline
5. Sort rows by `save_per_month` descending

**Output format:**
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  farenheit ── Portfolio (4 products)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Product               US/mo    Best deal            Save/mo   How to pay
  ─────────────────────────────────────────────────────────────────────
  Spotify Premium       $13.99   🇹🇷 Turkey   $1.27   $12.72    🟡 Gift card · Seagm
  Adobe Creative Cloud  $59.99   🇮🇳 India    $35.99  $24.00    🟡 Wise card (INR)
  Notion Plus           $16.00   🇮🇳 India    $5.00   $11.00    🟡 Wise card (INR)
  Grammarly Premium     $30.00   🇮🇳 India    $10.00  $20.00    🟡 Wise card (INR)
  ─────────────────────────────────────────────────────────────────────
  Total at US prices:   $119.98/mo   ($1,439.76/yr)
  Cheapest possible:    $52.26/mo    ($627.12/yr)
  💰 You're leaving:   $67.72/mo    ($812.64/yr) on the table
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Run /farenheit [product] for full market breakdown.
  📡 FX: [date]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Notes:**
- Cheapest market per product may differ — show each independently, never blend
- Reference data products show `(ref: [date])` in the best deal cell
- "How to pay" column uses the Payment Method Matrix (see below, before Step 6)

---

### Step 4e — v3: Auto-catalog builder (`--add`)

Use this path when: user invokes `/farenheit --add [url]`

Reads a pricing page, classifies it, checks for duplicates, and outputs a ready-to-paste `catalog.json` entry.

**Execution:**

1. **Scrape the pricing page:**
   ```json
   { "url": "[provided url]", "formats": ["markdown"], "onlyMainContent": true }
   ```

2. **Classify from the scraped content:**
   - `name` / `vendor` — extract from page title or branding
   - `category` — one of: `streaming` · `saas` · `software` · `gaming` · `hardware` · `cloud` · `domestic`
   - `plan` — the most common paid individual tier (e.g. "Plus", "Pro", "Individual")
   - `live` — does this product use URL-addressable regional pricing?
     - Evidence for `live: true`: URL contains `/in/`, `/tr/`, `?country=`, `?region=`; pricing page loads country-specific content from URL alone
     - Evidence for `live: false`: single global URL, prices shown based on visitor IP; no country path variation
   - `markets` — default to `["US", "IN", "TR", "AR", "BR", "MX"]` unless page gives stronger signal
   - `control: true` — only if pricing is explicitly stated as globally fixed (e.g. "same price worldwide")

3. **For `live: true` products:** Derive `url_pattern` by testing one non-US variant (e.g. scrape `[base]/in/pricing`). If it returns different pricing → pattern confirmed. Set placeholder: `{country_path}`.

4. **For `live: false` products:** Extract US price from scraped page. Other markets will be fetched live via Step 4c (geo-proxy) when the product is queried.

5. **Duplicate check:** Fuzzy-match product name against existing catalog. If a close match is found, say so before showing the generated entry.

6. **Display generated entry:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  farenheit --add
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Product:   Linear
  Vendor:    Linear Orbit Inc.
  Category:  saas
  Mode:      live: false  (IP-detected pricing)
  Plan:      Plus ($8/mo US)
  Markets:   IN, TR, AR, BR, MX, US (default — verify)

  📋 Paste into catalog.json → "saas" array:

  {
    "id": "linear",
    "name": "Linear",
    "vendor": "Linear",
    "category": "saas",
    "plan": "Plus",
    "live": false,
    "variance_type": "international",
    "markets": ["US", "IN", "TR", "AR", "BR", "MX"],
    "url": "https://linear.app/pricing",
    "reference": {
      "US": { "local": "$8", "usd": 8.00, "currency": "USD" }
    },
    "reference_date": "2026-04-20",
    "payment_notes": null
  }

  ⚠ live: false — IP-detected pricing. Live prices fetched automatically via Firecrawl
    geo-proxy on next /farenheit linear invocation. Add --cached to skip live scrape.

  To contribute: copy JSON → append to catalog.json → submit PR to jefflitt1/farenheit
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### Step 4f — Live geo-comparison (no catalog entry, IP-based product)

**Trigger:** Product found via Step 2.5, URL probing confirmed no regional URL variants (`live: false` path).

This is the geo-proxy path for any product not in the catalog. Use Firecrawl's built-in location routing to scrape the same URL from multiple country IPs and compare the returned prices.

**Markets to probe (default set — prioritized for variance signal):**
`US · IN · TR · AR · BR · MX`
Skip markets where the product is unlikely to be sold (e.g. don't probe AR for enterprise SaaS with custom pricing).

**For each market:**
```json
{
  "url": "[pricing URL from Step 2.5]",
  "formats": ["markdown"],
  "onlyMainContent": true,
  "location": { "country": "IN", "languages": ["en-IN"] },
  "proxy": "stealth"
}
```
Swap `country` and `languages` per market. Run markets in parallel where Firecrawl rate limits allow.

**Extract price:** From each returned page, identify the target plan's price. Use the same plan across all markets (the most common paid individual tier identified in Step 2.5c).

**Classify the result:**
- **Variance detected** → show full comparison table (Step 6 format), source: `live scrape (geo-proxy)`
- **All markets same price** → classify as control: `🔒 [Product] — $X/mo globally. No geo-pricing detected.`
- **Some markets return different currency/price** → show table with FX conversion (Step 5)
- **Market returns error/blocked** → skip that market, note in output

**After showing results:** If meaningful variance is found (>10% difference anywhere), offer:
> `💾 Add to catalog? Run /farenheit --add [url] to save this entry for faster future lookups.`

Do not auto-write to catalog — wait for explicit `--add` invocation.

---

### Payment Method Matrix

Used by Step 6 and Step 4d to populate the "How to pay" column. Apply in order: category default → market modifier → product-specific override.

**Category defaults:**

| Category | Method | Risk | Source |
|----------|--------|------|--------|
| `gaming` | Gift card | 🟢 LOW | Seagm · G2G · Eneba |
| `streaming` | Gift card | 🟡 MEDIUM | Seagm · G2G |
| `software` | Wise/Revolut card (local currency billing) | 🟡 MEDIUM | wise.com · revolut.com |
| `saas` | Wise/Revolut card or VPN at signup | 🟡 MEDIUM | wise.com |
| `hardware` | Import — no easy arbitrage | 🔴 HIGH | — |
| `cloud` | No geo-pricing | 🔒 N/A | — |

**Market risk modifiers:**

| Market | Modifier | Reason |
|--------|----------|--------|
| Argentina (AR) | +1 risk level | Accounts reset ~every 6mo; economic volatility |
| Turkey (TR) | +0 | Stable, widely used for arbitrage |
| India (IN) | +0 | Lowest friction; Wise INR card accepted by most Stripe merchants |
| Brazil (BR) | +0 | Stable |
| Others | +0 | Default |

**Product-specific overrides (check `payment_notes` in catalog entry if present):**
- **Netflix:** 🔴 HIGH — aggressive geo-detection; requires local payment method, not just VPN
- **Spotify + Argentina:** 🟡 MEDIUM — gift cards available but accounts reset ~6mo
- **JetBrains:** 🟢 LOW — currency param accepted at checkout with any card
- **Adobe CC:** 🟡 MEDIUM — Wise INR card reliable via Stripe BIN check

**Format for "How to pay" cell:**
`[emoji] [method] · [source]`

Examples:
- Xbox + Turkey: `🟢 Gift card · Seagm/G2G`
- Spotify + Argentina: `🟡 Gift card (resets ~6mo) · Seagm`
- Adobe + India: `🟡 Wise card (INR billing)`
- Netflix + any: `🔴 Local payment + VPN required`

---

### Step 5 — Calculate and rank

For each product with prices across markets:

1. Identify baseline (US price in USD)
2. For each other market:
   - `delta_usd = market_usd - us_usd`
   - `delta_pct = (delta_usd / us_usd) * 100`
3. Sort markets from cheapest to most expensive
4. Flag the cheapest market with ✓
5. Flag any market >10% MORE expensive than US with ↑

### Step 6 — Output

**Single product output:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  farenheit ── Spotify Premium (Individual)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Market          Local        USD/mo    vs. US    How to pay
  ──────────────────────────────────────────────────────────────────────
  🇦🇷 Argentina   ARS$369      ~$0.41   -97%  ✓   🟡 Gift card (resets ~6mo) · Seagm
  🇹🇷 Turkey      ₺39.99       $1.27    -91%      🟡 Gift card · Seagm/G2G
  🇮🇳 India       ₹119         $1.43    -90%      🟡 Gift card · Seagm
  🇧🇷 Brazil      R$10.90      $2.01    -86%      🟡 Gift card · Seagm
  🇲🇽 Mexico      MX$59        $3.19    -77%      🟡 Gift card · Seagm
  🇵🇱 Poland      PLN 23.99    $6.10    -56%      🟡 Gift card · Seagm
  🇩🇪 Germany     €10.99       $12.03   +14%  ↑   —
  🇬🇧 UK          £11.99       $15.05   +57%  ↑   —
  🇺🇸 US          $13.99       $13.99   —  (baseline)  —
  ──────────────────────────────────────────────────────────────────────
  💡 Cheapest: Argentina — save $13.58/mo ($163/yr)
  📡 Source: live scrape · FX: 2026-04-20
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**"How to pay" column rules:**
- Omit `—` for markets that are MORE expensive than US (no point in paying more)
- Use the Payment Method Matrix above to derive the cell value
- Only include a source link where a specific platform is recommended (Seagm, G2G, wise.com)

**Category scan output:** Show each product as a condensed row:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  farenheit ── Gaming (5 products)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Product                    US Price  Best Deal          Savings
  ─────────────────────────────────────────────────────────────
  Xbox Game Pass Ultimate    $19.99    🇹🇷 Turkey $2.30   -88%  ✓
  PlayStation Plus Essential $79.99/yr 🇦🇷 Argentina $12  -85%
  Nintendo Switch Online     $19.99/yr 🇮🇳 India $2.50   -87%
  Steam (varies per game)    —         🇹🇷 Turkey -70%+  varies
  ─────────────────────────────────────────────────────────────
  Run `/farenheit [product]` for full market breakdown.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Control products** display with a note:
```
🔒 GitHub Pro — $4.00/mo globally. No geo-pricing detected.
🔒 Vercel Pro — $20.00/mo globally. No geo-pricing detected.
```

### Step 7 — Add context (for single product lookups)

After the table, add 1-2 sentences of useful context:
- Is this a well-known geo-pricing case?
- Any practical gotcha (e.g. "Argentina pricing requires an Argentine payment method")
- If reference data: "Run with `--live` flag to get current prices via web scrape"

### Handling errors

- **Firecrawl can't parse the page:** Skip that market, note "price not found" in the table row
- **Product not in catalog:** Do NOT stop — run Step 2.5 live discovery instead
- **Live discovery can't find pricing page:** Say so explicitly, ask user to provide the URL directly
- **Geo-proxy returns blocked/CAPTCHA:** Skip that market, note "geo-blocked" in the table row
- **FX fetch fails:** Proceed with approximate rates, add warning
- **Category has 0 live products:** Show reference data only, note this clearly
- **All markets same price:** Classify as control `🔒`, do not show a savings table

---

## Notes for contributors

The catalog lives at `~/.claude/skills/farenheit/catalog.json`. The fastest way to add a product:

```
/farenheit --add [pricing page url]
```

This auto-classifies the product, derives the URL pattern, and outputs a ready-to-paste JSON entry. Then:

1. Paste the generated entry into `catalog.json` under the correct category array
2. Add `payment_notes` if there's a product-specific gotcha not covered by the Payment Method Matrix
3. Test with `/farenheit [product name]`
4. Submit a PR to `jefflitt1/farenheit` on GitHub

The FX normalization, risk scoring, and output formatting are handled by this skill — you only need accurate URLs and reference prices.

---

## Roadmap

- **v1:** International geo-pricing — 32 products, 12 markets, live scraping + reference data ✓
- **v2a:** `--zip` mode — intra-US pricing via Playwright + Firecrawl (6 domestic products) ✓
- **v2b:** Firecrawl geo-routing for IP-based products ✓
- **v3:** `--portfolio`, `--add`, risk-scored output ✓
- **v4:** Open-world mode — catalog is a cache, not a gate; live discovery for any product via Firecrawl search + geo-proxy ✓
- **v1.3:** Geo-proxy is now always-on for `live: false` products (no `--proxy` flag); `--cached` flag for reference-data-only reads ✓
- **v5:** `--watch` mode — price change alerts (requires Supabase persistence)
- **v6:** `--travel` mode — Amadeus API for flight/hotel geo-pricing
