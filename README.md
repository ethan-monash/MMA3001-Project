# MMA3001 Project: Estimating Sparse Indoor-Environment Measurements from Dense Sensor Data

Individual project for **MMA3001 – Numerical Methods and Machine Learning** (Monash University), Dataset 2: Monash smart-infrastructure environmental sensor data.

## 1. What this tool is for

Indoor air-quality sensors record some variables on every reading (CO₂, temperature, humidity, pressure) but others only some of the time (formaldehyde, light volatile organic compounds, particulate matter PM1 / PM2.5 / PM4 / PM10). Gaps in the sparse variables make it hard to monitor indoor environmental quality continuously.

This tool **estimates the sparse variables from the dense ones** and measures honestly how good those estimates are. It compares three numerical methods:

| Method | Inputs used | Idea |
|---|---|---|
| Multivariate linear regression | all 4 predictors | one linear model per target |
| Linear interpolation | 1 predictor at a time | mean target at each predictor value, joined by straight lines |
| Cubic spline | 1 predictor at a time | the same mean curve, joined by a smooth cubic spline |

All estimates are clipped to physical limits taken from the training data (never negative, never above the largest training value), and interpolants do not extrapolate.

**Inputs:** the raw sensor CSV (`id`, `sensorid`, `jsondata`, …). The `jsondata` column holds a JSON list of readings with variable names, units and values.
**Outputs:** estimated values for every row, and score tables (R², MAE, RMSE, baseline MAE, skill versus baseline) for each target, method and predictor.

**Intended use and limits.** This is a learning and evaluation project, not a calibrated measurement instrument. The results show that the four dense variables carry only modest information about the sparse ones (see the report), so estimates should not be used for safety-critical decisions.

## 2. Features

- Parses the nested JSON in the raw CSV into one column per variable, with units in the headings.
- Reproducible 75 % / 25 % train / test split (`random_state=42`).
- Three estimation methods for six targets (6 targets × 4 predictors = 24 single-predictor models, plus 6 regression models).
- Physical-limit clipping and reporting of how often it triggers.
- Common metrics for all methods: R², MAE, RMSE, baseline MAE, skill versus baseline.
- Three validation tests: **A** the random lot 1 / lot 2 split, **B** the same split with repeated readings removed, **C** a masked-gap test that withholds 10 contiguous time blocks in turn.
- Input checks with clear errors (missing columns, empty data, bad split fractions); malformed JSON rows become empty rows instead of crashing.
- Unit tests (pytest) and HTML documentation generated from docstrings (pdoc).
- Optional figures of every fitted curve against the data.

## 3. Installation

Requires Python 3.9 or newer (developed and tested on Python 3.12).

```bash
git clone https://github.com/ethan-monash/MMA3001-Project.git
cd MMA3001-Project
python -m pip install -r requirements.txt
```

Place the raw data file in a `data/` folder in the repository root, named `Environmental sensor.csv` (or `Environmental_sensor.csv`). The data is supplied by the unit and is not redistributed here.

```
MMA3001-Project/
├── data/                      # put the raw CSV here
├── sensor_methods.py          # all reusable code (documented, tested)
├── run_analysis.py            # runs everything from the command line
├── tests/test_sensor_methods.py
├── docs/                      # HTML code documentation (pdoc)
├── reports/                   # pytest test report
├── requirements.txt
├── requirements-dev.txt       # pytest, pytest-html, pdoc
├── make_docs_and_tests.sh/.bat
├── LICENSE
└── README.md
```

## 4. Quick tutorial

**Run the whole analysis (about 40 seconds):**

```bash
python run_analysis.py --plots
```

This prints the result tables and saves them to `results/` (CSV files) and `results/figures/` (PNG plots).

**Or use the code directly in Python / Jupyter:**

```python
import sensor_methods as sm

df = sm.load_environmental_data()            # reads data/Environmental sensor.csv
X_cols, y_cols = sm.get_columns(df)          # predictor and target column names
lot_1, lot_2 = sm.split_lots(df)             # 75 % train / 25 % test

# Regression: fit on lot 1, score on lot 2
models, bounds = sm.fit_regression(lot_1, X_cols, y_cols)
print(sm.evaluate_regression(lot_2, lot_1, models, bounds, X_cols, y_cols).round(3))

# Interpolation methods: one curve per (target, predictor)
tables = sm.build_interp_tables(lot_1, X_cols, y_cols)
estimates = sm.interp_rows(lot_2, tables, X_cols, y_cols)
print(sm.best_predictors(sm.evaluate_pairs(lot_2, lot_1, estimates, X_cols, y_cols)))

# Estimates for every row, clipped to physical limits
all_rows = sm.predict_rows(df, models, bounds, X_cols, y_cols)
```

### Test case

With the unit's data in `data/`, the regression test on lot 2 should give, for formaldehyde:

| rows scored | R² (clipped) | MAE (clipped) | baseline MAE | skill vs baseline |
|---|---|---|---|---|
| 16,458 | 0.768 | 4.420 | 12.008 | 0.632 |

If your numbers match, the installation and data are correct. The unit tests check the same behaviour on small synthetic data:

```bash
python -m pytest -v
```

Expected: all tests pass (the one real-data test is skipped if `data/` is empty).

## 5. Documentation and further reading

- **Code documentation (HTML):** `docs/index.html` – generated from the docstrings (open the file in a browser after cloning).
- **Test report:** `reports/test_report.html` (and `reports/test_report.txt`).
- **Project report:** submitted separately via Moodle; it covers the engineering problem, validation, comparison of methods, limitations and the AI-use reflection.
- **Regenerate docs and test report:**
  - Windows: `make_docs_and_tests.bat`
  - Mac / Linux: `bash make_docs_and_tests.sh`
- Libraries used: [pandas](https://pandas.pydata.org/), [NumPy](https://numpy.org/), [SciPy](https://scipy.org/) (`CubicSpline`), [scikit-learn](https://scikit-learn.org/) (`LinearRegression`, metrics), [Matplotlib](https://matplotlib.org/).

## 6. Licence

Released under the **GPL-2.0 license** – see the [`LICENSE`](LICENSE) file. The sensor data belongs to its owners and is provided through the MMA3001 unit; it is not covered by this licence.

## 7. AI use

AI tools (Claude, Anthropic) assisted with code review, refactoring into the documented module, test writing and drafting this README. All code, results and claims were checked and are the author's responsibility. The full AI-use reflection is in the project report.

