# Python Programming Assignment 5 CO5

**Student:** [Enter full name]  
**Register number:** [Enter register number]  
**GitHub repository:** [Paste working repository URL here]  
**Python used for execution evidence:** Python 3.14.7  
**Libraries:** Python standard library and pandas 2.2.3; charts are generated as labelled SVG using standard-library string/file operations.

This report addresses all five design tasks and four implementation tasks in the city air-quality case study. The sample dataset is intentionally small for inspection; the performance task separately generates 1, 12 and 120 station-years. Generated measurements are machine-dependent. A full 120-station year contains 1,051,200 hourly readings (8,760 per station).

## Task A1 Design the data acquisition

### A1.1 Source structure and expected quality problems

Each station sends a UTF-8 CSV headed `station,date,hour,pm25,pm10,no2,temperature,temp_unit`, followed by one station-hour record. The station identity may be repeated in the row or supplied as trusted file metadata. Dates use ISO calendar form and hours range from 0 through 23. Anticipated defects include blank or `NA` readings (for example `pm25=NA`), negative sensor values (`pm10=-4`), impossible spikes (`pm25=1500`), duplicate station/hour rows, malformed dates (`2026-13-40`), invalid hours (`27`), missing columns, wrong delimiters, inconsistent station spelling, invalid numbers such as `high`, short/truncated rows, and temperatures reported in Fahrenheit by two stations. Files can also be empty or encoded differently from UTF-8. The B1 fixture includes all these record and file-level cases, split across its main and adversarial input files. Each defect needs a measurable validation rule and an audit record.

### A1.2 SQLite storage design

`stations(station_id INTEGER PRIMARY KEY, station_name TEXT NOT NULL UNIQUE, temperature_unit TEXT NOT NULL CHECK(temperature_unit IN ('C','F')))` stores station metadata. `readings(station_id INTEGER NOT NULL REFERENCES stations(station_id), observed_at TEXT NOT NULL, pm25 REAL, pm10 REAL, no2 REAL, temperature_c REAL, PRIMARY KEY(station_id, observed_at))` stores one normalized reading per station and hour. `observed_at` is an ISO timestamp (`YYYY-MM-DDTHH:00:00`); using Celsius in storage removes unit ambiguity. Nullable numeric readings represent missing observations, while a separate ingestion error table records rejected rows and reasons. The CSV station name maps to `stations.station_name`, the station lookup supplies the foreign key, date and hour combine into `observed_at`, and pollutant columns map to numeric fields after validation and conversion. An index on `(station_id, observed_at)` supports station-and-time lookups.

```text
BEGIN ACQUIRE_STATION_FILE(file)
    OPEN file as UTF-8 CSV
    VALIDATE required headers and station metadata
    FOR EACH row WITH line_number
        PARSE date, hour, pollutant readings and temperature
        IF row is structurally valid THEN
            NORMALIZE station and units
            INSERT station if not already present
            INSERT reading with parameterized SQL
        ELSE
            RECORD file, line_number and validation reason
        END IF
    END FOR
    COMMIT accepted rows
END ACQUIRE_STATION_FILE
```

### A1.3 CSV and SQL for three board questions

| Board question | Repeated CSV scan | SQL query |
|---|---|---|
| Adyar mean PM2.5 in October | Open all daily files, parse date/station, accumulate matching values; O(n) per question. | `SELECT AVG(pm25) FROM readings JOIN stations USING(station_id) WHERE station_name=? AND observed_at>=? AND observed_at<?`; indexed filtering avoids re-parsing every file. |
| Highest daily PM10 by station | Scan records into a dictionary keyed by station and date, then compare; O(n). | `SELECT station_id, date(observed_at), MAX(pm10) ... GROUP BY station_id,date(observed_at) ORDER BY MAX(pm10) DESC LIMIT 1`; database performs grouping and sorting. |
| Hours with worst PM2.5 | Parse and retain/filter hourly records, then sort; O(n log n) if fully ordered. | `SELECT observed_at,pm25 ... ORDER BY pm25 DESC LIMIT 10`; index/engine can filter and order near the data. |

CSV remains useful as an interchange and archival format. SQLite is more suitable for repeated selective queries, constraints, indexes and concurrent programmatic workflows. For one small ad-hoc question, a single CSV scan is simpler and needs no schema migration or database setup.

### Alternative and complexity

