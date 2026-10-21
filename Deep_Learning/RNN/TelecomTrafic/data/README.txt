Network Traffic Forecasting - Dataset Documentation
=================================================

1) What this dataset is
-----------------------
This project uses a synthetic univariate telecom time series dataset that represents
network data consumption over time.

Main dataset file:
- data/sim_1year.csv

Columns:
- timestamp: hourly datetime index
- data_consumption_gb: simulated traffic volume in GB

Metadata file:
- data/sim_1year_meta.json

For the current generated file (sim_1year):
- Time range: 2023-01-01 00:00:00 to 2023-12-31 23:00:00
- Frequency: hourly
- Rows: 8,760
- Min: 19.719 GB
- Max: 403.99 GB
- Mean: 152.6547 GB
- Injected events (spikes): 31
- Injected drops (outages): 20


2) Why we created synthetic data
--------------------------------
Real telecom traffic data is usually private, restricted, or expensive to obtain.
To build and evaluate a realistic forecasting pipeline, we generated controlled
synthetic traffic with known behavior patterns that mimic production-like conditions.

This allows:
- reproducible experiments
- safe model development
- stress-testing forecasting methods under spikes and outages


3) How the dataset was created (detailed)
-----------------------------------------
Generator script:
- data/simulate_telecom_consumption.py

Core idea:
The final signal is built by combining trend + seasonality + structured noise,
then injecting spikes and drops.

Detailed generation steps:

Step A - Build timestamp index
- The script creates a DatetimeIndex from start_date and either:
  - months/years duration, or
  - fixed number of periods.
- For sim_1year, config uses:
  - years = 1
  - months = 0
  - freq = h
  - start_date = 2023-01-01

Step B - Create long-term trend
- Linear trend component: trend_per_step * t
- Compounded growth component:
  base_consumption * ((1 + monthly_growth_rate)^(t / steps_per_month) - 1)
- This emulates gradual traffic increase over time.

Step C - Add seasonality patterns
- Daily seasonality (24-hour cycle) using sine wave
- Weekly seasonality (7-day cycle) using sine wave
- Monthly seasonality (12-month cycle) using sine wave
- Evening peak boost (Gaussian-shaped) around ~21:00
- Weekend multiplier reduces traffic level on weekends

Step D - Add realistic residual noise
- Heteroskedastic white noise: noise level varies by hour/day
  (peak hours are noisier)
- AR(1) persistence is added:
  ar_noise[i] = ar1_strength * ar_noise[i-1] + eps[i]
- This creates temporal correlation commonly seen in network usage.

Step E - Inject sudden positive events (spikes)
- At each step, a spike may occur with spike_probability
- If triggered:
  - random duration between spike_min_duration and spike_max_duration
  - random magnitude between spike_min_magnitude and spike_max_magnitude
  - smooth burst profile is applied (rise + decay)
- Event clustering can increase the chance of nearby related spikes
  (event_cluster_scale), mimicking periods like sports finals or launches.

Step F - Inject network drops/outages
- At each step, a drop may occur with drop_probability
- If triggered:
  - random duration between drop_min_duration and drop_max_duration
  - random depth between drop_min_depth and drop_max_depth
- After each drop, a recovery tail is applied over a random recovery window,
  creating a gradual return to normal traffic.

Step G - Post-processing
- Values are clipped at 0 (no negative consumption)
- Values are rounded to 3 decimals
- Output CSV and JSON metadata are written to disk


4) Configuration used for data/sim_1year.csv
--------------------------------------------
The following key parameters were used (from sim_1year_meta.json):

- base_consumption: 120.0
- trend_per_step: 0.006
- monthly_growth_rate: 0.012
- noise_std: 5.0
- ar1_strength: 0.62

- daily_amplitude: 25.0
- weekly_amplitude: 12.0
- monthly_amplitude: 6.0
- weekend_factor: 0.86
- evening_peak_boost: 18.0

- spike_probability: 0.003
- spike_min_magnitude: 70.0
- spike_max_magnitude: 220.0
- spike_min_duration: 1
- spike_max_duration: 12
- event_cluster_scale: 0.25

- drop_probability: 0.0018
- drop_min_depth: 0.35
- drop_max_depth: 0.85
- drop_min_duration: 1
- drop_max_duration: 10
- recovery_min_duration: 2
- recovery_max_duration: 10

- seed: 42 (for reproducibility)


5) How to regenerate the dataset
--------------------------------
From the project root:

python data/simulate_telecom_consumption.py --years 1 --months 0 --freq h --start-date 2023-01-01 --output-csv data/sim_1year.csv --output-metadata data/sim_1year_meta.json


6) Notes for modeling
---------------------
- This is a univariate series (time -> data_consumption_gb).
- Because spikes/drops are intentionally included, stationarity transforms
  and robust validation are important for forecasting quality.
- Synthetic data is useful for prototyping, but production deployment should
  eventually be validated on real operator traffic.

