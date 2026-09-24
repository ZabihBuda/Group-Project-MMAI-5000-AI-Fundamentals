# CLAUDE.md

Guidance for Claude Code when working in this repository.

## Project overview

MMAI 5000 (AI Fundamentals) group project. It analyzes the Toronto Police Service
**Shootings & Firearm Discharges (SFD)** open dataset (2004–2022, 5,707 raw incidents).
It then predicts the number of shootings per police division per month.

The work has three parts:
1. **Data cleaning**: turn `NSA` (unknown neighbourhood) into missing values and drop rows
   where `NEIGHBOURHOOD_158` is unknown, extract `Date` from `OCC_DATE`, drop redundant
   columns, rename, remove 8 duplicates. Result: 5,635 incidents.
2. **Exploratory analysis**: incidents by neighbourhood, year, month, weekday, time of day
   and police division; deaths and injuries over time; a Folium cluster map (saved to HTML,
   not shown inline); risk levels per neighbourhood from 2018–2022 incidents per year with
   percentile cut-offs (50th/90th). Each chart has a markdown explanation.
3. **Modelling** ("Predicting Monthly Shootings by Police Division"): a division × month
   table (17 × 228 = 3,876 rows) with lag/rolling/citywide/calendar features built only
   from past months; a time-based split (train 2005–2018, test 2019–2022); two baselines
   (last month, 12-month average); Linear Regression; and a Random Forest tuned with
   `GridSearchCV` + `TimeSeriesSplit`. The Random Forest is the final model.
   Test MAE: RF 1.298, Linear 1.315, 12-month baseline 1.341.

Prophet and the old incident-level regression were removed on purpose, at the user's
request. Don't reintroduce Prophet.

## Project goals

<!-- TODO (project owner): replace with the real goals, e.g. course deliverable due date,
     portfolio polish, better model accuracy, clearer write-up. -->
- Make the analysis correct, reproducible and easy for a reader or grader to follow.
- Keep the modelling section honestly evaluated: time-based split, baselines, test metrics.

## Repository layout

| File | What it is |
|---|---|
| `MMAI_5000_AI_FUNDAMENTALS.ipynb` | The whole project (about 81 cells). Originally written in Google Colab. |
| `Group Project Data-SFD Data-Toronto Police.xlsx` | Raw dataset: sheet 1 is the data, sheet 2 is a short variable description. **Read-only; never modify it.** |
| `toronto_map.html` | Folium map exported by the notebook (about 4 MB, generated output). |
| `*.png` | Charts saved by the notebook with `plt.savefig` (generated output). |
| `README.md` | Short project description. |

There is no `requirements.txt`, no `src/` package and no tests yet. `.gitignore` covers
`.DS_Store`, `__pycache__/`, `.ipynb_checkpoints/` and `.venv/`.

## Environment

- Python 3.9+ (local machine: 3.9.13, pandas 2.0, scikit-learn 1.2). The notebook
  metadata targets Colab.
- Required packages: `pandas`, `numpy`, `matplotlib`, `seaborn`, `scikit-learn`,
  `openpyxl` (for reading the `.xlsx`), `folium`, `jupyter`.
- `openpyxl` and `folium` are **not** installed locally right now.
  Do not install packages globally without asking. Prefer a project virtual environment:
  ```bash
  python3 -m venv .venv && source .venv/bin/activate
  pip install pandas numpy matplotlib seaborn scikit-learn openpyxl folium jupyter
  ```
- To run the notebook headlessly as a check:
  `jupyter nbconvert --to notebook --execute MMAI_5000_AI_FUNDAMENTALS.ipynb --output /tmp/executed.ipynb`
  The Random Forest grid search takes about 20 seconds.

## Working with the notebook

- Once executed, the `.ipynb` file grows to several MB of saved chart outputs. Never read
  the raw JSON in full. To inspect it, read only the cell sources,
  for example with a small Python script that loads the JSON and prints `cell['source']`.
