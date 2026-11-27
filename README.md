<div align="center">

# What's up with the AI, Data and Software Engineering job market in Austria, Germany and Switzerland?

[![tests](https://img.shields.io/github/actions/workflow/status/Chrisinho8/DACH-data-job-market/tests.yml?branch=main&label=tests&labelColor=221F1A&color=6FA37F&style=flat-square)](https://github.com/Chrisinho8/DACH-data-job-market/actions/workflows/tests.yml)
[![live postings](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fraw.githubusercontent.com%2FChrisinho8%2FDACH-data-job-market%2Fmain%2Fdocs%2Fdata%2Fmeta.json&query=%24.live_postings&label=live%20postings&labelColor=221F1A&color=E3A23F&style=flat-square&cacheSeconds=1800)](https://Chrisinho8.github.io/DACH-data-job-market/)

**[Open the live tracker](https://Chrisinho8.github.io/DACH-data-job-market/)** • [![updated](https://img.shields.io/badge/dynamic/json?url=https%3A%2F%2Fraw.githubusercontent.com%2FChrisinho8%2FDACH-data-job-market%2Fmain%2Fdocs%2Fdata%2Fmeta.json&query=%24.updated&label=updated&labelColor=221F1A&color=E56743&style=flat-square&cacheSeconds=1800)](https://Chrisinho8.github.io/DACH-data-job-market/)

</div>

A daily snapshot of the DACH AI, data and software-engineering job market: **3,500+ live postings from 1,500+ employers** across Germany, Austria and Switzerland. Every day at 08:00 UTC+2 a Databricks pipeline queries the Adzuna API for 21 AI, data and software-engineering job titles in all three countries, deduplicates and classifies the results through a bronze/silver/gold Delta Lake. It then gets scanned against a curated 87-term skills dictionary and published to a static GitHub Page.

## How the tracker might help you

- **See the shape of the market** - volumes by role, city and country, how long postings stay open, seniority mix, tool mentions and many more.
- **Read the postings behind any number.** Every chart is a count of real postings, and those postings are browsable: search by title or employer, filter by role, country, city and seniority, sort by newest or longest-open.
- **Watch it move.** Nothing is ever overwritten, so each run adds to the record and every figure gains a second dimension: not just how many AI roles are open, but whether that number is climbing or declining.

## Overview of the map

Demo of 06/11/2026
 

<p align="center">
  <img src="assets/map-demo.gif" width="820"
       alt="Filtering the DACH map by role and switching between the bubble and heat-map views">
</p>


## Main take-aways of the data (State of 20.08.2026)

**Germany is the biggest market.** 3,133 of 3,625 postings are German - 86.4%, against 6.2% Austrian and 7.3% Swiss. The blue and green clusters on the map are Vienna, Graz, Linz, Innsbruck and the Zurich-Bern-Geneva line, and that is close to all of them. Any single-country claim about AT or CH rests on a few dozen adverts per family.
 
**Hiring is concentrated in a handful of cities.** Berlin (422) and Munich (344) are 21% of DACH between them; the top five are 36%. The rest is spread across 250-odd cities - the scatter of small bubbles through the Rhine-Ruhr and the south - most of them carrying single-digit counts.
 
**Speed varies more by city than by role.** Düsseldorf clears in 48 days and Munich in 49, while Frankfurt sits at 80, Zurich at 104 and Vienna at 105. The two non-German capitals are the slowest markets on the board, at more than twice Munich's pace, and the red figures in the "Avg days" column are where that shows.
 
**Berlin and Munich are where AI hiring is the most prominent.** 37% of Berlin's postings and 42% of Munich's are AI-related roles, against 19% in Hamburg and 13% in Düsseldorf. Berlin is also the most international: 58.5% of its postings are in English, versus 14.9% in Düsseldorf.


 
## Scope: what counts as an AI, Data or Software Engineering job

**Data:** data engineer, data analyst, data scientist, data architect, analytics engineer, DWH / ETL, data governance, data consultant, BI developer.

**AI:** AI engineer, GenAI / LLM, ML engineer, MLOps, AI consultant, AI research.

**Software engineering:** software engineer, backend, frontend, fullstack, mobile, DevOps / cloud / SRE.

**Excluded:**  Titles saying "AI"/"KI" and naming no role, Data centre infrastructure, Embedded / firmware and QA / test engineering, 
Speculative applications.

Note: Excluded rows land in `silver.excluded` so the decision stays auditable.

## Architecture of the pipeline

```
Adzuna API              de · at · ch, 21 job titles   (GitHub Actions port: de · ch)
    │
    ▼
Unity Catalog volume    raw JSON cached, one file per query
    │
    ▼
Auto Loader             trigger availableNow
    │
    ▼
BRONZE    Delta         append-only, every snapshot preserved
    │
    ▼
SILVER    Delta         parsed, classified, scope-filtered,
    │                   deduplicated, quality-gated
    ▼
GOLD      Delta         aggregates + daily history,
    │                   postings_public (the one row-level table)
    ▼
docs/data/*.json        committed to this repo
    │
    ▼
GitHub Pages            static site, no backend
```


## Limitations of this tracker


- **`created` is the aggregator's date**, possibly when it indexed the posting rather than when the employer published it.
- **Descriptions are capped at 500 characters** and 99.6% are truncated. Tool counts measure *mentioned in the title or opening paragraph* - a floor, not a requirement rate. The window is identical for every posting, so comparing tools holds; absolute rates do not.
- **AI counts are a floor**, for the reason in the scope table. The dropped share is printed on every run of `03_silver_clean.py`; if it climbs, the six rules are going stale.
- **A posting in the browser is not necessarily open.** It was live at the last refresh. Nothing re-checks whether it has since been filled or withdrawn.
- **There is no remote or on-site distinction.** The aggregator returns a single city per listing and no remote flag, so the pipeline cannot tell a remote job from an on-site one.
- **There is no remote or on-site distinction.** This means, that jobs that are listed in more than one city, are also displayed in more than one city. I thought about putting up a threshold number, but the inflation is fairly minimal so I left it like this.


## Running it yourself

### Option 1: GitHub Actions (no Databricks)

`src/pipeline.py` merges two sources and rebuilds every `docs/data/*.json` file the site reads:

- **AI and data postings** come from the [upstream tracker](https://chrisinho8.github.io/DACH-data-job-market/) by Christoph Ruckensteiner (MIT): its published `docs/data/postings.json`, already classified. Fetching it costs no Adzuna calls.
- **Software engineering postings** come from this repo's own Adzuna queries (9 titles, Germany and Switzerland), stored in `state/swe_postings.json`.

`.github/workflows/pipeline.yml` runs in two modes:

| Trigger | Budget | What it does |
|---|---|---|
| Daily at 06:00 UTC | 0 calls | Fresh upstream AI/data + the stored SWE snapshot, ages moved forward |
| Actions → pipeline → Run workflow | 25 calls (default) | Also refreshes the SWE snapshot |

25 calls is the Adzuna free tier's monthly allowance: page 1 of all 18 SWE queries plus 7 deeper German pages, about 500–1,000 SWE postings. The script waits 13 s between calls to stay under the free tier's 5 calls/minute (`CALL_PACE` variable to change it).

Setup: add `ADZUNA_APP_ID` and `ADZUNA_APP_KEY` under **Settings → Secrets and variables → Actions**, then run the workflow manually once.

Austria is left out on both sides so every country carries every role group. SWE figures are as old as the last manual run (`swe_updated` in `meta.json`), while AI/data refresh daily. Upstream rows carry no salary, so "% with salary" is computed over the SWE rows only.

The title rules in `src/classify.py` are copied from `notebooks/03_silver_clean.py`. Change both together; `tests/test_classify.py` checks them.

### Option 2: Databricks

1. Get an `app_id` and `app_key` from `developer.adzuna.com`.
2. Create a Databricks workspace (Free Edition is enough) and run `notebooks/setup_uc.py` to create the catalog, schemas and volumes.
3. Put credentials in the `conf` volume, outside this repo:

```python
pathlib.Path("/Volumes/jobs/bronze/conf/adzuna.json").write_text(
    json.dumps({"app_id": "...", "app_key": "..."}))
```

4. Import `notebooks/` and run 01 to 06 in order.
5. Chain them into a daily Databricks Workflow-


## Tech stack

![Databricks](https://img.shields.io/badge/Databricks_Free_Edition-221F1A?style=flat-square&logo=databricks&logoColor=E56743)
![Python](https://img.shields.io/badge/Python-221F1A?style=flat-square&logo=python&logoColor=E3A23F)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-221F1A?style=flat-square&logo=databricks&logoColor=6FA37F)
![PySpark](https://img.shields.io/badge/PySpark-221F1A?style=flat-square&logo=apachespark&logoColor=E56743)
![Databricks Workflows](https://img.shields.io/badge/Databricks_Workflows-221F1A?style=flat-square&logo=databricks&logoColor=E3A23F)
![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-221F1A?style=flat-square&logo=githubactions&logoColor=6FA37F)
![GitHub Pages](https://img.shields.io/badge/GitHub_Pages-221F1A?style=flat-square&logo=github&logoColor=E56743)
![SQL](https://img.shields.io/badge/SQL-221F1A?style=flat-square&logo=postgresql&logoColor=6FA37F)
![pytest](https://img.shields.io/badge/pytest-221F1A?style=flat-square&logo=pytest&logoColor=6FA37F)
![JavaScript](https://img.shields.io/badge/Vanilla_JS-221F1A?style=flat-square&logo=javascript&logoColor=E3A23F)


## Licence

MIT. The underlying job data belongs to Adzuna and its source boards.

---

<p align="center">Built by <b>Christoph Ruckensteiner</b> · <a href="https://www.linkedin.com/in/cruckensteiner112">LinkedIn</a> · <a href="https://github.com/Chrisinho8">GitHub</a></p>
<p align="center">Software engineering edition by <b>Shahriar Yazdipour</b> · <a href="https://yazdipour.com/">yazdipour.com</a> · <a href="https://github.com/yazdipour">GitHub</a></p>
