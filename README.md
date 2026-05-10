# Global Patent Intelligence Pipeline


An end-to-end data-engineering project that turns raw USPTO
PatentsView bulk data into a clean SQLite warehouse, publication-
quality charts, three flavours of reports, and an interactive Plotly
dashboard.

```
PatentsView bulk files → Python → pandas → SQLite → SQL → Reports / Charts / Dashboard
```

**Description **

- SQLite warehouse with seven fully-indexed tables
- 9 analytical SQL queries (7 required + 2 bonus CPC queries)
- Console report with pretty `rich` tables
- 6 CSV exports + one JSON summary in `reports/`
- 4 publication-quality PNG charts in `reports/charts/`
- A branded, interactive **Plotly dashboard** with KPI cards, filters,
  a world map, tabs, a live SQL browser, and a **TF-IDF** text tab (sample patents)
  
---

## reproducibility

```powershell
git clone <your-repo-url>
cd big_data_patent

python -m venv .venv
.\.venv\Scripts\Activate.ps1            # Windows
# source .venv/bin/activate             # macOS / Linux

pip install -r requirements.txt
python -m src.run_all                   # download → clean → load → report → charts

# launch the dashboard
streamlit run src/dashboard.py
```

The dashboard opens at <http://localhost:8501>.

---

## What gets downloaded

Default mode downloads only two PatentsView files
| File | Size | Used for |
|------|------|----------|
| `g_patent.tsv.zip`                 | 230 MB | Patents (title, grant date, year) |
| `g_location_disambiguated.tsv.zip` |   3 MB | Real country distribution |


### Opting in to the real data

Flip the flags in `src/config.py` and rerun `python -m src.run_all`:


## Project layout

```
patent-intel-pipeline/
├── data/
│   ├── raw/                   # downloaded zips (gitignored)
│   └── processed/             # clean_*.csv (committed - tiny)
├── db/patents.db              # SQLite (gitignored, self-heals)
├── reports/
│   ├── top_inventors.csv, top_companies.csv, ...
│   ├── report.json
│   └── charts/
│       ├── trend.png
│       ├── top_countries.png
│       ├── top_companies.png
│       ├── top_inventors.png
│       └── cpc_sections.png   # only when USE_CPC = True
├── sql/
│   ├── schema.sql             # tables + indexes + CPC section seed
│   └── queries.sql            # Q1..Q9
├── src/
│   ├── config.py              # flags, paths, branding, tuning knobs
│   ├── download.py            # streaming downloader w/ progress bar
│   ├── clean.py               # pandas cleaning + synthetic + CPC
│   ├── load.py                # builds SQLite from clean CSVs
│   ├── analyze.py             # parses queries.sql and runs each Qn
│   ├── report.py              # console + CSV + JSON outputs
│   ├── plot.py                # matplotlib PNG charts
│   ├── dashboard.py           # Streamlit + Plotly interactive dashboard
│   └── run_all.py             # one-command orchestrator
├── .streamlit/config.toml     # branded dashboard theme
├── requirements.txt
└── README.md
```

---

## Database schema

```sql
patents         (patent_id PK, title, abstract, filing_date, year)
inventors       (inventor_id PK, name, country)
companies       (company_id PK, name)
patent_inventor (patent_id, inventor_id)          -- M:N
patent_company  (patent_id, company_id)           -- M:N
cpc_sections    (section_code PK, description)    -- seeded A..H, Y
patent_cpc      (patent_id, section_code, subclass)
```

The brief showed a single "relationships" table; patents have
**independent** many-to-many relationships to inventors and companies,
so splitting them avoids a Cartesian cross-product.

See [`sql/schema.sql`](sql/schema.sql) for the DDL, including the
indexes that make every query run in under a second.

---

## The queries

All in [`sql/queries.sql`](sql/queries.sql), parsed at runtime by
`src/analyze.py` so `dashboard.py` and `report.py` share one source of
truth.




## Reports produced

### A. Console (via `rich`)
Runs during `python -m src.report` — prints a banner, a KPI line, and
one pretty table per query.

### B. CSV (six files)
`top_inventors.csv`, `top_companies.csv`, `top_countries.csv`,
`country_trends.csv`, `companies_avg_per_year.csv`,
`inventor_rank_by_country.csv` — all in `reports/`.

### C. JSON (`reports/report.json`)
```json
{
  "generated_at": "2026-04-21T10:09:33+00:00",
  "total_patents": 100000,
  "top_inventors":  [{"name": "...", "country": "US", "patents": 8352}, ...],
  "top_companies":  [{"name": "Sony Group", "patents": 5672}, ...],
  "top_countries":  [{"country": "US", "patents": 51940, "share": 0.42}, ...],
  "patents_per_year":[{"year": 2020, "patents": 20000}, ...]
}
```

### D. Charts (`reports/charts/*.png`)
Produced by `python -m src.plot` — matplotlib, 150 DPI, branded,
publication-quality. Trend line, top countries / companies / inventors
and (when `USE_CPC=True`) a donut of CPC section share.

---
