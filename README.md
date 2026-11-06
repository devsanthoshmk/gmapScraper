# mapScraper

A Python Google Maps business data scraper and lead generation pipeline.

## Purpose

mapScraper is a Python Google Maps extraction tool and business data scraper for local lead generation. It extracts business details, contact information, review metrics, and geographic coordinates from Google Maps without paid API keys. The built-in pipeline enriches raw search records with website signals, domain validation, and lead scores from 0 to 100.

Key capabilities:
- Collect business name, place ID, category, address, and Google Maps URL.
- Extract local and international phone numbers.
- Retrieve website URLs and domain names.
- Record GPS coordinates, star ratings, and review counts.
- Inspect business websites for contact pages, services, and modern JavaScript frameworks.
- Calculate transparent lead scores and assign business size segments.
- Run concurrent searches across multiple queries from text files.

## Architecture

The project separates raw data extraction from lead qualification.

```
gmapScraper/
├── main.py                 Pipeline CLI for scraping, enrichment, and batch processing
├── mapScraperX.py          Standalone search crawler CLI
├── requirements.txt        Python dependencies
├── pipeline/
│   └── orchestrator.py     Sequential workflow coordinator
├── mapScraper/
│   ├── placesCrawlerV2.py  Internal Maps SPA parser and JSON extractor
│   └── places_crawler.py   HTML places crawler with async phone fallback
├── enrichment/
│   ├── features.py         Feature extraction and ratio calculations
│   ├── web_scraper.py      Async website crawler for contact and framework signals
│   └── scoring.py          Lead scoring and segmentation logic
└── data/
    └── output.csv          Default output directory and sample CSV
```

Component responsibilities:
- `mapScraper/placesCrawlerV2.py` sends requests to Google Maps search endpoints, extracts the canonical `pb=` search token, and parses structured JSON records from the `tbm=map` response.
- `mapScraper/places_crawler.py` parses Google Places HTML and uses async calls to Google endpoints to retrieve missing phone numbers and addresses.
- `enrichment/features.py` parses phone numbers, checks domain formats, calculates review density, and weights ratings by review counts.
- `enrichment/web_scraper.py` runs asynchronous HTTP requests with `aiohttp` against business websites to detect contact pages, services pages, and frontend frameworks such as React, Vue, or Next.js.
- `enrichment/scoring.py` evaluates review volume, star ratings, website signals, and phone availability to assign a score from 0 to 100.
- `pipeline/orchestrator.py` runs search tasks and enrichment sequentially within a single process. Scraped data writes to CSV first, so enrichment runs on saved files without repeating search requests.

## Setup

### Prerequisites

- Python 3.10 or higher
- pip package manager

### Installation

Clone the repository and install dependencies:

```bash
git clone https://github.com/devsanthoshmk/gmapScraper.git
cd gmapScraper
pip install -r requirements.txt
```

Core libraries:
- `aiohttp`: asynchronous HTTP requests
- `beautifulsoup4` and `lxml`: HTML parsing
- `pandas`: tabular data processing and CSV storage
- `phonenumbers`: phone number parsing and formatting
- `tqdm`: progress bar tracking

## Usage

The repository provides two command line interfaces: `main.py` for pipeline workflows and `mapScraperX.py` for direct place searches.

### Pipeline CLI (main.py)

`main.py` supports three modes:
- `scrape`: queries Google Maps and outputs a raw CSV.
- `enrich`: reads an existing CSV, extracts website signals, and outputs an enriched CSV.
- `full`: runs the scraper and then enriches the results.

Examples:

```bash
# Scrape places for a single query
python main.py --mode scrape "marketing agencies in New York" --limit 50

# Scrape places using a query file
python main.py --mode scrape --queries-file query_example.txt --limit 25

# Enrich an existing CSV file
python main.py --mode enrich --input data/output.csv

# Enrich an existing CSV without fetching websites
python main.py --mode enrich --input data/output.csv --no-web-scraping

# Run the complete scrape and enrichment pipeline
python main.py --mode full --queries-file query_example.txt --output-file data/leads.csv

# Enable detailed debug logging
python main.py --mode enrich --input data/output.csv --log-level DEBUG
```

### Scraper CLI (mapScraperX.py)

`mapScraperX.py` runs searches with configurable fallback behavior.

Examples:

```bash
# Single search with result limit
python mapScraperX.py "restaurants in Miami" --limit 50

# Multiple searches from a text file
python mapScraperX.py --queries-file query_example.txt

# Run complete mode with phone and address fallback
python mapScraperX.py "plumbers in Chicago" --mode complete --concurrent 5
```

## Configuration

### Pipeline options (main.py)

| Option | Default | Description |
|---|---|---|
| `--mode` | `scrape` | Pipeline mode: `scrape`, `enrich`, or `full`. |
| `query` | None | Single search query text. |
| `--queries-file` | None | Path to a text file with one query per line. |
| `--lang` | `en` | Google Maps language code. |
| `--country` | `us` | Google Maps country code. |
| `--limit` | None | Maximum results per query. |
| `--output-file` | `data/output.csv` | Output file path for raw scrape results. |
| `--concurrent` | `3` | Maximum concurrent search tasks. |
| `--input` | None | Input CSV path for enrich mode. |
| `--no-web-scraping` | False | Skip website requests during enrichment. |
| `--web-concurrent` | `10` | Maximum concurrent website requests. |
| `--web-batch-size` | `100` | Number of URLs processed per batch. |
| `--web-timeout` | `10` | Per-request timeout in seconds for website fetches. |
| `--log-level` | `INFO` | Logging level: `DEBUG`, `INFO`, `WARNING`, `ERROR`. |

