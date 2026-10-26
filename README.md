# AdScrape

Scrapes the [Post Affiliate Pro Affiliate Program Directory](https://www.postaffiliatepro.com/affiliate-program-directory/),
adds company data (HQ, revenue, headcount) for each program, and filters the
results against your criteria. Output is CSV plus a formatted Excel workbook.

## Setup

Requires Python 3.10+.

```sh
pip install -r requirements.txt
```

### API keys (optional)

Company data comes from the first provider that returns a result, in this order:

| Provider | Env var | Notes |
|---|---|---|
| Apollo | `APOLLO_API_KEY` | Paid; enrich by domain |
| People Data Labs | `PDL_API_KEY` | Paid; enrich by website |
| Wikidata | none | Free; mostly covers larger and public companies |

Put keys in a `.env` file next to the script (one `KEY=value` per line).
Real environment variables take precedence. Providers without a key are skipped.

```
APOLLO_API_KEY=your-key-here
```

## Usage

```sh
python pap_directory_scraper.py                     # full scrape + enrich
python pap_directory_scraper.py --max-pages 2       # quick test
python pap_directory_scraper.py --limit 200         # first 200 programs only
python pap_directory_scraper.py --no-enrich         # directory data only, no API calls
python pap_directory_scraper.py --max-lookups 80    # spend at most 80 paid lookups
python pap_directory_scraper.py --refresh           # re-download directory pages for new programs
```

### Filters

Programs that fail a filter are kept in `programs_all.csv` with the reason in
**Why Not Qualified**. Directory-only filters run first, so rejected programs
don't use paid lookups.

| Flag | Effect |
|---|---|
| `--min-employees N` | Reject below N employees, or if headcount is unknown |
| `--min-revenue USD` | Reject below this annual revenue, or if revenue is unknown |
| `--hq-countries "A,B"` | Only keep companies headquartered in these countries |
| `--exclude-industries "A,B"` | Reject industries containing any of these (partial match) |
| `--exclude-keywords "A,B"` | Reject if the name or description contains any of these words |
| `--no-explicit` | Reject programs that allow explicit content |
| `--require-domain` | Reject programs with no company domain |
| `--delay S` | Seconds between directory requests (default 1.5) |

Example:

```sh
python pap_directory_scraper.py --min-employees 50 --min-revenue 10000000 \
    --hq-countries "United States,Canada,United Kingdom" --exclude-keywords "casino,betting"
```

## Output

| File | Contents |
|---|---|
| `programs_all.csv` | Every program, enriched |
| `programs_qualified.csv` | Only programs that pass your filters |
| `programs.xlsx` | Summary, All Programs, and Qualified sheets with links and number formatting |
| `cache.json` | Downloaded pages and lookup results |

If an output file is open in Excel, the run saves to a timestamped copy instead.

## Caching

`cache.json` stores every downloaded page and every enrichment result per
provider, so re-runs are fast and don't repeat paid lookups. `--refresh` clears
only the cached pages and keeps lookup results. Delete `cache.json` to start
from scratch.

The scraper honours the directory's `robots.txt` and backs off on rate limits.
If a provider fails three times in a row (bad key, out of credits), it's turned
off for the rest of the run, and the failed lookups are retried next time.

## How company domains are found

Each program's domain comes from the affiliate manager's contact email on its
directory page. Free-mail addresses (Gmail, Outlook, etc.) are ignored, so those
programs can't be enriched.
