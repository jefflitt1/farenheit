# farenheit

**Geographic price transparency for digital products.**

Find out how much a product costs when searched from different countries — and how much you could save.

---

## Usage

```
/farenheit [product name]                  — look up a single product (international geo-pricing)
/farenheit --category [category]           — scan a full category
/farenheit --all                           — scan everything in the catalog
/farenheit [product] --markets IN,TR,AR    — specific markets only
/farenheit --proxy [product]               — v2b: live scrape IP-based products via Firecrawl geo-routing
/farenheit goodrx lisinopril --zip 10001,60601,90210  — v2a: intra-US zip comparison
/farenheit amazon [ASIN] --zip 10001,78701             — v2a: Amazon price by delivery zip
/farenheit --category domestic --zip 10001,60601,90210 — v2a: scan all domestic products
```

**Categories:** `streaming` · `saas` · `software` · `cloud` · `gaming` · `hardware` · `domestic`

**Examples:**
```
/farenheit spotify
/farenheit adobe creative cloud
/farenheit --category gaming
/farenheit --all
/farenheit xbox game pass --markets IN,TR,AR,US
/farenheit --proxy notion
/farenheit goodrx lisinopril --zip 10001,60601,90210,78701,98101
/farenheit --category domestic --zip 10001,60601,90210
```

---

## How to Execute This Skill

### Step 1 — Parse the request and route

Read the user's input and determine:
- **Mode:** single product, category scan, or full catalog
- **Product match:** fuzzy match against `catalog.json` by `name`, `id`, or `vendor`
- **Markets:** default to all markets in `markets.json`; override if user specifies `--markets`

If no product match is found, say so clearly and list similar entries from the catalog.

**Then apply routing logic** — determine which variance type applies before fetching anything:

| If the product/category is... | Route to | How |
|---|---|---|
| Streaming, Gaming, SaaS, Software | **International** | Steps 3-5 below (Firecrawl + FX) |
| `domestic` category products | **Intra-US zip (v2a)** | Step 4b below — `--zip` required |
| Insurance, Home services, Healthcare | **Intra-US zip (v2a)** | Step 4b below — `--zip` required |
| E-commerce (Amazon), Gig economy | **Both** | International first, then domestic with `--zip` |
| Hardware (Apple, Samsung) | **International + note sales tax** | Steps 3-5, add import duty note |
| Cloud/infra (AWS, Vercel, DO) | **Control** | Reference data only, flag as low-variance |
| `--proxy` flag present | **Firecrawl geo-routing (v2b)** | Step 4c below — overrides `live: false` |

**If `variance_type: domestic` with no `--zip` flag:** Prompt the user:
> "⚠ [Product] pricing is primarily domestic (US zip-code level). Add `--zip` with comma-separated zip codes to compare. Example: `--zip 10001,60601,90210,78701,98101`"

**If `variance_type: both` (Apple hardware, e-commerce):** Run the international comparison first (Steps 3-5). After the international table, note: "💡 This product also varies by US delivery zip. Add `--zip 10001,60601,...` for an intra-US comparison." If the user provided `--zip` in the same invocation, run Step 4b immediately after the international output and show both tables.

### Step 2 — Load data files

Read both data files:
- `~/.claude/skills/farenheit/markets.json`
- `~/.claude/skills/farenheit/catalog.json`

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

**For `live: false` products** — use reference data directly:
1. Read `reference` object from catalog entry
2. Note `reference_date` — display as "(ref: [date])"
3. No scraping needed — instant output

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

### Step 4c — v2b: Firecrawl geo-routing (`--proxy` flag)

Use this path when: `--proxy` flag is present OR user wants live data for a `live: false` product.

**What this enables:** Products currently marked `live: false` use IP-based detection and can't be scraped with a plain URL. Firecrawl's built-in `location` parameter routes the request through a geo-specific IP — no external proxy account needed.

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

**Which products benefit most from `--proxy`:** All `live: false` products in `saas`, `streaming` categories — Notion, Grammarly, Canva, Zoom, Hulu, Peacock, Paramount+. These are IP-detected services that previously could only show reference data.

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
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  farenheit ── Spotify Premium (Individual)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Market          Local Price    USD/mo    vs. US
  ─────────────────────────────────────────────────
  🇦🇷 Argentina   ARS$369       $0.41    -97%  ✓
  🇹🇷 Turkey      ₺39.99        $1.27    -91%
  🇮🇳 India       ₹119          $1.43    -90%
  🇧🇷 Brazil      R$10.90       $2.01    -86%
  🇲🇽 Mexico      MX$59         $3.19    -77%
  🇵🇱 Poland      PLN 23.99     $6.10    -56%
  🇩🇪 Germany     €10.99        $12.03   +14%  ↑
  🇬🇧 UK          £11.99        $15.05   +57%  ↑
  🇺🇸 US          $13.99        $13.99   —  (baseline)
  ─────────────────────────────────────────────────
  💡 Cheapest: Argentina — save $13.58/mo ($163/yr)
  📡 Source: live scrape · FX: 2026-04-20
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

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
- **Product not in catalog:** Say so, suggest similar matches, offer to run with a direct URL if user provides one
- **FX fetch fails:** Proceed with approximate rates, add warning
- **Category has 0 live products:** Show reference data only, note this clearly

---

## Notes for contributors

The catalog lives at `~/.claude/skills/farenheit/catalog.json`. To add a product:

1. Determine if it's `live: true` (URL-addressable regional pages) or `live: false` (IP-based, needs reference data)
2. Add the entry to the appropriate category array
3. Test with `/farenheit [product name]`
4. Submit a PR to `jefflitt1/farenheit` on GitHub

The FX normalization and output formatting are handled by this skill — you only need to provide accurate URLs and reference prices.

---

## Phase 2 (not yet implemented)

- `--proxy` flag: route Firecrawl through residential proxy for IP-based products
- `--watch [product]`: alert when price changes (requires Supabase persistence)
- `--travel`: Amadeus API integration for flight/hotel geo-pricing
