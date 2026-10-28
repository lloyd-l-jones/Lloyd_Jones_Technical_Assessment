# Lloyd Jones - Colibri Digital - Technical Assessment

This project loads one month of hourly readings from 15 wind turbines, cleans the data, summarises the daily power output of each turbine and flags any turbine whose daily average is unusual compared with the rest of the wind farm. The full pipeline is in a single Jupyter notebook.

## Project structure

```
.
├── Lloyd_Jones_Technical_Assessment.ipynb   # the full pipeline
├── data/                                    # input CSV files (data_group_1.csv to data_group_3.csv)
├── output/                                  # created by the notebook
│   ├── turbine_readings_clean.csv           # cleaned hourly readings with out-of-range flags
│   ├── daily_turbine_summary.csv            # daily min, max and average per turbine, with anomaly flags
│   └── cleaning_log.csv                     # each cleaning check, its count and the action taken
└── README.md
```

## How to run

1. Install Python 3 (I used Python 3.14.10) and the required packages:
   ```
   pip install pandas numpy matplotlib seaborn jupyter
   ```
   The versions of these packages I used were:
   ```
   pandas==3.1.0
   numpy==2.5.4
   matplotlib==3.11.2
   seaborn==0.13.2
   ```
2. Place the three CSV files in a folder called `data` next to the notebook. Alternatively, place the data ZIP file (named `data*.zip`) next to the notebook and it will be extracted automatically.
3. Open `Lloyd_Jones_Technical_Assessment.ipynb` and run all cells in order.
4. The outputs are saved to the `output` folder.

The notebook follows the five steps in the brief (sections 1 to 5), with some additional observations in section 6. All of the settings and parameters are defined in section 0.2, so any change only needs to be made in one place.

## Problems found in the data and how each was handled

The brief describes readings that are missing, duplicated or physically impossible. I wrote a check for each of these, but none of them were present in the supplied files. All 11,160 rows (15 turbines × 744 hours) are complete, unique and within plausible ranges. Each check still runs and records its count in the cleaning log, even when the count is zero, so the pipeline would handle these problems if they appear in future files.

| Step | Problem | What I did | Why |
|---|---|---|---|
| 2.1 | Values that cannot be converted to the correct type, for example text in a numeric column | Set to missing | The later steps then decide how to handle them |
| 2.2 | Missing timestamp or turbine ID | Removed the row | The reading cannot be placed in time or assigned to a turbine |
| 2.2 | Turbine ID not between 1 and 15 | Removed the row | The reading cannot be assigned to a known turbine |
| 2.2 | Timestamp outside March 2022 | Removed the row | The reading does not belong to this month's data |
| 2.3 | Exact duplicate rows | Kept the first copy | The copies contain the same information |
| 2.3 | More than one row for the same turbine and hour but with different values | Removed all copies | There is no way to tell which reading is correct |
| 2.4 | Wind speed, wind direction or power output outside its valid range | Set the value to missing and flagged it | The other valid measurements in the row are kept |
| 2.5 | Rows with no measurements at all | Removed the row | The row contains no information |
| 2.5 | Other missing measurements | Left as missing | Filling them would invent readings, and pandas skips missing values in the daily min, max and average |

### Observation about the data

Although every individual value is plausible, the power output does not follow the wind speed. For a real turbine, power output increases with wind speed until it reaches its rated power. In this data, the correlation between the two is -0.002 across all turbines, and every power level appears at every wind speed (see section 6.2). This suggests the data has been randomly generated. I did not treat this as a cleaning problem, as no single reading is invalid, but it does affect how the anomaly flags should be interpreted.

## Flagged turbines

For each day, I calculated the mean and standard deviation of the 15 turbine averages and flagged any turbine whose average was more than 2 standard deviations above or below the mean. In total, 17 of the 465 turbine-days were flagged:

| Turbine | Days flagged |
|---|---|
| 1 | 7 March (high) |
| 2 | 2 March (low) |
| 3 | 6 March (high), 28 March (low) |
| 4 | 11, 20 and 26 March (low) |
| 7 | 13 March (low) |
| 8 | 4 March (low) |
| 9 | 15 March (high), 29 March (low) |
| 11 | 8 and 22 March (low) |
| 12 | 3 and 12 March (low) |
| 15 | 1 and 18 March (high) |

Turbines 5, 6, 10, 13 and 14 were not flagged on any day. The full results, including the z-score for each turbine's day, are in `output/daily_turbine_summary.csv`.

I believe these flags are most likely random variation rather than any faulty turbines. If the daily averages were normally distributed, a threshold of 2 standard deviations would flag about 21 of the 465 turbine-days by chance alone and 17 is close to this. No turbine is flagged on two days in a row, and as noted above the power output does not follow the wind speed. Turbine 4 was flagged most often (three low days) which would be worth monitoring.

## Assumptions

- The rated capacity of each turbine is 5 MW, as this is not given in the brief (the highest value in the data is 4.5 MW). Power output above 5 MW is treated as physically impossible.
- Wind speed must be between 0 and 30 m/s. The upper limit is my own assumption for an hourly average because after some reading it seems that past 30 m/s would be classified as a violent storm where widespread damage is likely to occur. I believe this kind of storm would not be a common occurence. (source: https://www.rmets.org/metmatters/beaufort-wind-scale)
- Negative power output is invalid. In practice a turbine can draw a small amount of power when idle so this is a simplification.
- Wind direction must be between 0 and 360 degrees.
- The data covers March 2022, with one reading per turbine per hour. The period is defined in the settings rather than taken from the data.
- All timestamps are in the same time zone.
- The anomaly check uses the sample standard deviation (the pandas default). The population standard deviation would flag 20 turbine-days instead of 17. To change this replace `daily_groups.std()` with `daily_groups.std(ddof=0)` in section 4.2.

## What I would do next with more time

- Test each cleaning step against a copy of the data with deliberately added faults, ideally as automated tests.
- Add a check for missing hours. At the moment, gaps would only show up indirectly through the `n_readings` column in the daily summary.
- Save the outputs as Parquet instead of CSV, so that column types such as dates are preserved.
- Make the anomaly detection more robust, for example by comparing each turbine with the other 14 turbines only, or by only raising an alert when a turbine is flagged on several days in a row.
- Compare each turbine's power output against an expected power curve for the wind speed which would be a better way to identify underperforming turbines with real data.