With CSV plus pandas, each question loads and parses all n rows into memory, filters, then aggregates: O(n) time per question and O(n) working memory. Multiple questions repeat parsing unless the DataFrame is retained, which itself uses O(n) memory. SQLite pays one ingestion cost, stores rows on disk, and can use a B-tree index to locate a station/time range in about O(log n + k), where k is the matching result count. Indexes need additional disk and update work. CSV is simple and portable; SQL storage gives faster repeated selective queries and integrity checks.

## Task A2 Design cleaning and normalisation

### A2.1 Cleaning rules

| Defect | Rule | Reason |
|---|---|---|
| Missing pollutant reading | Forward-fill the latest valid reading for that station for a short isolated gap; if no prior reading exists, use that station's valid median. Keep an imputation flag. | Preserves a usable timeline; flag makes estimated values distinguishable. For a long gap, keep NULL instead of inventing a trend. |
| Negative or implausible concentration | Reject the measurement as invalid, retain the row in the audit log, and never clamp it to zero. | A faulty sensor must not be disguised as clean air. A domain threshold should be approved and versioned. |
| Duplicate station/hour | Keep the first valid reading and log the duplicate, unless a documented aggregation rule is approved. | Avoids double-counting and preserves reproducibility. |
| Fahrenheit temperature | Convert using `(F - 32) * 5 / 9`, round only for display, and store the full precision in Celsius. | All stations become comparable in one unit. |
| Invalid date, hour, or numeric text | Reject that field/record with filename, line number and reason. | Keeps database constraints and charts meaningful. |

The small B1 program demonstrates PM2.5 validation, station-wise forward fill, Fahrenheit conversion, duplicate removal and a cap at 999. When all earlier values are missing, a median fallback is used. The full design also retains all pollutant columns and imputation flags.

### A2.2 Normalisation

For valid PM2.5 values `x`, min-max scaling is `x' = (x - min(x))/(max(x)-min(x))`; its output is in [0,1] and is useful for bounded visual scales or distance-based models. If all values are equal, return zero for every value to avoid division by zero. Z-score standardisation is `z=(x-mean(x))/population_standard_deviation(x)`; it centres values at zero with unit spread and suits methods sensitive to feature scale. If standard deviation is zero, return zero. Fit min/max or mean/deviation on training data only and reuse those parameters on later periods to avoid data leakage. Both methods are sensitive to extreme sensor outliers, so clean faults first.

### A2.3 Standard formats

Use station IDs as stable keys and display names in trimmed title case; retain an alias table instead of fuzzy matching silently. Store dates as ISO 8601, with hourly timestamps `YYYY-MM-DDTHH:00:00` in UTC or with an explicit time zone. Store pollutant values as finite numeric values in documented units (`µg/m³`), temperature in Celsius, and use decimal points with no thousands separators in CSV. Represent missing values consistently as empty CSV fields and SQL NULL; retain a separate quality/imputation status.

```text
BEGIN CLEAN_READINGS(rows)
    SORT rows BY station and observed_at
    FOR EACH row
        NORMALIZE station, timestamp and temperature unit
        IF required identifiers or values are invalid THEN LOG and CONTINUE
        IF pollutant is missing THEN forward-fill short gap or median fallback; FLAG estimate
        IF pollutant is outside approved bounds THEN LOG and CONTINUE
        IF station/time duplicate THEN apply documented keep-first rule and LOG
        EMIT normalized row in canonical units
    END FOR
END CLEAN_READINGS
```

### Alternative and complexity

Dropping missing rows takes O(n) time and can reduce an hourly average bias if missingness is related to pollution or weather; it also reduces sample counts. Forward fill is O(n) and O(1) extra working memory per station while streaming, but can flatten peaks during a long outage. Computing a median by sorting all valid values is O(n log n) time and O(n) space; median-of-medians or selection can reach O(n) expected/linear time depending on the implementation, but still needs retained data unless an approximate streaming quantile is used. The report's imputation choice keeps a clearly flagged estimate, while a sensitivity comparison should report dropped and imputed averages side by side.

## Task A3 Design the analysis and visualisation

### A3.1 Chart choices