### Scraper options (mapScraperX.py)

| Option | Default | Description |
|---|---|---|
| `query` | None | Single search query text. |
| `--queries-file` | None | Path to a text file with one query per line. |
| `--mode` | `standard` | Scraping mode: `fast` (no fallback), `standard` (fallback if phone is missing), `complete` (always fallback). |
| `--lang` | `en` | Language code. |
| `--country` | `us` | Country code. |
| `--limit` | `0` | Maximum results to retrieve. 0 means no limit. |
| `--output-file` | `data/output.csv` | Output CSV file path. |
| `--concurrent` | `3` | Maximum concurrent search tasks. |

### Lead scoring model

The enrichment step scores each lead out of 100 points across four categories:

1. Review volume (maximum 30 points)
- 500 or more reviews: 30 points
- 200 to 499 reviews: 24 points
- 100 to 199 reviews: 18 points
- 50 to 99 reviews: 12 points
- 10 to 49 reviews: 7 points
- 1 to 9 reviews: 3 points
- 0 reviews: 0 points

2. Star rating (maximum 25 points)
- 4.5 stars or higher: 25 points
- 4.0 to 4.4 stars: 20 points
- 3.5 to 3.9 stars: 15 points
- 3.0 to 3.4 stars: 10 points
- Greater than 0 stars: 5 points

3. Website signals (maximum 30 points)
- Website URL present: 10 points
- Valid domain format: 5 points
- Contact page detected: 5 points
- Services page detected: 5 points
- Modern frontend framework detected: 5 points

4. Phone reachability (maximum 15 points)
- Phone number present: 15 points

Segment thresholds:

| Segment | Score range |
|---|---|
| micro | 0 to 24 |
| small | 25 to 49 |
| medium | 50 to 74 |
| large | 75 to 100 |

## Output format

### Raw scrape CSV

The raw scraper generates 13 columns:

| Column | Description | Example |
|---|---|---|
| `id` | Google Place ID | `ChIJN1t_tDeuEmsRUsoyG83frY4` |
| `url_place` | Google Maps URL | `https://www.google.com/maps/place/?q=place_id:...` |
| `title` | Business name | `Joe's Pizza` |
| `category` | Business category | `Pizza restaurant` |
| `address` | Full address | `123 Main St, New York, NY 10001` |
| `phoneNumber` | Local phone number | `(555) 123-4567` |
| `completePhoneNumber` | International phone number | `+1 555-123-4567` |
| `domain` | Website domain | `joespizza.com` |
| `url` | Website URL | `https://www.joespizza.com` |
| `coor` | Geographic coordinates | `40.7128,-74.0060` |
| `stars` | Average star rating | `4.5` |
| `reviews` | Total review count | `234` |
| `source_query` | Original search query | `pizza in New York` |

### Enriched CSV

The enriched file includes all 13 raw columns plus 12 enrichment columns:

| Column | Type | Description |
|---|---|---|
| `has_phone` | bool | True if a phone number is present. |
| `has_website` | bool | True if a domain or website URL is present. |
| `domain_valid` | bool | True if the domain passes syntax validation. |
| `rating_score` | float | Rating multiplied by the logarithm of review volume. |
| `review_density` | float | Normalized review count relative to the batch maximum (0 to 1). |
| `web_has_contact` | bool | True if a contact page was detected on the website. |
| `web_has_services` | bool | True if a services or products section was detected. |
| `web_keywords` | str | Comma-separated list of top content keywords. |
| `web_is_modern` | bool | True if modern frontend signals (React, Vue, Next.js) were detected. |
| `web_scraped` | bool | True if the website was reachable and inspected. |
| `score` | float | Overall lead score from 0 to 100. |
| `segment` | str | Business size segment: micro, small, medium, or large. |

## Supported languages and countries

The `--lang` flag sets Google interface language (`hl`), and `--country` sets geolocation (`gl`).

### Common languages and countries

| Code | Language | Code | Country |
|---|---|---|---|
| `en` | English | `us` | United States |
| `es` | Spanish | `es` | Spain |
| `fr` | French | `fr` | France |
| `de` | German | `de` | Germany |
| `it` | Italian | `it` | Italy |
| `pt` | Portuguese | `br` | Brazil |
| `ja` | Japanese | `jp` | Japan |
| `ko` | Korean | `kr` | South Korea |
| `zh` | Chinese | `cn` | China |

## Troubleshooting

Empty results or consent wall:
- Google may present a consent page if geolocation does not match the request. Match `--lang` and `--country` to the target area.

Enriched CSV contains empty website fields:
- If a website is down or blocks automated requests, `web_scraped` is set to False. The pipeline records empty values and continues. Increase `--web-timeout` to 20 seconds for slow websites.

Parsing errors:
- If Google changes the internal response structure, run with `--log-level DEBUG` to inspect the raw response payload.

## License

This project is provided for educational and research purposes. Users must comply with Google Terms of Service and applicable local regulations.
