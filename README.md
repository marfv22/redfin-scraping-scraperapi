# Scrape Redfin Without Getting Blocked: Why Plain HTTP Requests Fail, Which Sort Orders Redfin Rejects, and How Per-Request Credits Are Really Charged (Field-Level Output Guide)

Redfin serves search results from a JSON endpoint, not from the HTML your browser renders. That one fact decides most of your architecture: no headless browser, no DOM selectors for list pages, just a payload you parse. It also explains the failures people post about. The endpoint sits behind anti-scraping protection, it answers with a `{}&&` prefix that breaks `json.loads`, and it behaves differently depending on which IP the request leaves from.

Below: the four ways people actually pull Redfin listings, six traps that quietly break pipelines, what the output should look like field by field, and the credit rules that decide which ScraperAPI tier you need.

> Most developers should start on ScraperAPI's free plan (1,000 API credits, max 5 concurrent connections) or the 7-day trial (5,000 credits), point it at one Redfin city URL, and price a real Redfin request with the Domain Cost Estimator before paying for anything. Redfin sits behind bot protection, and bypassing that adds credits, so picking a paid tier before you know the per-request cost is guesswork. 👉 [See the current ScraperAPI tiers and credit allowances](https://www.scraperapi.com/pricing?fp_ref=coupons)

## ScraperAPI plans at a glance

ScraperAPI's pricing FAQ names eight tiers. What each one costs per month and how many credits it carries changes with billing cycle, and the pricing page is the only place with live numbers. What does not change is how each tier behaves when you run out.