| Question | Chart | Why |
|---|---|---|
| PM2.5 trend over a month | Line chart, one series per station or selected station | Time order and changing levels are visible; label date and concentration units. |
| Compare stations | Bar chart of mean or median with valid sample count; box plot when distribution matters | Bar supports simple comparisons; box plot reveals spread and outliers hidden by a mean. |
| Temperature versus PM2.5 | Scatter plot with a trend line only if its assumptions are stated | Shows association, clusters and nonlinear patterns without implying causation. |
| Hours with worst air | Hour-of-day bar chart of mean/median with counts; histogram for concentration distribution | Aggregates recurring peak hours; a histogram describes a single variable's distribution. |

Do not compare station means without reporting missingness and the number of valid readings. Extreme but valid episodes should be visible and investigated, not silently removed.

### A3.2 Daily summary table

The summary table is keyed by `(station_id, day)` and contains `minimum_pm25`, `maximum_pm25`, `mean_pm25`, `valid_reading_count`, and optionally `missing_count` and `imputed_count`. SQL implementation: filter to validated numeric values, then `GROUP BY station_id, date(observed_at)` and calculate `MIN`, `MAX`, `AVG`, and `COUNT`. Preserve unrounded values in storage and round only for display. SQL aggregation streams rows with modest working memory; the Python alternative is described below.

```text
BEGIN BUILD_DAILY_SUMMARY(readings)
    GROUP valid readings BY station_id and calendar day
    FOR EACH group
        CALCULATE minimum, maximum, mean and valid count
        STORE one summary record
    END FOR
END BUILD_DAILY_SUMMARY
```

### Alternative and complexity

Python loops read n valid rows and update an accumulator dictionary keyed by station/day; time is O(n), space O(s×d) for s stations and d days. SQL `GROUP BY` also scans O(n) rows, with grouping/index work and a query plan dependent on indexes; it can spill temporary state to disk and keep aggregation close to stored data. A Python implementation that stores all rows first and then groups is still O(n) time but uses O(n) space. For daily automation and reuse of the database index, SQL is preferable; for one compact exported file, Python is portable.

## Task A4 Plan debugging and profiling

### A4.1 Debugging plan

Use temporary `print()` calls to inspect a few raw records, normalized station IDs, parsed units and counts during an early prototype; remove or gate them before processing a full year. Use assertions for invariants such as `0 <= hour <= 23`, finite nonnegative accepted values, unique station/hour primary keys and normalized temperatures. Send progress and recoverable bad-row details through `logging` with filename, line, level and reason; use DEBUG for diagnostic samples and INFO for stage counts. Run the failing input under `pdb` or an IDE breakpoint at the parser, inspect the current row and exception, and step through the unit conversion and database insert. Do not catch programming errors as malformed data.

### A4.2 Profiling plan

Measure end-to-end stages with `time.perf_counter()` around ingestion, cleaning, insertion, aggregation and chart export. Use `timeit` on isolated parsing/conversion functions with representative valid and invalid rows, repeated enough for stable medians. Run `cProfile` on the 120-station synthetic workload and sort by cumulative time to identify whether CSV parsing, Python grouping, SQLite inserts or plotting dominates. Record Python/library versions, row count, repetitions and cold/warm database state. Optimize only the measured bottleneck, and rerun correctness checks after changes.

```text
BEGIN PROFILE_PIPELINE(input)
    START high-resolution timer
    RUN ingestion, cleaning, database and chart stages
    RECORD elapsed time for each stage
    RUN timeit on isolated parsing operation
    RUN cProfile and SORT functions by cumulative time
    REPORT the dominant measured stage with row count and environment
END PROFILE_PIPELINE
```

### Alternative and complexity

Replacing logging and debugger use with thousands of `print()` calls still visits each row once (O(n) algorithmic work), but synchronous terminal output can dominate runtime and bury useful messages. `logging` adds small per-call checks and can be set to WARNING/ERROR or disabled by configuration without editing application code; DEBUG diagnostics can be filtered. A debugger is interactive and does not print every record; it adds no production overhead when not attached. Printing is appropriate for a tiny demonstration, while levels and structured context are safer for a scheduled ETL job.

## Task A5 Design citizen complaint AI

### A5.1 Lexicon sentiment and extractive summary

Tokenize lower-case words and score known positive and negative terms as +1 and -1. If the immediately preceding token is `not`, `never`, or `no`, flip that word's sign; sum scores and label positive, negative, or neutral. The runnable `Task05/ai_design_demo.py` implements this and demonstrates “not helpful” and “not bad.” An extractive summary splits a complaint into sentences, counts frequent informative words (ignoring short/common words), ranks sentences by overlap, selects the top k, then restores their original order. It reuses the citizen's wording rather than inventing claims. Ties are stable by sentence position.

