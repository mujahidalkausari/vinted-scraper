# Vinted Products Crawler — scrape Vinted listings through the Apify API

Get Vinted listings into your own code as structured JSON: title, price, brand, size,
condition, photos and seller, from any of Vinted's 27 country sites. The scraper runs on
[Apify](https://apify.com), so there are no proxies, browsers or anti-bot measures for you
to manage. You send a search URL and get rows back.

**▶ Run it on Apify: [Vinted Products Crawler](https://apify.com/alkausari_mujahid/vinted-products-crawler)**

This repository holds documentation and examples for developers. The scraper itself runs
on the Apify platform.

---

## Scrape Vinted with one HTTP request

Paste any Vinted search or catalog URL from your browser. This call runs the scraper and
returns the listings in the same response:

```bash
curl -X POST \
  "https://api.apify.com/v2/acts/alkausari_mujahid~vinted-products-crawler/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "startUrls": [{ "url": "https://www.vinted.fr/catalog?search_text=levis%20501&order=newest_first" }],
    "maxItems": 50
  }'
```

Your API token is under **Settings → API & Integrations** in [Apify Console](https://console.apify.com).
The synchronous endpoint waits up to 300 seconds, which suits small runs. For larger ones,
start the run and fetch the dataset when it finishes, as in the examples below.

## Scrape Vinted from JavaScript

```bash
npm install apify-client
```

```js
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: process.env.APIFY_TOKEN });

const run = await client.actor('alkausari_mujahid/vinted-products-crawler').call({
    startUrls: [{ url: 'https://www.vinted.de/catalog?search_text=nike%20air%20max&order=newest_first' }],
    maxItems: 200,
});

const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(`${items.length} listings`, items[0]);
```

## Scrape Vinted from Python

```bash
pip install apify-client
```

```python
import os
from apify_client import ApifyClient

client = ApifyClient(os.environ["APIFY_TOKEN"])

run = client.actor("alkausari_mujahid/vinted-products-crawler").call(run_input={
    "startUrls": [{"url": "https://www.vinted.co.uk/catalog?search_text=barbour%20jacket"}],
    "maxItems": 200,
    "fetchItemDetails": True,
})

items = client.dataset(run["defaultDatasetId"]).list_items().items
print(len(items), "listings")
```

---

## Input

| Field | Default | What it does |
|---|---|---|
| `startUrls` | — | Vinted catalog or search URLs, direct item URLs (`/items/…`), or a mix. Any country domain works, and domains can be mixed in one run. |
| `maxItems` | `0` | Exact number of listings to return. `0` means no limit. |
| `fetchItemDetails` | `false` | Also open each listing's own page for the fields in [Fast vs full details](#fast-vs-full-details). |
| `startPageNum` / `endPageNum` | `1` / `1` | Catalog pages to read, up to Vinted's limit of 10. A page holds about 96 listings. |
| `maxConcurrency` | `10` | Pages fetched in parallel. |
| `proxyConfiguration` | Apify residential | Leave as is unless you have a reason not to. |

## Output

One dataset row per listing, with the same keys on every row of a run. This record was
scraped from `vinted.fr` with full details on:

```json
{
    "itemId": 9627980284,
    "url": "https://www.vinted.fr/items/9627980284-pantalon-hugo-boss-beige-clair",
    "domain": "vinted.fr",
    "title": "pantalon Hugo Boss beige clair taille 50 coupe habillée",
    "price": "€10.00",
    "priceAmount": 10,
    "currency": "EUR",
    "totalPrice": "€11.20",
    "size": "W50 | FR 60",
    "condition": "Très bon état",
    "brand": "Hugo Boss",
    "brandUrl": "https://www.vinted.fr/catalog/261-tailored-trousers/brand/120-hugo-boss",
    "color": "Beige",
    "description": "Pantalon homme Hugo Boss, coloris beige clair / écru…",
    "uploaded": "Il y a 14 minutes",
    "seller": "lysker",
    "sellerProfileUrl": "https://www.vinted.fr/member/14081384",
    "sellerRating": 5,
    "sellerRatingCount": 253,
    "sellerLastSeen": "Vu la dernière fois : il y a 41 minutes",
    "favorites": 0,
    "images": ["https://images1.vinted.net/tc/05_02362_GTAwfUjrJ4XjL4aoiRA6v68Q/1786381468.jpeg"],
    "shippingPrice": "à partir de 2,83 €",
    "freeShipping": false,
    "material": null,
    "measurements": null,
    "scrapedAt": "2026-08-10T17:19:09.234Z"
}
```

Results can be downloaded as JSON, CSV, Excel or XML, or read straight from the API.

## Fast vs full details

| | Fast (default) | Full details |
|---|---|---|
| Requests | one per ~96 listings | one per listing |
| Fields | 16 | 29 |
| Photos | main photo | up to 9 |
| Seller | profile link only (`seller` is `null`) | username, rating, review count, last seen |
| Adds | — | description, colour, material, measurements, upload date, shipping price |

Fast mode suits price tracking and bulk monitoring. Full details is slower, because it
opens every listing, and is worth it when you need descriptions or seller data.

---

## Things to design around

**Each search URL tops out at about 960 listings.** Vinted serves 10 pages per search, on
the website too. To go further, split one search into narrower ones, for example by price
band, and pass them all in `startUrls`:

```json
{
    "startUrls": [
        { "url": "https://www.vinted.com/catalog?search_text=jordan%201&price_from=0&price_to=49.99" },
        { "url": "https://www.vinted.com/catalog?search_text=jordan%201&price_from=50&price_to=99.99" },
        { "url": "https://www.vinted.com/catalog?search_text=jordan%201&price_from=100" }
    ],
    "endPageNum": 10
}
```

**`maxItems` is shared across all URLs in a run**, filled first come, first served. For even
coverage across several searches or countries, leave it at `0` and size each search with
`endPageNum` instead.

**Text comes in the site's own language.** `condition`, `size`, `uploaded` and
`sellerLastSeen` read "Très bon état" on vinted.fr and "Sehr gut" on vinted.de. To compare
across countries, use `priceAmount`, `currency`, `sellerRating` and `domain`, which are
language-neutral.

**Some keys exist but are always empty.** `sellerLocation` and `sellerBadges` are part of
the full-details schema and currently come back `null`, so don't build on them.

**Listings sell mid-run.** In full-details mode, a listing that sells between being found
and being opened is still returned, with the data from its catalog page. A direct item URL
that no longer exists is skipped and noted in the run log.

---

## Pricing

Pay per result: **$1.50 per 1,000 listings** in fast mode. Full details adds a per-listing
charge for opening each listing's own page; the current rate is on the Actor's
[Pricing tab](https://apify.com/alkausari_mujahid/vinted-products-crawler/pricing). Listings
whose page can't be read aren't charged the full-details rate.

Set **Maximum cost per run** in the run options and the scraper stops cleanly once it's
reached, keeping everything collected so far.

## Use it from an AI assistant

Apify's MCP server lets Claude, Cursor and other MCP clients run this scraper for you. Add
this server URL to your client:

```text
https://mcp.apify.com?tools=alkausari_mujahid/vinted-products-crawler
```

Then ask in plain words, for example "get the 100 newest Levi's 501 listings on vinted.fr
under €30".

## Schedules and integrations

On Apify you can run a search on a schedule, get a webhook when a run finishes, and send
results to Google Sheets, Zapier, Make, Slack or your own endpoint. All of it is set up from
the [Actor page](https://apify.com/alkausari_mujahid/vinted-products-crawler).

---

## Support

Questions, bugs or feature requests: open an issue in the
[Actor's Issues tab](https://apify.com/alkausari_mujahid/vinted-products-crawler/issues), or
email [alkausarimujahid@gmail.com](mailto:alkausarimujahid@gmail.com).

*This is an independent tool. It is not affiliated with, endorsed by or sponsored by Vinted
Limited. "Vinted" is a trademark of its owner and is used here only to describe the data
source.*
