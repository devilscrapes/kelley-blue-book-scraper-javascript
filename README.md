<p align="center"><a href="https://apify.com/DevilScrapes/kelley-blue-book-valuation-scraper"><img src="assets/banner.svg" alt="Kelley Blue Book Valuation Scraper — JavaScript example" width="100%"></a></p>

# Kelley Blue Book Scraper for JavaScript and Node.js: KBB valuations inside your app

<p align="center">
<a href="https://apify.com/DevilScrapes/kelley-blue-book-valuation-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/kelley-blue-book-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/kelley-blue-book-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

Use this repository to request Kelley Blue Book valuations from Node.js through the **[Kelley Blue Book Valuation Scraper](https://apify.com/DevilScrapes/kelley-blue-book-valuation-scraper)**. The whole client is a single ES module that leans on the official `apify-client`: send a list of vehicles, get back an array of objects, one per trim, with the Fair Purchase Price, the Fair Market Price low and high and the original MSRP as plain numbers. The kbb.com pages are fetched and read on Apify, so your backend, worker or serverless function never parses HTML. Run it from the command line to save `results.json` and `results.csv`, or import the call into the service that needs the prices.

> **Actor:** [Kelley Blue Book Valuation Scraper](https://apify.com/DevilScrapes/kelley-blue-book-valuation-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [kelley-blue-book-scraper-python](https://github.com/devilscrapes/kelley-blue-book-scraper-python)

## 🔥 What you get

- **Every trim gets its own row**: a single make, model and year usually covers several trims, and each one arrives with its own Fair Purchase Price, the low and high ends of the Fair Market Price range and the original sticker price.
- **Two ways to list vehicles**: structured `{make, model, year}` objects, KBB-style `make/model/year` strings, or both in one input. They are merged into a single list of at most 50 vehicles.
- **Numbers stay numbers**: prices are numeric fields in US dollars, with `currency` fixed to `USD`, so there is no string parsing between you and a chart.
- **The schema.org offer figures as well**: `offer_price` and `offer_max_price` carry the offer values the page publishes for each trim.
- **A missing vehicle does not fail the batch**: a make, model or year that KBB does not list, or a page that will not parse, is logged and skipped while the rest of the batch completes.
- **Temporary errors are retried**: rate-limit and server-error responses are retried with backoff, and plain Apify Proxy is enough for this site.

## 🎯 Use cases

### Show a KBB reference next to a listing

In a marketplace or dealer site, look up the trim being viewed in your cached valuations and render the Fair Market range beside the asking price. Refresh the cache with a scheduled run instead of fetching on every page view.

### Pull reference values inside a dealer tool

A sales tool can send the make, model and year a customer names to a small Express or Fastify route that starts a run and returns the trims. The salesperson picks the right trim and has the KBB figures without leaving the app.

### Feed valuations into an underwriting service

Lenders and insurers can run the vehicles in a batch of applications, store the rows keyed by year, make, model and trim, and let the scoring service read the reference value from their own database.

### Alert on valuation changes

Run the same vehicle list weekly from GitHub Actions or a cron container, diff each trim's `fair_purchase_price` against last week's and send a webhook to Slack when a model moves by more than a set amount.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/kelley-blue-book-scraper-javascript.git
cd kelley-blue-book-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Kelley Blue Book Valuation Scraper](https://apify.com/DevilScrapes/kelley-blue-book-valuation-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

The core of the client is short enough to paste into any service: one constructor, one `call()` that waits for the run, and one `listItems()` that returns the trim records.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/kelley-blue-book-valuation-scraper').call({"vehicles": [{"make": "toyota", "model": "camry", "year": 2021}], "vehicleSlugs": ["ford/f-150/2022"], "proxyConfiguration": {"useApifyProxy": true}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` reads `input.json` and forwards it untouched. One vehicle goes in as an object and one as a `make/model/year` string, so you can see both formats; the Actor merges them before fetching. The sample run in the Actor's write-up returned ten trim rows for these two, which keeps a first try close to the start fee.

```json
{
  "vehicles": [
    {
      "make": "toyota",
      "model": "camry",
      "year": 2021
    }
  ],
  "vehicleSlugs": [
    "ford/f-150/2022"
  ],
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `vehicles` | array | yes | `[{"make": "honda", "model": "civic", "year": 2020}, {"make": "ford", "model": "f-150", "year": 2022}]` | — |
| `vehicleSlugs` | array | no | `["toyota/camry/2021"]` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Each record describes one trim. Prices are numbers or null, `year` is an integer, `currency` is always `USD`, and make and model come back lowercased and hyphenated, so normalize your own keys the same way before joining. `source_url` gives you a link back to the KBB page for display or audit.

| Field | Type | Always present |
|---|---|---|
| `make` | string | yes |
| `model` | string | yes |
| `year` | integer | yes |
| `trim` | string or null | no |
| `fair_purchase_price` | number or null | no |
| `fair_market_price_low` | number or null | no |
| `fair_market_price_high` | number or null | no |
| `original_msrp` | number or null | no |
| `offer_price` | number or null | no |
| `offer_max_price` | number or null | no |
| `currency` | string | yes |
| `source_url` | string | yes |
| `scraped_at` | string | yes |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result row emitted | $0.003 |

Prices as configured on 2026-09-15. The [Actor page](https://apify.com/DevilScrapes/kelley-blue-book-valuation-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

This script checks an asking price against the saved trims, the way a listing page or a chat bot might. Call it as `node check-price.js 2021 toyota camry 23500`; it prints each trim's range and says whether the price falls under, inside or over it. Only Node.js built-ins are used.

```javascript
import { readFile } from 'node:fs/promises';

const [year, make, model, asking] = process.argv.slice(2);
if (!asking) {
    console.error('Usage: node check-price.js <year> <make> <model> <asking price in USD>');
    process.exit(1);
}

const rows = JSON.parse(await readFile('results.json', 'utf8'));
const price = Number(asking);
const usd = (n) => `$${Math.round(n).toLocaleString('en-US')}`;
const trims = rows.filter(
    (r) => r.year === Number(year) && r.make === make.toLowerCase() && r.model === model.toLowerCase(),
);

const verdict = ({ fair_market_price_low: low, fair_market_price_high: high }) => {
    if (low == null || high == null) return 'no range on the page';
    if (price < low) return `${usd(low - price)} under the range`;
    if (price > high) return `${usd(price - high)} over the range`;
    return 'inside the fair market range';
};

if (trims.length === 0) {
    console.log(`No rows for ${year} ${make} ${model}; add it to input.json and run again.`);
} else {
    console.table(trims.map((r) => ({
        trim: r.trim ?? '(single valuation)',
        low: r.fair_market_price_low,
        high: r.fair_market_price_high,
        fairPurchase: r.fair_purchase_price,
        verdict: verdict(r),
    })));
}
```

## ❓ FAQ

### Do I need anything from Kelley Blue Book to use this?

No account, key or login. The Actor reads the public valuation pages, so an Apify token is the only secret your service has to store, which keeps prototypes and internal tools simple to deploy.

### Should an HTTP request wait for the run?

For a handful of vehicles a request can usually wait, but for a full batch it is better to start the run with `start()`, return quickly and pick up the rows from an Apify webhook or a background job. Caching rows by year, make, model and trim avoids repeat runs for popular cars.

### Does it cover other countries?

No. It reads the US site, www.kbb.com, and every price is in US dollars. Regional KBB sites are not covered.

### How do I cap what a test costs?

Keep the vehicle list short and give the CLI a dollar ceiling with `--max-charge`. The limit travels with the run to Apify, so a test job in CI cannot spend more than you allowed it.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/kelley-blue-book-valuation-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Kelley Blue Book Valuation Scraper on Apify Store](https://apify.com/DevilScrapes/kelley-blue-book-valuation-scraper)
- [Python version of this example](https://github.com/devilscrapes/kelley-blue-book-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [AutoTrader.ca Listings Scraper](https://apify.com/DevilScrapes/autotrader-ca-listings-scraper) — live Canadian dealer and private listings, to compare asking prices with valuations.
- [Bring a Trailer Auctions Scraper](https://apify.com/DevilScrapes/bring-a-trailer-auctions-scraper) — collector-car auction results with final bids, a second opinion on older and enthusiast cars.
- [AutoScout24 Germany Car Scraper](https://apify.com/DevilScrapes/autoscout24-germany-cars) — used-car listings from the German market for cross-border price checks.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/kelley-blue-book-valuation-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Kelley Blue Book or Cox Automotive.