### A5.2 Rule-based chatbot

Normalize the question, detect a supported pollutant keyword and station alias, parse an ISO date or use a clearly stated “today” date supplied by the application, then call a parameterized SQL query for the station/day. Respond with the measured value, units and timestamp, or say no reading was found. Unsupported questions receive a prompt naming the understood format. Never concatenate user text into SQL. Limitations: narrow vocabulary, station/date ambiguity, no context across turns, basic negation, no unit conversion or uncertainty, and a missing sensor value can be mistaken for no data. The demo accepts a lookup callback so database access remains separate from text rules.

```text
BEGIN ANSWER_QUESTION(question)
    NORMALIZE question and detect supported pollutant, station and date
    IF any required item is unknown THEN RETURN format guidance
    QUERY database with parameter placeholders
    IF reading exists THEN RETURN value, unit and time
    ELSE RETURN no-reading response
END ANSWER_QUESTION
```

### Alternative and complexity

A Naive Bayes text classifier needs labelled complaint data and training; prediction is roughly O(v) per message for v active features, with model memory O(classes×vocabulary). A pre-trained language model has much larger fixed memory and inference cost and may need a GPU or hosted service, but handles context, paraphrase and more complex negation better. A lexicon is O(w) in w tokens and uses O(L) memory for lexicon size L, is explainable and cheap, but fails at sarcasm, domain language and phrase scope. Start with the transparent scorer as a triage signal; route uncertain/high-impact complaints for human review and compare against labelled local data before choosing a trained model.

## Task B1 Implement and validate the cleaning pipeline

### B1.1 Before and after

`Task06/clean_pipeline.py` runs on `data/stations.csv` and `data/adversarial/`. The inputs include blank and `NA` PM2.5, negative and >999 values, Fahrenheit records, duplicate station/hour, station case variation, malformed date/hour, invalid numeric text, a short row, missing columns, wrong delimiters, UTF-16 content opened as UTF-8, and an empty file. The sample is printed before and after. Missing Adyar observations are forward-filled to 48.0; Guindy 86°F/90°F convert to 30.0°C/32.22°C; negative and 1200 readings are rejected; duplicate Velachery station/hour is retained once. Seven accepted records are written to `outputs/cleaned.csv`; quality notes list rejected, imputed and duplicate rows. For a production dataset the audit records should be persisted as CSV/SQL, rather than only printed.

### B1.2 Comparison with design

The implementation agrees with A2 on forward-fill, temperature conversion, impossible-value rejection, duplicate policy and ISO date normalization. It uses station-specific valid median only as a fallback when no earlier value exists. It illustrates PM2.5 and temperature, while the full table design also retains PM10, NO2, timestamps and imputation flags; that is a deliberate scope boundary of this compact test fixture.

### B1.3 SQLite and queries

The script writes cleaned CSV, creates a constrained `readings` table keyed by `(station, day, hour)` and indexed by `(station, day)`, inserts with one `executemany()` call, and answers: Adyar October mean (48.75); station means (Adyar 48.75, Guindy 108.0, Velachery 80.0); and the three highest hourly values (Guindy 121, Guindy 95, Velachery 80). It also writes a daily min/max/mean/count summary. The exact SQL results are in `outputs/query_results.txt` and the database is reproducible from the script.

### B1.4 Charts and interpretation

Four labelled SVG charts are generated: a multi-station daily trend, mean station bar chart, temperature/PM2.5 scatter, and PM2.5 histogram. The available seven rows show higher observed PM2.5 at Guindy than Adyar, but the tiny and partly imputed sample cannot support a monthly or city-wide conclusion. The scatter shows no reliable causal relation from seven records. The histogram is a shape summary only; more readings and counts per bin are required. These vector charts are `outputs/pm25_trend.svg`, `station_comparison.svg`, `temperature_scatter.svg`, and `pm25_histogram.svg`.

### Complete the code

`Task06/complete_clean_todo.py` copies the supplied exercise structure, fills both TODOs, and adds the required >999 test row. `csv.DictReader` filters missing/NA and negative PM2.5 then converts valid values to float; `min_max()` maps values to [0,1] rounded to three decimals and handles a constant input. Its executed output matches the expected cleaned tuples and `[0.0, 0.644, 1.0]`. The broader pipeline is in `clean_pipeline.py`.

