# International Migration — Stocks & Flows (2011–2021)

**Short summary (TL;DR)**  
This project uses OECD/UN data to build international migration metrics for the 2011–2021 period, including **inflow / outflow / stocks / asylum / nationality**, using an ETL → SQL-first EDA → Power BI dashboard workflow. Goal: to demonstrate data engineering + data analysis + BI series in a single, repeatable work.

---

# Contents
1. Project Summary  
2. Short Timeline 
3. Folder Structure   
4. Reproducibility — How to run it 
5. Key Scripts and explanations  
6. SQL & EDA — Queries and sample output 
7. Power BI — Report pages, visuals, required DAX measures  
8. Data Quality — Issues found & numerical summaries  
9. Key Findings (short, impactful)   
10. License & Attribution

---


# 1) Project objective
The objective of this project is to make migration data sourced from the OECD/UN **accurate, repeatable, and processable**; to answer analytical questions using SQL and prepare professional dashboards using Power BI. An end-to-end (data → BI) case study for portfolio/LinkedIn sharing.

---

# 2) Short timeline — what you did (step by step, in one sentence)
- Environment setup: `.venv` was created in the project root, dependencies were installed with `pip install -r requirements.txt`.  
- Raw data was placed in `data/raw/` (OECD xlsx & additional CSV).  
- Raw files were converted to Parquet and SQLite using `src/ingest/ingest_all.py`.  
- Corrupted/special files (duplicate columns, filtered Excel) were detected and cleaning code was added.  
- DB structure and table row checks were performed using `src/analysis/inspect_db.py`.  
- Ran a quick EDA script using `src/analysis/eda.py` to generate `docs/global_inflows_trend.png`.  
- Manual: Identified and converted merged headers and meta rows such as “% of total population” in some xlsx files.  
- Clean CSV files were loaded into SQLite using `load_to_db.py`, and `fact_migration` + views were created.  
- A Power BI report (5 pages) was created: Global Overview, Top Countries, Country Profile, Asylum Spikes, Data Quality.  
- README, docs, and SQL analysis files were prepared.

---

```
# 3) Folder structure (in the project root)

International Migration Stocks and Flows/
├── data/
│ ├── raw/ # Original (.xlsx, .csv)
│ ├── clean/ # Cleaned CSV/XLSX
│ └── warehouse/ # migration.db (SQLite)
├── dashboards/
│ ├── MigrationDashboard.pbix
│ ├── MigrationDashboard.pdf
│ └── screenshots/
├── docs/
│ ├── methodology.md
│ └── global_inflows_trend.png
├── sql/
│ ├── inflows_analysis.sql
│ ├── outflows_analysis.sql
│ ├── stocks_analysis.sql
│ ├── joins_analysis.sql
│ ├── asylum_analysis.sql
│ ├── nationality_analysis.sql
│ └── data_quality_checks.sql
├── src/
│ ├── ingest/ingest_all.py
│ ├── ingest/load_to_db.py
│ ├── processing/clean_dimensions.py
│ └── analysis/eda.py
├── requirements.txt
└── README.md
```

# 4) Reproducibility — step by step (Windows PowerShell example)

**Preparation**
```powershell

cd “C:\Users.........\International Migration Stocks and Flows”
python -m venv .venv
# If a PowerShell policy issue arises:
# Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1         # or .venv\Scripts\activate.bat (cmd)
python -m pip install --upgrade pip
pip install -r requirements.txt


# Example:
Copy-Item “C:\Users......\*.xlsx” -Destination “.\data\raw\”

python src\ingest\ingest_all.py
# Expected: data/processed/*.parquet files are created and data/warehouse/migration.db is prepared.


import sqlite3, pandas as pd
conn = sqlite3.connect(‘data/warehouse/migration.db’)
print(pd.read_sql_query(“SELECT name FROM sqlite_master WHERE type=‘table’”, conn))
conn.close()


python src\analysis\eda.py
# This script should produce docs/global_inflows_trend.png.



Notes:

If you see a duplicate column name error, add duplicate-handling to the clean_dataframe() function in the CSV reading/column cleaning step.

Temporarily remove or re-download corrupted .xlsx files (openpyxl style errors) from data/raw.

```
# 5) Important scripts — what they do