- The cells have no `id` fields, so the NotebookEdit tool can't target them. Edit with a
  Python script that loads the JSON, changes only the target cells (assert the expected
  content first), and writes back with
  `json.dumps(nb, indent=2, ensure_ascii=False) + '\n'`. That reproduces the file's
  formatting exactly, so the diff shows only real changes. Back up the file first.
- After editing the file on disk, tell the user to run **Revert File** in VS Code (or close
  the tab without saving and reopen it). Otherwise VS Code keeps showing the old version,
  and saving would overwrite the edits.
- Keep the narrative order: Data Cleaning → Exploratory Data Analysis (where / when /
  severity / risk levels) → Predicting Monthly Shootings by Police Division (Steps 1–8)
  → Conclusion.
- Count columns are named `Incidents` (not `Crimes`): the data is shootings, not all crime.
- Pair every analysis step with a short markdown cell explaining **why** it is done and
  **what the result means**. The notebook is read and graded as a report.
- Any numbers quoted in markdown must match what the code produces. If code changes,
  re-run it and update the explanation text.
- Charts need a title, axis labels with units, and readable tick labels. Call
  `plt.savefig(...)` **before** `plt.show()`, because `show()` clears the figure.
- Don't print the Folium map inline in the notebook (it bloats the file). Save it to
  `toronto_map.html` instead.

## Data facts

Raw columns: `X, Y, OBJECTID, EVENT_UNIQUE_ID, OCC_DATE, OCC_YEAR, OCC_MONTH, OCC_DOW,
OCC_DOY, OCC_DAY, OCC_HOUR, OCC_TIME_RANGE, DIVISION, DEATH, INJURIES, HOOD_158,
NEIGHBOURHOOD_158, HOOD_140, NEIGHBOURHOOD_140, LONG_WGS84, LAT_WGS84`.

- There are no nulls, but unknown locations are coded: neighbourhood `NSA` (64 rows), and
  62 of those have placeholder coordinates (0, −85.49).
- `OCC_DATE` is always 04:00 or 05:00 UTC, which is midnight Toronto time. The real hour is
  in `OCC_HOUR`. `X`/`Y` are identical to `LONG_WGS84`/`LAT_WGS84`.
- The 140- and 158-neighbourhood schemes differ for 23% of rows. They are not duplicates.
- The injury outliers are real events: 24 injured in West Hill (Jul 2012, the Danzig Street
  shooting) and 13 in Danforth (Jul 2018).

After cleaning (5,635 rows), the columns are: `Year, Month, Weekdays, Day_of_the_month,
Hour, Time` (the time-of-day range: Morning/Afternoon/Evening/Night), `Division,
Number_of_Death, Number_of_Injuries, Neighbourhood, Longitude, Latitude, Date`.

Each row is **one incident**. Counts come from `groupby`/`value_counts`, not from a
count column in the raw data.

## Known issues

All issues found in the earlier reviews have been fixed. Remaining ideas, not bugs:

- Incident counts are not adjusted for population. Rates would need City population data.
- The modelling conclusion lists possible improvements (external data, Poisson regression,
  quarterly predictions).

## Rules

- **Never modify the raw `.xlsx` file.** All cleaning happens in code.
- Only report a metric after actually computing it. If a cell was not executed, say so.
- Make small, reviewable changes. Explain *what* changed and *why* in plain language,
  because this is a learning project and the user wants to understand every change.
- Ask before large restructures (splitting the notebook, moving code into `.py` modules,
  swapping models), before installing packages, and before any `git commit` or `git push`.
- When suggesting modelling improvements, prefer methods covered in an introductory AI/ML
  course (time-based splits, cross-validation, baselines, regularized regression,
  tree ensembles, Poisson regression) and explain the trade-offs.
- Features for the model must only use information from **before** the month being
  predicted (no leakage). Never use deaths or injuries from the same period as inputs.
- This is a group project. Keep the notebook runnable top-to-bottom so teammates can
  reproduce it.
