<p align="center"><a href="https://apify.com/DevilScrapes/suumo-japan-real-estate"><img src="assets/banner.svg" alt="Suumo.jp Japan Real Estate Scraper — Python example" width="100%"></a></p>

# Suumo Scraper in Python: Japanese rental and condo listings to pandas and CSV

<p align="center">
<a href="https://apify.com/DevilScrapes/suumo-japan-real-estate"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="Python" src="https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/suumo-scraper-python/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/suumo-scraper-python/actions/workflows/ci.yml/badge.svg"></a>
</p>

Here is a short Python client for the **[Suumo.jp Japan Real Estate Scraper](https://apify.com/DevilScrapes/suumo-japan-real-estate)** on Apify. You open suumo.jp, narrow a search to the area and budget you care about, copy the address bar, and this script returns that result set as rows: listing ID, building name, monthly rent or asking price, kanrihi, shikikin, reikin, madori, square metres, floor, building age, address and every nearby station with walking minutes. Money is already integer yen, so the numbers go straight into pandas arithmetic. The crawling runs on Apify; locally you get `results.json` and `results.csv`.

> **Actor:** [Suumo.jp Japan Real Estate Scraper](https://apify.com/DevilScrapes/suumo-japan-real-estate) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in JavaScript:** [suumo-scraper-javascript](https://github.com/devilscrapes/suumo-scraper-javascript)

## 🔥 What you get

- **Your search, not ours**: the input is a results URL copied from suumo.jp, so every filter Suumo offers (ward, line, rent band, layout, age) is already applied before the Actor reads a page.
- **One row per room, not per building**: Suumo's rental cards group several vacant units under one building; each unit becomes its own row and inherits the building's name, address, type, age and stations.
- **Yen as integers**: display strings such as `6.5万円` arrive as `65000`. A deposit or key money shown as `なし` becomes `0`, while a dash stays `null`, so "none charged" and "not stated" remain different things.
- **Every station on the card**: `stations` is a list of objects with `line`, `station` and `walk_minutes`. Rental cards list up to three, used-condo cards list one.
- **Rentals and used condos with one schema**: a `/jj/chintai/ichiran/` URL is read as rent mode, a `/jj/bukken/ichiran/` URL as buy mode, and `listing_mode` on each row says which.
- **Agency details when you ask for them**: with `fetchListingDetails` on, the Actor opens each listing page for the handling agency's name (plus phone on rentals, and management fee and floor on condos), billed as a separate event only when a name is found.
- **Caps that hold across URLs**: `maxItems` counts rows over all your search URLs together, and `maxPages` limits how far each one paginates. Pages that answer with temporary errors are retried with growing pauses.

## 🎯 Use cases

### Price the real cost of moving into a Tokyo flat

Rent alone hides the upfront bill. Add `deposit_yen`, `key_money_yen`, the first month's `price_yen` and `management_fee_yen` per row and you get a move-in figure you can sort on. The recipe below does this and ranks stations by median rent per square metre.

### Build a ward-level rent dataset for research

Save one search URL per ward of interest, run them weekly from cron and append each CSV to a Parquet file. Because rows carry `area_sqm` and `building_age_years`, a regression of rent on size, age and walk time is a notebook away.

### Screen used condos for an investment model

Paste a `/jj/bukken/ichiran/` search and every row comes back in buy mode with the sale price in yen, layout, floor area and construction-derived age. Divide price by area for a yen-per-m² column and compare districts before anyone opens a listing page.

### Give a relocation client a shortlist in English

Names and addresses stay in Japanese, which is what you want for matching against maps or translation APIs. Filter in pandas by walk time and total upfront cost, translate only the shortlist, and send it as a spreadsheet.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/suumo-scraper-python.git
cd suumo-scraper-python
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export APIFY_TOKEN=your_token_here
python main.py                      # uses input.json
python main.py my-input.json --out data --max-charge 2
```

The script starts the [Suumo.jp Japan Real Estate Scraper](https://apify.com/DevilScrapes/suumo-japan-real-estate), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

If the wrapper script is more than you need, these five lines are the whole job: hand the Actor a Suumo URL, let `call()` wait for the run, then loop over the dataset.

```python
from apify_client import ApifyClient

client = ApifyClient("YOUR_APIFY_TOKEN")
run = client.actor("DevilScrapes/suumo-japan-real-estate").call(run_input={'searchUrls': ['https://suumo.jp/jj/chintai/ichiran/FR301FC001/?ar=030&bs=040&ta=13&sc=13104&page=1'], 'maxItems': 10, 'maxPages': 1, 'proxyConfiguration': {'useApifyProxy': True}})
for item in client.dataset(run.default_dataset_id).iterate_items():
    print(item)
```

## 📥 Input

`main.py` forwards `input.json` unchanged. The shipped file is a rental search in Shinjuku ward, Tokyo, held to one results page and ten rows, which is enough to inspect the shape of the data for a fraction of a cent. Swap in your own Suumo URL, raise `maxItems` and set `maxPages` to `0` when you want the whole result set.

```json
{
  "searchUrls": [
    "https://suumo.jp/jj/chintai/ichiran/FR301FC001/?ar=030&bs=040&ta=13&sc=13104&page=1"
  ],
  "maxItems": 10,
  "maxPages": 1,
  "proxyConfiguration": {
    "useApifyProxy": true
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `searchUrls` | array | yes | `["https://suumo.jp/jj/chintai/ichiran/FR301FC001/?ar=030&bs=040&ta=13&sc=13104&page=1"]` | — |
| `maxItems` | integer | no | `100` | — |
| `maxPages` | integer | no | `0` | — |
| `fetchListingDetails` | boolean | no | `false` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": true}` | — |

## 📤 Output

Each row is one rentable unit or one condo for sale. The money fields are `price_yen` (monthly rent in rent mode, total asking price in buy mode), `management_fee_yen`, `deposit_yen` and `key_money_yen`; the last two are always `null` for condos. Size and position come as `layout` (for example `1LDK`), `area_sqm`, `floor` and `building_age_years`, where a brand-new building counts as `0`. Location is `address` plus a `stations` list of `{line, station, walk_minutes}` objects. `listing_id`, `url`, `name`, `property_type`, `listing_mode`, `agency_name`, `agency_phone` and `scraped_at` complete the row. In the CSV export, Apify flattens `stations` into numbered columns, so prefer `results.json` for analysis.

| Field | Type | Always present |
|---|---|---|
| `listing_id` | string | no |
| `name` | string or null | no |
| `url` | string | no |
| `property_type` | string or null | no |
| `price_yen` | integer or null | no |
| `management_fee_yen` | integer or null | no |
| `deposit_yen` | integer or null | no |
| `key_money_yen` | integer or null | no |
| `address` | string or null | no |
| `stations` | array | no |
| `listing_mode` | string | no |
| `agency_name` | string or null | no |
| `agency_phone` | string or null | no |
| `layout` | string or null | no |
| `area_sqm` | number or null | no |
| `floor` | string or null | no |
| `building_age_years` | integer or null | no |
| `scraped_at` | string or null | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.00001 |
| Result emitted | $0.002 |
| Detail page enriched | $0.005 |

Prices as configured on 2026-08-12. The [Actor page](https://apify.com/DevilScrapes/suumo-japan-real-estate) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in Python

This pandas script (`pip install pandas`) reads `results.json`, picks the closest station for each unit, computes rent per m² and the upfront cost of signing, then prints the stations with the lowest median rent per m².

```python
import json

import pandas as pd

units = pd.DataFrame(json.load(open("results.json", encoding="utf-8")))
units = units[units["listing_mode"] == "rent"].copy()


def closest(stations):
    timed = [s for s in stations or [] if s.get("walk_minutes") is not None]
    return min(timed, key=lambda s: s["walk_minutes"]) if timed else {}


near = units["stations"].map(closest)
units["station"] = near.map(lambda s: s.get("station"))
units["walk_min"] = near.map(lambda s: s.get("walk_minutes"))
units["yen_per_sqm"] = (units["price_yen"] / units["area_sqm"]).round()
fees = ["price_yen", "management_fee_yen", "deposit_yen", "key_money_yen"]
units["move_in_yen"] = units[fees].fillna(0).sum(axis=1).astype(int)

by_station = (
    units.dropna(subset=["station", "yen_per_sqm"])
    .groupby("station")
    .agg(units=("listing_id", "count"), median_yen_per_sqm=("yen_per_sqm", "median"),
         median_move_in=("move_in_yen", "median"), median_walk=("walk_min", "median"))
    .sort_values("median_yen_per_sqm")
)
units.to_csv("units_with_costs.csv", index=False)
print(by_station.to_string())
```

## ❓ FAQ

### Does it handle sale listings as well as rentals?

It handles rentals and used condominiums. A search URL containing `/jj/bukken/ichiran/` is parsed in buy mode, where `property_type` is always `中古マンション`. Other Suumo sections, such as new-build projects or detached houses for sale, are not parsed as their own mode; anything that is not a bukken URL is treated as a rental search.

### Why is `agency_name` empty?

The search pages do not show it. Set `fetchListingDetails` to `true` and the Actor visits each listing's own page to read the agency. That roughly doubles the requests and adds a per-row event, so turn it on only for the listings you will act on.

### Is `building_age_years` exact?

It is in whole years. For rentals it comes from the age Suumo prints on the card, with new construction as `0`. For condos it is the current year minus the construction year, so it can be one year off depending on the month.

### My run finished with zero rows. What should I check?

Open the URL in a browser first: if Suumo shows no results there, the Actor has nothing to read. If it does, look at the run log for HTTP errors and try a Japanese exit by adding `"apifyProxyCountry": "JP"` to `proxyConfiguration`. This Actor reports an empty result in its status message but still finishes as succeeded, so check the row count in scheduled jobs.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/suumo-japan-real-estate) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Suumo.jp Japan Real Estate Scraper on Apify Store](https://apify.com/DevilScrapes/suumo-japan-real-estate)
- [JavaScript version of this example](https://github.com/devilscrapes/suumo-scraper-javascript)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [Rightmove Scraper — UK Property Listings](https://apify.com/DevilScrapes/rightmove-uk-property) — UK sale and rental listings from Rightmove.
- [Immowelt Germany Real Estate Scraper](https://apify.com/DevilScrapes/immowelt-germany-real-estate-scraper) — German apartments and houses listed on Immowelt.
- [Otodom Poland Real Estate Scraper](https://apify.com/DevilScrapes/otodom-poland-real-estate-scraper) — Polish property offers from Otodom.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/suumo-japan-real-estate) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with SUUMO, Suumo.jp or Recruit Co., Ltd..
