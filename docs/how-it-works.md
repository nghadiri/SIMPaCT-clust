# How the SIMPaCT sensor optimisation works

This guide explains the methods as implemented in `src/notebooks`. For the original treatment, experiments and figures, read the [paper](../README.md#paper).

## Background in one minute

SIMPaCT irrigates a 42 ha park from forecasts of soil moisture. The sensor network behind those forecasts has 202 soil-moisture sensors, 50 ambient temperature and humidity sensors and 13 weather stations, and the models degrade when sensor data is missing. Two questions follow:

1. **If a moisture sensor fails, can its neighbours in behaviour stand in for it?** And, turned around, are some sensors redundant?
2. **Could a few robots that visit sensor sites in turn replace many fixed sensors?**

Both are answered by running the *same* downstream moisture forecast on altered input and comparing its **Mean Absolute Error (MAE)** with the baseline run on the original data.

## The two methods

### Method 1: cluster sensors, then substitute within clusters

**Input.** One time series per moisture sensor. The paper clusters hourly or daily readings, and concatenates April and September 2023 so that a cluster means "similar in a warm month and in a cold month".

**Two clustering algorithms**, both from `tslearn`:

| Algorithm | Distance | Strength for this data |
|---|---|---|
| **DTW-based K-means** | Dynamic Time Warping: the cheapest alignment that may stretch or compress time | Tolerates shifted peaks and uneven timing |
| **K-shape** | Shape-based distance: one minus the best normalised cross-correlation over all shifts | Ignores amplitude, so two sensors with the same pattern at different moisture levels group together |

**Choosing k.** The notebook sweeps k and records the silhouette score for each. The paper adds a second check, cluster occupancy: in its experiments the k that maximised K-shape's average silhouette left three of seven clusters empty and one cluster with only 8 members against 126 in another. The values used are **k = 21 for K-shape and k = 16 for DTW K-means**.

**Using the clusters.**

- *Replacement:* a missing sensor's series is replaced by the average of the other sensors in its cluster (`replace_sensor_values_avg` in `Replace sensors.ipynb`). Only members that are present in the evaluation data can contribute.
- *Anomaly detection:* when plotted, a sensor that follows its cluster for one month and then departs is a candidate fault, or a real local change worth investigating.
- *Redundancy:* sensors whose series closely duplicate their cluster mates are candidates for removal.

### Method 2: robotic sequential data collection

The `n_p` sensor sites are partitioned into `n_s` groups, and one robot per group visits its sites in turn every `t_s` minutes, so a full round takes `T_r = n_s * t_s`. Between visits a site has no new reading.

The notebooks **emulate** this on existing data instead of deploying robots: two groups of six sensor sites are chosen, one laid out like a straight line and one like a circle, and only the readings that fall on the visit interval (every 120 minutes) are kept. The forecast is then rerun on that thinned data. Per-sensor, per-date MAE for the baseline and both path shapes is saved to `results/rob_mae_all.csv`.

## The evaluation loop

For each target sensor and evaluation date, `Replace sensors.ipynb` and `Robotic evaluation.ipynb`:

1. build a training window of earlier hours for that sensor (`get_training_set`) and the next 24 hours to predict (`get_prediction_interval`);
2. fit a KNN-based forecaster, choosing its neighbour count by validation (`get_best_k`, `run_knn`);
3. repeat on the altered data (cluster-average replacement, or two-hourly robot readings);
4. compare the resulting MAE with the baseline MAE.

## What is innovative

1. **Sensor-level clustering for data reliability.** Related work mostly clusters IoT data to cut communication or computation. Here the aim is the quality of the model's input data.
2. **Two seasons in one clustering.** Concatenating a warm and a cold month demands consistent behaviour year-round before sensors are treated as interchangeable.
3. **Occupancy-aware choice of k.** Silhouette alone favours degenerate partitions; checking cluster sizes catches them.
4. **One analysis, three uses.** The same clusters support imputation, anomaly spotting and network pruning.
5. **Real data from an operating urban park.** Much comparable work uses synthetic or simulated data, or assumes uniform sensor layouts such as straight lines. The robotic emulation here uses irregular real sites.
6. **Evaluated on the production metric.** Each variant is judged by downstream forecast MAE, the metric SIMPaCT uses to choose models.

## What the results do and do not show

Read the paper's Table I for exact figures. In summary, as reported there:

- The abstract reports up to a 5.4% decrease in average error from cluster-based replacement, and decreases of 17.2% (circular path) and 2.1% (linear path) for robotic collection.
- Replacement is not uniformly an improvement: for one of the two target sensors, replacing it with cluster averages raised MAE, K-shape more than DTW. DTW-based replacement stayed closer to the baseline error with less spread in the paper's experiments.
- The robotic results are an emulation. Battery life, maintenance and weather are listed in the paper as practical limits, and are not modelled.

Treat replacement as a graceful fallback when a sensor is lost, not a free substitute for a working one.

## Mapping to the code

| Concept | Where |
|---|---|
| Load, clean and pivot sensor series | `clustering/TS DTW KMeans and KShape Clustering.ipynb` |
| `cluster_sensors`: pivot, fill gaps with the sensor mean, scale, cluster | `clustering/TS Clustering and Silhouette.ipynb` |
| Sweep of k and silhouette scores for both algorithms | `clustering/TS Clustering and Silhouette.ipynb` |
| Final clusterings (k = 21 K-shape, k = 16 DTW K-means) | `clustering/TS Clustering and Silhouette.ipynb` |
| Cluster-average replacement | `evaluation/Replace sensors.ipynb`: `replace_sensor_values`, `replace_sensor_values_avg` |
| Downstream KNN forecast and MAE | `evaluation/*.ipynb`: `get_training_set`, `create_train_test_sets`, `run_knn` |
| Robotic path emulation | `evaluation/Robotic evaluation.ipynb` |
| Stored robotic results | `results/rob_mae_all.csv` |
| Sensor site map | `data/sensor_locations/map_sensor_locations.html` |
