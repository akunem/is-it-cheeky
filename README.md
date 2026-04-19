# Is It Cheeky?

A small static web app for England and Wales that helps first-time buyers sanity-check an asking price against HM Land Registry data: the property’s own sold history, comparable sales on the same street (last three years, ideally matching property type), and an HPI-adjusted “expected price today” derived from the UK House Price Index.

It is guidance only, not valuation, financial, or legal advice.

## Tech stack

| Layer | Choice |
|--------|--------|
| UI | Single-page HTML, CSS, and vanilla JavaScript (no build step or framework) |
| Typography | Google Fonts (DM Sans, DM Serif Display) |
| Sold prices & HPI | HM Land Registry [linked data](https://landregistry.data.gov.uk/) via SPARQL over `fetch` |
| Address search | Ideal Postcodes HTTP API (autocomplete + UDPRN detail) |
| Contact form | `POST` to a Zapier catch hook (`contact.html`) |
| Optional local server | [`serve`](https://github.com/vercel/serve) with [`serve.json`](serve.json) (rewrite rules) |

There is no `package.json` in this repo; dependencies are loaded from CDNs or public APIs in the browser.

## How Land Registry SPARQL queries work

All Land Registry requests go to the public SPARQL endpoint:

`https://landregistry.data.gov.uk/landregistry/query`

### Price Paid (address history)

The app builds a query that matches one address using the Land Registry common vocabulary: postcode, **PAON** (primary addressable object name—house number or name), and street, bound to transactions in the price-paid graph (`lrppi:` / `lrcommon:` prefixes). It returns each transaction’s date and price, ordered by date descending so the most recent sale is first.

Implementation: `buildAddressQuery` and `fetchSPARQL` in [`index.html`](index.html).

### Price Paid (street comparables)

A second query matches the same street and postcode, optional PAON, SAON, and property type on the transaction, and filters transaction date to the last three years (`FILTER` with `xsd:date`). Results are capped (`LIMIT 100`) and ordered by date.

Property type filtering is applied in JavaScript after the response: the query may return a mix of types; the code keeps only rows whose `lrppi:propertyType` URI matches the user’s selection. If that yields no rows, it falls back to all types on that street and shows a notice.

Price-paid SPARQL uses `POST` with `Content-Type: application/x-www-form-urlencoded`, body `query=...`, and `Accept: application/sparql-results+json`. The address query uses a 5s timeout; the street query uses 20s because it can be heavier.

### UK HPI via SPARQL

UK HPI monthly resources are queried by constructing the canonical resource URI:

`http://landregistry.data.gov.uk/data/ukhpi/region/{region}/month/{YYYY-MM}`

…then issuing a small SPARQL `SELECT` with `OPTIONAL` blocks for the type-specific index (e.g. detached) and the general `ukhpi:housePriceIndex`, preferring the typed value when present. Those requests use `GET` with the query in the URL and `_format=json` (see `fetchUKHPI` in [`index.html`](index.html)).

## How the HPI calculation works

1. Region is inferred from the property postcode (see below).
2. Historic index: for the date of the latest recorded sale at that address, the app fetches HPI for that sale’s calendar month and the previous and next month in parallel, then uses the first month that returns data (handles edge cases or missing months).
3. Current index: because published HPI lags by a few months, the app requests the index for 2, 3, and 4 months before “now” in parallel and takes the first non-null value (most recent available).
4. Property type: the form maps to UK HPI predicates—detached, semi-detached, terraced, flat—falling back to the all-dwellings index when needed.
5. Expected price today:

   `expectedPrice = lastSoldPrice × (currentHPI ÷ historicHPI)`

   So the last sale is scaled by regional index movement, not by street-level repeat sales.

6. Verdict (vs that expected price): Below Market if more than 5% below; Looks Fair within ±5%; A Bit Cheeky between 5% and 15% above; Very Cheeky if more than 15% above. (See [`caveats.html`](caveats.html) for the same definitions in user-facing copy.)

If there is no sale history for the exact address, there is no HPI-based verdict (but street comparables may still show). If HPI calls fail, the UI explains partial data.

## How postcode-to-region mapping works

UK HPI regional series use slug names such as `london`, `south-east`, `wales`. The app does not call an external geocoder for region; it uses a static map `POSTCODE_REGION` from outward postcode letters to that slug.

Algorithm (`getRegion` in [`index.html`](index.html)):

1. Normalise postcode, take the leading run of letters (outward code prefix).
2. Look up two-character prefix first (e.g. `SW`, `EC`), then one-character (e.g. `B`, `M`).
3. If nothing matches, default to `england` (national series).

This is a deliberate trade-off: simple and fast, but not identical to every official boundary (e.g. cross-border quirks). The HPI adjustment is always regional, not postcode-level.

## How to run locally

Opening `index.html` directly as a `file://` URL may cause `fetch` to fail against Land Registry due to CORS or browser security. Serve the folder over HTTP instead, for example:

```bash
npx serve .
```

Then open the URL the CLI prints (often `http://localhost:3000`). The app itself suggests this pattern when it detects a likely network/CORS failure.

[`serve.json`](serve.json) configures rewrites for `serve` (e.g. mapping `/index.html`); adjust if your host uses different routing.

## How to deploy

This is a static site: deploy the repository root (or the files `index.html`, `caveats.html`, `contact.html`, and `serve.json` if your host reads it) to any static host, for example:

- GitHub Pages, Cloudflare Pages, Netlify, Vercel (static), or S3 + CloudFront

Set the site entry to `index.html`. No server-side runtime is required for the property checker; all Land Registry calls run in the user’s browser.

#### Secrets and third-party services

- Ideal Postcodes: the HTML embeds an API key for address autocomplete. For a fork or production hardening, replace it with your own key (or proxy requests through a small backend if you want to hide the key).
- Contact form: [`contact.html`](contact.html) posts to a Zapier webhook URL; change that URL if you fork the project.

## Data sources

- [HM Land Registry linked open data](https://landregistry.data.gov.uk/) — Price Paid and UK House Price Index  
- Coverage: England and Wales only  

## Licence

Add a licence file if you intend to open-source this repository; none is present in the tree at the time of writing.

## Project status

This was built as a personal project during a week off and is not actively being developed with new features. Bugs will be fixed when time allows. Feedback is welcome via the contact form on the site at [www.isitcheeky.com](https://www.isitcheeky.com).