src/ingest/ingest_all.py — reads xlsx/csv files in data/raw, cleans them, produces data/processed/*.parquet, and writes to the SQLite DB. (Cache: parquet)

src/ingest/load_to_db.py — Takes cleaned CSV files (data/clean/*_clean.csv) and loads them into SQLite (with row summaries).

src/processing/clean_dimensions.py — Utility functions that normalize country names (mapping, trim, case).

src/analysis/inspect_db.py — Quickly displays tables, row counts, and sample rows in the DB.

src/analysis/eda.py — Automated EDA: Generates a sample global inflow trend graph (docs/global_inflows_trend.png).


6) SQL & EDA — important queries

Note: Table names were loaded into the DB as *_clean. Verify the table name before running it with SELECT name FROM sqlite_master...

```

    1) Determining the last year

        SELECT MAX(Year) AS latest_year FROM inflows_of_foreign_population_clean;

    2) Top Countries

        WITH ly AS (SELECT MAX(Year) AS y FROM inflows_of_foreign_population_clean)
        SELECT Country, SUM(Value) AS total_inflow
        FROM inflows_of_foreign_population_clean
        WHERE Year = (SELECT y FROM ly)
        GROUP BY Country
        ORDER BY total_inflow DESC
        LIMIT 20;

    3) GLOBAL YEAR Total Inflow (Trend & Yoy)

        SELECT Year, SUM(Value) AS total_inflow
        FROM inflows_of_foreign_population_clean
        GROUP BY Year
        ORDER BY Year;

        And YoY:

            SELECT Year,
                total_inflow,
                ROUND( (total_inflow - LAG(total_inflow) OVER (ORDER BY Year)) / LAG(total_inflow) OVER (ORDER BY Year) * 100, 2) as pct_change
            FROM (
                SELECT Year, SUM(Value) AS total_inflow
                FROM inflows_of_foreign_population_clean
                GROUP BY Year
            ) t
            ORDER BY Year;

    4) Net migration (inflow − outflow)

        SELECT i.Country, i.Year, i.Value - COALESCE(o.Value,0) AS net_migration
        FROM inflows_of_foreign_population_clean i
        LEFT JOIN outflows_of_foreign_population_clean o
             ON i.Country = o.Country AND i.Year = o.Year;


    5) Stocks vs Inflows

        WITH ly AS (SELECT MAX(Year) AS y FROM stocks_of_foreign_population_clean)
        SELECT i.Country, i.Value AS inflow, s.Value AS stock
        FROM inflows_of_foreign_population_clean i
        LEFT JOIN stocks_of_foreign_population_clean s
            ON i.Country = s.Country AND i.Year = s.Year
        WHERE i.Year = (SELECT y FROM ly);
         
       


(Other SQL files used in the project: stocks_analysis.sql, joins_analysis.sql, asylum_analysis.sql, nationality_analysis.sql, data_quality_checks.sql — all located in /sql/).
```

# 7) Power BI — Report structure, visuals & DAX measures

Report pages (5):

A. Global Overview — KPI (Total Inflow), Line (Global Trend), Map (Country map)
B. Top Countries — Year slicer, Top 10 bar chart, detailed table
C. Country Profile — Country slicer, multi-series line (inflow/outflow/stock), KPI cards
D. Asylum Increases — Heatmap (Country × Year) + Alert table
E. Data Quality and Notes — Missing data table + methodology text box

Basic DAX measures (Power BI → Modeling → New Measure — add each as a separate measure)


```
Total Number of Entries

Total Inflow People =
CALCULATE(
  SUM(fact_migration[value_people]),
  fact_migration[measure] = "inflow"
)
```


```
Total Outflow People

Total Outflow People =
CALCULATE(
  SUM(fact_migration[value_people]),
  fact_migration[measure] = "outflow"
)
```


Net Migration People

Net Migration People = [Total Inflow People] - [Total Outflow People]



```
Inflow YoY %

Inflow YoY % =
VAR Curr = [Total Inflow People]
VAR Prev = 
  CALCULATE(
    [Total Inflow People],
    FILTER(ALL(fact_migration[Year]), fact_migration[Year] = MAX(fact_migration[Year]) - 1)
  )
RETURN
IF(ISBLANK(Prev), BLANK(), DIVIDE(Curr - Prev, Prev))
```


```
Inflow % Share (example)

Inflow % Share =
VAR TotalThisYear = CALCULATE([Total Inflow People], ALLSELECTED(fact_migration[Country]), fact_migration[Year] = MAX(fact_migration[Year]))
RETURN DIVIDE([Total Inflow People], TotalThisYear, 0)
```

```
Inflow CAGR (example)

Inflow CAGR =
VAR Y0 = MIN(fact_migration[Year])
VAR Y1 = MAX(fact_migration[Year])
VAR StartVal = CALCULATE([Total Inflow People], FILTER(ALL(fact_migration[Year]), fact_migration[Year] = Y0))
VAR EndVal = CALCULATE([Total Inflow People], FILTER(ALL(fact_migration[Year]), fact_migration[Year] = Y1))
VAR Years = Y1 - Y0
RETURN
IF(Years <= 0 || StartVal <= 0, BLANK(), POWER(DIVIDE(EndVal, StartVal), 1/Years) - 1)
```

Note: Add each measure separately as a “New Measure.” Do not paste them all into a single DAX window and try to run them — Power BI requires each measure to be registered individually.

## Sample Visuals

![Global Inflow Trend](https://github.com/ReyhanHRZ/international-migration-analysis/blob/main/A.png)
![Top 10 Countries](https://github.com/ReyhanHRZ/international-migration-analysis/blob/main/B.png)
![Asylum Heatmap](https://github.com/ReyhanHRZ/international-migration-analysis/blob/main/C.png)




# 8) Data quality — findings (brief summary)

(Summary from database run outputs)

There were ~30 missing value rows in the inflows table.

There were ~18 missing rows in the outflows table.

There were ~57 missing rows in the stocks table.
(Available in the detailed data_quality_checks.sql content; put a summary on the Data Quality page.)

Common problems and solutions:

Merged header, description rows → Promote/drop header in Power Query/ingest phase.

Duplicate column name (from CSV) → Clean duplicate column names with df.columns = make_unique().

Country naming mismatch (Turkiye/Türkiye) → Map/replace with clean_dimensions.py.






# 9) Brief findings (example sentences — for LinkedIn/README)

The total global inflow trend from 2011 to 2021 showed an upward trajectory between 2011 and 2019, followed by a sharp decline in 2020 (due to COVID-19); A partial recovery occurred in 2021 (2020: -34.37% vs. 2021: +17.64%).

Countries with the highest inflows in 2021 (in order): Germany (1139.816), United States (740.002), Turkey (615.095), Spain (456.573), Canada (405.795).

Data quality note: Some rows in the stocks table contain metrics such as “% of total population” — do not include these in the analysis without converting them to numeric.


```

Contact & References

Prepared by: Reyhan HOROZ— Junior Data Analyst / Engineer

Email: reyhanhrz53@gmail.com

LinkedIn: linkedin.com/in/reyhan-horoz-bb4839292
```






