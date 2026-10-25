# Setup and usage

## Requirements

- Python 3.9+ and Jupyter.
- Packages: `pandas`, `numpy`, `matplotlib`, `statsmodels`, `scikit-learn`, `tslearn`. The evaluation notebooks also import `boto3` for an optional S3 loader that you can ignore when loading from local files.

```bash
pip install pandas numpy matplotlib statsmodels scikit-learn tslearn jupyter
```

No build step or package is provided: the code lives in notebooks.

## Data

The full SIMPaCT sensor dataset is not distributed here. What is included:

| File | Content |
|---|---|
| `data/example_model_input.csv` | Sample of the model input schema: `datetime`, `bom_forecasted_precipitation_quantity`, `sm_soil_moisture`, `bom_forecasted_relative_humidity`, `bom_forecasted_relative_temperature` (hourly) |
| `data/sensor_locations/map_sensor_locations.html` | Map of the sensor sites; open in a browser |
| `results/rob_mae_all.csv` | Columns `sensor`, `date`, `Base`, `Linear`, `Circular`: forecast MAE per sensor and date |

To run on your own network, provide long-format sensor data with a datetime index, a sensor name column (`sm_station_name`) and a soil-moisture column (`sm_soil_moisture`), plus any forecast features the model uses.

**File names and paths.** The notebooks were written against local paths and file names that are not in this repository (for example `SOPA-data-TS-daily-clean-2023-04-and-2023-09.csv` and an S3 loader). Edit the `data_path`, `datapath` and `read_csv` lines at the top of each notebook to point at your files. The clustering notebooks resolve paths relative to the notebook's working directory.

## Running order

1. **`src/notebooks/clustering/TS DTW KMeans and KShape Clustering.ipynb`**: load and clean the raw series, write the cleaned daily and hourly files, then run a first K-shape and DTW K-means clustering.
2. **`src/notebooks/clustering/TS Clustering and Silhouette.ipynb`**: sweep k with both algorithms, save silhouette scores, then fit the final clusterings (k = 21 K-shape, k = 16 DTW K-means) and plot clusters.
3. **`src/notebooks/evaluation/Replace sensors.ipynb`**: replace a target sensor with the average of its cluster mates and compare the forecast MAE against the baseline.
4. **`src/notebooks/evaluation/Robotic evaluation.ipynb`**: emulate linear and circular robot paths by keeping only visit-interval readings, and write `rob_mae_all.csv`.

## Key parameters

| Parameter | Where | Meaning |
|---|---|---|
| `n_clusters_range` | silhouette notebook | Range of k to sweep |
| `n_clusters` | silhouette notebook | Final k per algorithm |
| sensor name list | evaluation notebooks | Target sensor and the cluster mates used to replace it |
| training interval, evaluation dates | `get_training_set`, `create_train_test_sets` | Hours of history used and the days predicted |
| visit interval | robotic notebook | Spacing of kept readings; the paper uses 120 minutes |

## Output

- Silhouette scores per k, as a CSV beside the data.
- Cluster plots per method.
- MAE comparisons for baseline, replaced and robotic variants, with `results/rob_mae_all.csv` holding the robotic runs.