### Alternative and complexity

The pandas alternative uses `read_csv`, `replace(['NA',''], NaN)`, numeric conversion, domain masks and vectorized min-max scaling. `Task06/pandas_scaling.py` uses the reproducible B3 CSV and reports one-run read+clean timings and raw-plus-cleaned DataFrame memory: 1,000 rows took 0.0032 s/272,132 bytes; 100,000 rows 0.0156 s/27,200,132 bytes; 1,051,200 rows 0.1501 s/285,926,532 bytes. The larger input is all valid so this benchmark measures the vectorized pipeline without removing rows; all defect cases are covered by the B1 fixtures. CSV parsing and vectorized transforms remain O(n); the DataFrame and temporary masks/columns use O(n) memory. A streaming standard-library production variant can keep per-station state and write accepted rows incrementally, using O(s) state plus audit records. Timings are one-run observations and should be repeated on target hardware.

## Task B2 Engineer adversarial data

### Adversarial files and SQL

`Task07/adversarial.py` tries a required column missing, semicolon-delimited data read as comma CSV, UTF-16 bytes decoded as UTF-8, and a zero-byte input. The parser rejects the first two as missing required fields, reports `UnicodeDecodeError` for the encoding mismatch, and treats empty input as zero records. The policy is to identify format/header errors at file level and keep a clear error rather than silently accepting shifted columns. Real ingestion should record filename and exception class for each case.

For station text `Adyar' OR '1'='1`, string concatenation creates a WHERE clause with a true OR condition and returns the average of both sample rows (71.5). The `?` query treats the whole attack string as a literal station name and returns `None`. `Task07/complete_sql_todo.py` fills the supplied TODO and prints `54.0 108.0` followed by `None`. Parameterization protects values; identifiers and sort directions still need allowlists.

The lexicon outputs are recorded for sarcasm (“Great ... smoke”), double negation (“Not unhappy and not bad”), and mixed opinions (“good ... bad”). It scores sarcasm negative because of “smoke,” fails to scope the two negations, and cancels mixed sentiment to neutral. These are expected model limits, not file/database failures.

### Alternative and complexity

Interpolating a name inside `f"... WHERE station='{name}'"` lets `Adyar' OR '1'='1` change SQL structure and can disclose all rows; a placeholder preserves it as data. Both forms scan O(n) without an index, but only parameterized execution is safe. `executemany()` sends n bindings through one prepared statement/transaction and avoids repeated Python call/parse overhead; O(n) row work and O(1) extra application space when fed an iterator. Calling `execute()` in a loop is also O(n), but incurs n API/statement costs; one transaction is essential for a fair comparison. This assignment demonstrates safe executemany use in B1 rather than claiming a timing that was not measured.

## Task B3 Loops versus libraries versus SQL

### Correctness and measured scale

`Task08/compare_methods.py` generates reproducible readings with seed 42 and groups 8,760 hours per station by station/day. On the one-station sample the loop, pandas groupby and SQLite `GROUP BY` results agree to tight floating-point tolerance (365 daily groups). Then each implementation is measured at one, twelve and 120 stations.

| Stations | Rows | Python loop (s) | pandas groupby (s) | SQLite GROUP BY (s) |
|---:|---:|---:|---:|---:|
| 1 | 8,760 | 0.0010 | 0.0015 | 0.0032 |
| 12 | 105,120 | 0.0119 | 0.0150 | 0.0449 |
| 120 | 1,051,200 | 0.1301 | 0.1528 | 0.4705 |

The largest CSV is generated at `outputs/synthetic_120_stations.csv` so the experiment can be repeated. Timing includes method execution and in the SQLite case in-memory table creation/insertion plus grouping, so it is an end-to-end comparison, not a query-only benchmark. These are one-run measurements and small timings are sensitive to process and filesystem state.

### Profile and engineering trade-off