| Plan | Auto-upgrade when credits run out | Pay-as-you-go past your limit | Current price |
| --- | --- | --- | --- |
| Free | No | No | [Free plan details](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Hobby | Yes | No | [Current price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Startup | Yes | No | [Current price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Business | Yes | No | [Current price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Scaling | Yes | Yes | [Current price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Professional | Yes | Yes | [Current price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Advanced | No | Yes | [Current price](https://www.scraperapi.com/pricing?fp_ref=coupons) |
| Enterprise | No | Yes | [Current price](https://www.scraperapi.com/pricing?fp_ref=coupons) |

Two numbers are published outright: the free plan gives 1,000 API credits at a maximum of 5 concurrent connections, and the trial runs seven days with 5,000 credits. Both are enough to test one city page and see whether the response parses cleanly in your pipeline.

## What Redfin actually puts in your way

Redfin runs anti-scraping protection. Scrapfly's open-source Redfin example scraper makes this explicit: its base config enables the ASP bypass and sets the proxy country to `US` on every request, because the site will not serve content otherwise. Your geographic location matters as much as your user agent.

The list data comes from a JSON search API. Scrapfly's parser strips a `{}&&` prefix before decoding, then reads the homes array out of `payload.homes`:

python
json.loads(response.content.replace("{}&&", ""))["payload"]["homes"]


Skip that replace step and every parse throws. It is a three-line bug that costs people an afternoon.

Whether a plain request works at all depends on your exit IP. Unbrowse's Node scraper sends the same request from your machine with a normal browser user agent and reports it works without a key. Scrapfly's config says the opposite for generic crawlers. Both can be true: residential IPs and a clean request profile get further than data-center IPs, and the difference shows up as a 403 rather than a documented quota.

Detail pages are still HTML, and the anchors shift when Redfin redesigns. Scrapfly's example uses `//div[@data-rf-test-id='abp-price']/div` for price, `.est-monthly-payment` for the estimated payment, `div[contains(@class,'street-address')]` plus `div[contains(@class,'cityStateZip')]` for the address, `#marketing-remarks-scroll p span` for remarks, and `.amenity-group ul div.title` for grouped features. Rentals take a different path: pull `og:image` from the page, split the content on `rent/`, take the 36-character `rental_id`, then hit `https://www.redfin.com/stingray/api/v1/rentals/{rental_id}/floorPlans` for floor plan data.

## Four ways to get Redfin data, and where ScraperAPI fits

**Call Redfin's JSON directly.** Unbrowse's scraper requests the map-search API the site itself loads, parses locally, and caps at 350 homes per request with thousands per region across requests. It documents its own limits plainly: the public tool covers homes for sale, not rentals or sold listings.

**Bypass the protection.** The Scrapfly approach: a scraping API that toggles ASP and a US proxy, then either parse the JSON for list pages or parse HTML for detail pages. This is the pattern most production pipelines end up on.

**Hand a URL to a universal scraping API.** Oxylabs' Python tutorial posts `{"source": "universal", "url": "https://www.redfin.com/city/29470/IL/Chicago"}` and gets HTML back, then parses `div.bottomV2`, `span.homecardV2Price`, `span.collapsedAddress` and `div.stats` with BeautifulSoup before exporting to CSV. This is the closest analogue to ScraperAPI's general proxy/render mode, and it carries the same weakness: HTML parsing breaks when the page layout changes.

**Run a hosted actor.** The Apify automation-lab Redfin actor returns 20+ fields, defaults to `maxListings: 100`, paginates up to a 50-page safety limit, and exports JSON, CSV or Excel on a schedule. It claims no proxy is needed, since it uses Redfin's public GIS endpoints plus the Zippopotam.us geocoder for ZIP coordinates. For reference on the hosted-app end, ScrapeHero bills its Redfin app at 10 data credits per record.

Same idea on all four: Redfin's official Data Center API has a waitlist and targets enterprise users, so everyone else buys a third-party path.

### How ScraperAPI handles Redfin

Two modes, and they solve different problems.

**Proxy and render mode.** Send the Redfin URL with your `api_key` and a `country_code`, and get back the page. Point it at the search JSON endpoint and you get JSON; point it at a listing page and you get rendered HTML for the selectors above. This mode handles the ASP and IP problem so your code stays a plain GET.

**Structured data endpoints.** ScraperAPI publishes Redfin endpoints that return parsed JSON, so no BeautifulSoup and no selector maintenance. The listing search endpoint takes a Redfin search URL, filters included:


https://www.redfin.com/city/1826/MA/Boston/filter/min-beds=3,min-baths=2


and returns records keyed by `address`, `phone`, `number_beds`, `number_baths`, `sq_ft`, `thumbnail_img_url`, a `price` array, `badge`, `key_facts` and `badge_sold`. There is a separate Redfin rental endpoint that takes a listing URL and returns structured JSON. 👉 [Look at the Redfin templated endpoints](https://www.scraperapi.com/solutions/structured-data/redfin-rental-properties-scraper/?fp_ref=coupons)

Read the sample response before you trust the field fill rate. ScraperAPI's own documentation includes a record with `"address": ""` and an empty `price` array, so filter blanks before you run inventory counts.

## Six traps that break Redfin pipelines

**1. ZIP URLs do not carry a region id.** Redfin's `/zipcode/78704` style URLs leave out the identifier its API needs. Unbrowse's guidance is to pull the city or county and filter on `zip` afterward; the Apify actor notes ZIP searches are a filter over a wider market response, so they may not expose every listing. Pull city-level and filter locally.

**2. Sorting by newest fails at the source.** This is the one almost nobody documents. The Apify actor rejects `newest` and `days-on-market-asc` because Redfin's GIS endpoint rejects them with an explicit input error. Supported orders are recommended, price ascending and descending, square footage descending, and lot size descending. Pull with a supported sort, then order by listing date or days on market in your own database.

**3. Sold and pending coverage is not guaranteed.** Filters for `sold` and `pending` pass through to Redfin, but the endpoint may return no matching inventory even where the website shows older sales. If your analysis depends on comparables, budget for gaps.

**4. An empty result is not a crash.** The distinction matters for monitoring: no inventory in that area and filter is a legitimate empty result, while an invalid URL, failed ZIP geocoding, or malformed response is a failure. ScraperAPI returns a 429 when you exceed your plan's concurrency, with no extra charge and no queueing, which is a third state again: throttle, not failure. Build all three into your alerting.

**5. Some MLSs hide fields.** Days on market is hidden by certain MLSs, and listing agent and broker fields surface only when the MLS permits it. Do not build a model that assumes those columns are populated.

**6. A per-request cap is a tool limit, not a Redfin limit.** Unbrowse's 350 homes per request is that scraper's ceiling. Redfin will keep serving pages; you paginate.

## What the output should look like

Field names differ across tools; the underlying listing does not. A real record for 312 Horseback Holw, Austin, TX 78732 from Unbrowse, next to a ScrapeHero record for 214–216 Onondaga Ave, San Francisco, gives you the normalization checklist.

| Field | Example value | What to watch |
| --- | --- | --- |
| `propertyId` / `mlsId` | `31071505` / `2180998139890850489` | Redfin's id and the MLS number are different keys; never join on the wrong one |
| `status` | `Coming Soon` | Filter status before you count "active" inventory |
| `price` | `1050000` | ScrapeHero returns `"1699000"` as a string, ScraperAPI returns `price[0].cost` plus a currency field. Normalize to integer plus currency |
| `beds` / `baths` / `sqft` | `4` / `4` / `3619` | ScraperAPI's search endpoint returns display strings like `"3 beds"` and `"3,300 sq ft"`. Strip the units |
| `lotSize` | `10236` | Also appears as prose in `keyFacts` (`"0.23 acre lot"`). Pick one source |
| `yearBuilt` / `pricePerSqft` | `2008` / `290` | Price per square foot is often computed by the source, not Redfin |
| `daysOnMarket` | `1` | ScrapeHero shows `"4 hours"` for the same concept. Parse it yourself, and expect nulls where the MLS hides it |
| `latitude` / `longitude` | `30.3492835` / `-97.9038045` | Enough precision for map joins |
| `lastSoldDate` | `2022-03-22` | Not the same as `soldDate` on a pending sale |
| `photoCount` / `photo` | `79` | Photos live on Redfin's CDN; hotlinking is a separate question from scraping |
| `keyFacts` | `["0.23 acre lot", "Garage", "Pool"]` | Free-text highlights, not a controlled vocabulary |
| `listingAgent` / `listingBroker` | `Raquel Roque` / `Compass` | Present only when the MLS allows it |

## How credits are actually charged

A standard page request costs 1 credit. What you consume depends on the site and the parameters you send, and ScraperAPI prices the harder targets accordingly: Amazon at 5 credits, Google and Bing at 25, LinkedIn at 30. Sites behind Cloudflare, DataDome or PerimeterX add **10 credits per request** when ScraperAPI bypasses that protection.

Redfin is in that last category, which is the number that matters for budgeting. Until the Domain Cost Estimator in your dashboard gives you the exact figure for a specific Redfin URL, assume a request costs materially more than the base rate and size your plan on that assumption. `max_cost` lets you cap a single scrape so one expensive URL cannot burn through a month.

This is also where "cheap" gets deceptive across tools. ScrapeHero charges 10 data credits per record, Apify charges per actor event, Scrapfly per API call. A per-record price and a per-request price look comparable on a pricing page and diverge the moment pagination and retries enter the picture. 👉 [Compare the tiers that include pay-as-you-go](https://www.scraperapi.com/pricing?fp_ref=coupons)

## The legal part, briefly

Redfin listings come from public MLS feeds and display without a login, which is why third-party scrapers exist at all. In the US, hiQ Labs v. LinkedIn (9th Cir. 2022) addressed whether scraping public data violates the CFAA and came out favorable to scrapers; it says nothing about contracts, state law, or the GDPR. Redfin's terms of service govern your requests regardless of what a court says about the CFAA. If you are delivering listings to a client, settle the terms question before you build the pipeline. Third-party tools are not affiliated with or endorsed by Redfin.

## Which tier to buy

Run one city URL through the structured search endpoint on the free plan first (1,000 credits, 5 concurrent connections). If the fields you need come back populated and the sort order you want is supported, move to the trial for the volume test.

For recurring daily or weekly pulls across a few metros, pick a paid tier with auto-upgrade: Hobby, Startup, Business, Scaling or Professional. A month where a new metro pushes you past your allowance moves you up a tier instead of stopping your job mid-run. Annual billing suits workloads with a fixed budget, and annual plans can renew early for a fresh credit batch, which covers an unexpected spike without a plan change.

For seasonal or spiky volume, pay-as-you-go is the better fit: it is available on Scaling, Professional, Advanced and Enterprise at a fixed per-credit rate, capped at a monthly spend you set.

Three things to confirm before you enter a card number: run the Domain Cost Estimator against the exact Redfin URL pattern you plan to hit, check the fill rate on a 20-record sample rather than a single record, and verify that the sort parameter your pipeline depends on is one Redfin's GIS accepts. Get those three answers and the tier choice usually makes itself. 👉 [Start on the free plan and price a Redfin request](https://www.scraperapi.com/?fp_ref=coupons)