The loop method is O(n) and updates a dictionary; its aggregation state is O(stations×days), not O(n), although the benchmark deliberately materializes the source list for a fair common input. pandas performs the same grouping in compiled/vectorized internals but allocates a DataFrame and group intermediates, so memory is O(n). SQLite scans/group rows, additionally pays database insertion costs here, and stores its work in database structures. `cProfile` output is saved as `outputs/cprofile.txt`; the measured cumulative profile identifies `loop_csv` as the dominant Python function at about 0.120 s on the 1,051,200-row input. For a quick one-time analysis, pandas is concise when already installed and data fits RAM; for daily automated reports, SQLite with durable indexed storage and SQL aggregation is easier to query repeatedly. For a streaming constrained host, a running sum/count dictionary avoids retaining raw records.

### Alternative and complexity

Storing all readings then grouping has O(n) input space and O(n) work. A single-pass dictionary of `(sum,count)` per station/day also has O(n) time but O(s×d) state. The loop implementation itself is that streaming aggregation over a materialized benchmark fixture; a file-streaming version would reduce memory to group state. This is the better design when reports need means only; retaining rows remains necessary for percentiles, plots and later queries unless SQLite holds the source.

## Task B4 Final engineering decision

### B4.1 Complaint methods and agreement

`data/complaints.csv` contains 50 labelled examples, including positive, negative, neutral/mixed and negation cases. The lexicon labels 49 of 50 correctly (98% agreement) on this hand-built sample. It mislabels “The problem is not improving” as neutral because “improving” is outside the small lexicon and the negator has no known word to flip. `Task09/complete_sentiment_todo.py` fills the supplied scoring TODO and matches the four expected labels. The extractive summary chooses five sentences with frequent informative words and preserves their original order; the first-five baseline preserves the earliest examples but omits recurring complaints later in the file. The ranked sample captures frequent wording, but its top terms skew toward repeated positive terms and it still needs human review for coverage and urgency. Both outputs are in `outputs/complaint_summary.txt`. This small constructed sample is not evidence of general accuracy.

### B4.2 Indexed query

The experiment inserts 500,000 synthetic sensor rows, measures a station/day aggregate before an index, creates `idx_station_day` on `(station, day)`, then reports the indexed query median over seven runs and checks both results match. On this run, the unindexed call took 0.009744 s and indexed median was 0.000005 s (about 1,785× in this small local test). The initial unindexed timing is a single measurement, so the ratio is noisy; the indexed measurement uses seven repetitions. The index helps selective queries at the cost of extra disk and slower inserts/updates. A fair production comparison should repeat cold and warm runs with representative station selectivity and report the query plan.

### B4.3 Production decision

At 112 stations reporting every 15 minutes, the network adds 448 readings per hour and 10,752 per day (about 3.93 million per 365-day year), before retries or new pollutants. Use a scheduled ingestion service that validates schemas, normalizes units, records rejected rows and imputation flags, and writes batches into SQLite for a single-machine prototype or a managed relational/time-series database when multiple writers, availability and retention require it. Partition/archive by time as volume grows, index station and timestamp, retain raw source files for audit, and precompute daily summaries. Do not load the full year into a DataFrame for every query; stream ingestion and push repeated filters/aggregations to SQL. Profile representative volumes before selecting infrastructure; B3/B4 measurements are local synthetic evidence, not a forecast of production hardware.

Fifty thousand monthly complaints can be stored with timestamps, consent/privacy controls and retention limits. Keep the lexicon as an interpretable first-pass urgency/topic signal and evaluate on locally labelled complaints. At 49/50 agreement in the current small labeled sample, one classification error is still material and the sample is not representative; a trained model should only replace it after independent validation, subgroup/error analysis, monitoring and a human escalation path. A large language model brings higher memory/latency and possible service/privacy costs, so a compact supervised classifier may be preferable if it materially improves local validation.

### Alternative and complexity

Selecting the first k sentences needs O(k) time to return a prefix (ignoring tokenization). The frequency-ranked method tokenizes/counts all words and sentences, O(w + s log s) for w tokens and s sentences, and needs O(v+s) space for vocabulary counts and sentence scores. Ranking better captures later repeated issues but can select redundant sentences and discard chronology; the prefix preserves chronology but misses salient later material. On the complaint set, compare both summaries by manual coverage of the central issue; an extractive method should not be scored as a factual paraphrase generator.

## Submission checklist

Repository name: `CSA08-[RegisterNumber]-Assignment5`. Add your name, register number and working GitHub URL to the cover page and README. Keep one folder per assignment task, include your execution screenshots (the included run captures are text files, not screenshots), and commit work progressively. Include `requirements.txt` and submit the final PDF to the course portals yourself.
