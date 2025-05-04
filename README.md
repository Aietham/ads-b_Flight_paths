# ADS‑B Flight Paths

## Interactive Map – screen‑recording

https://github.com/user-attachments/assets/931afaf5-8def-4e0c-bf32-9f43d52388b8


**ADS‑B Flight Path Explorer** is a Jupyter Notebook–based pipeline that lets you:

1. **Fetch** ADS‑B log files (as ZIPs) over SFTP by date  
2. **Parse & clean** raw JSON‐line payloads into a tidy DataFrame  
3. **Segment** each aircraft’s track by time‑gaps (to break flights into coherent legs)  
4. **Aggregate** segments and compute two metrics:
   - **Irregularity**: how much a segment deviates from a straight line  
   - **Total Turning**: sum of absolute bearing changes  
5. **Visualize** the results on an interactive ipyleaflet map with:
   - A **Date Picker** to choose the log date (re‑runs the full pipeline)  
   - An **Irregularity Slider** + **Apply Filters** button (re‑filters cached data)  
   - **Hover popups** showing Flight ID & Irregularity  
   - A **Download CSV** link exporting the currently visible segments   
---

## Required Libraries

All of these are installed via `pip install -r requirements.txt`:

- **pandas**  
- **pyyaml** (for `config.yml`)  
- **paramiko** (for SFTP connectivity)  
- **folium** (for static map exports)  
- **ipyleaflet** (for interactive in‑notebook maps)  
- **ipywidgets** (for the date‑picker, slider, buttons, etc.)  

---

## Configuration

Create a `config.yml` in the project root with your SFTP credentials and paths:

```yaml
site: 
  hostname: "files.clarksonmsda.org"
  port: 2022  # Default SSH port
  username: "<username>"
  password: "******"
  log_path: 'adsb/ny0'
  base_server_path: "/srv/sftpgo/data/users/<username>"
```
  
# Notebok Functions


## 1.  Configuration Loader

This module reads environment-specific settings from a YAML file (`config.yml`) and exposes them as Python constants.


### Dependencies

- Python 3.x  
- [Paramiko](https://www.paramiko.org/) (`pip install paramiko`)

### Functions

#### `connect_to_server(cfg: dict) -> SSHClient`

- **Purpose:**  
  Opens an SSH connection using credentials in the config.
- **Parameters:**  
  - `cfg` (`dict`): Configuration loaded via `read_config`, with:
    - `site.hostname`  
    - `site.port`  
    - `site.username`  
    - `site.password` (optional)
- **Returns:**  
  An active `paramiko.SSHClient` connected to the server.

#### `get_sftp_client(cfg: dict) -> (SSHClient, SFTPClient)`

- **Purpose:**  
  Builds upon `connect_to_server` to open an SFTP session over SSH.
- **Parameters:**  
  - `cfg` (`dict`): Same as above.
- **Returns:**  
  A tuple `(ssh_client, sftp_client)` for shell and file operations.


#### `build_remote_zip_path(log_date: date, base_path: str) -> str`

- **Purpose:**  
  Generate the filename and full path for the zipped ADS-B log corresponding  
  to `log_date`, using the pattern `adsblog_ny0.txt.YYYYMMDD00.zip`.

- **Parameters:**  
  - `log_date` (`date`): The date of the log file.  
  - `base_path` (`str`): Remote directory where ZIP logs are stored.

- **Returns:**  
  A string representing the joined path, e.g.:  /mnt/server/logs/adsblog_ny0.txt.2022100400.zip

  
## 2.  Data Loading & Cleaning

This module handles fetching, extracting, and cleaning ADS‑B log archives
over SFTP, producing a ready‑to‑use pandas DataFrame.

### Constants

- **`COLUMNS_TO_DROP`**  
  A list of payload fields to remove after flattening JSON into a table.

### Functions

#### `load_and_clean_adsb_zip_json(sftp, remote_zip_path) -> pd.DataFrame`

- **Purpose:**  
  1. Download the ZIP archive at `remote_zip_path` via the `sftp` client.  
  2. Extract the first text member (e.g., `*.txt`) from the archive.  
  3. Parse each line as JSON, filtering for `"type": "new_adsb"`.  
  4. Flatten the payload and attach the original timestamp under `date`.  
  5. Drop all fields listed in `COLUMNS_TO_DROP`.  
- **Parameters:**  
  - `sftp` (`paramiko.SFTPClient`): Active SFTP session.  
  - `remote_zip_path` (`str`): Full server path to the `.zip` file.  
- **Returns:**  
  A `pandas.DataFrame` with cleaned ADS‑B data.

#### `clean_adsb_data(df: pd.DataFrame) -> pd.DataFrame`

- **Purpose:**  
  1. Parse the `date` column into pandas `datetime64[ns]`.  
  2. Sort records by `hex` (aircraft identifier) and then by timestamp.  
  3. Reset the index to ensure a clean, contiguous ordering.

- **Parameters:**  
  - `df` (`pd.DataFrame`): DataFrame returned by `load_and_clean_adsb_zip_json`, containing at least `hex` and `date` columns.

- **Returns:**  
  A cleaned `pd.DataFrame` with properly typed dates, sorted rows, and a fresh integer index.


#### `split_by_time_gap(group: DataFrame, max_gap_minutes: int = 10) -> DataFrame`

- **Purpose:**  
  Breaks a continuous flight path into discrete segments when there’s a prolonged
  gap between ADS‑B messages (e.g., aircraft parked or signal lost).

- **Parameters:**  
  - `group` (`pd.DataFrame`): Rows for one flight, containing a `date` column.  
  - `max_gap_minutes` (`int`, default `10`): Gap threshold in minutes.

- **Returns:**  
  A DataFrame with the original columns plus:  
  - **`time_diff`** (`float`): Minutes since the previous record.  
  - **`segment`** (`int`): Segment ID, starting at 0 and incrementing each time  
    `time_diff > max_gap_minutes`.

#### `segment_adsb(df: pd.DataFrame, max_gap: int = 10) -> pd.DataFrame`

- **Purpose:**  
  Apply time‑gap segmentation across the entire ADS‑B dataset by aircraft and flight,  
  using `split_by_time_gap` under the hood.

- **Parameters:**  
  - `df` (`pd.DataFrame`): DataFrame returned by `clean_adsb_data`, containing:
    - `hex` (aircraft identifier)  
    - `flight` (flight ID)  
    - `date` (timestamp)  
  - `max_gap` (`int`, default `10`): Threshold in minutes to break segments.

- **Returns:**  
  A `pandas.DataFrame` including:
  - **`time_diff`** (`float`): Minutes since the previous record.  
  - **`segment`** (`int`): Segment index for each continuous block.  
  The index is reset to a simple range.

#### `aggregate_flight_segments(g: pd.DataFrame) -> dict`

- **Purpose:**  
  Create a high‑level summary for one flight segment, including:
  - Start and end timestamps  
  - Total duration in minutes  
  - The ordered path as latitude/longitude pairs  

- **Parameters:**  
  - `g` (`pd.DataFrame`): Records for a single segment, must include:
    - `date` (`datetime64[ns]`)  
    - `lat`, `lon` (`float`)  

- **Returns:**  
  A `dict` with:
  - **`start_time`** (`pd.Timestamp`): First timestamp.  
  - **`end_time`** (`pd.Timestamp`): Last timestamp.  
  - **`duration`** (`float`): Elapsed minutes between start and end.  
  - **`coordinates`** (`List[Tuple[float, float]]`): Chronological path points.

#### `aggregate_segments(df: pd.DataFrame) -> pd.DataFrame`

- **Purpose:**  
  Combine all records for each `(hex, flight, segment)` into a single row,  
  using the logic defined in `aggregate_flight_segments`.

- **Parameters:**  
  - `df` (`pd.DataFrame`): Segmented ADS‑B DataFrame with columns:  
    - `hex` (aircraft ID)  
    - `flight` (flight identifier)  
    - `segment` (integer segment index)  
    - plus any fields needed by `aggregate_flight_segments` (`date`, `lat`, `lon`, etc.)

- **Returns:**  
  A DataFrame where each row represents one flight segment, with columns:  
  - `hex`, `flight`, `segment`  
  - `start_time` (first timestamp)  
  - `end_time` (last timestamp)  
  - `duration` (minutes between start and end)  
  - `coordinates` (list of `(lat, lon)` tuples)

#### `haversine(lat1, lon1, lat2, lon2) -> float`

- **Purpose:**  
  Compute the great-circle distance (in kilometers) between two points on Earth  
  specified by their latitude and longitude in decimal degrees.

- **Parameters:**  
  - `lat1`, `lon1` (`float`): Coordinates of the first point.  
  - `lat2`, `lon2` (`float`): Coordinates of the second point.

- **Returns:**  
  - `float`: Distance in kilometers.

#### `compute_irregularity(coords: List[Tuple[float, float]]) -> float`

- **Purpose:**  
  Quantify how much a given path deviates from a straight line by comparing its actual 
  traveled length to the direct great‑circle distance between the start and end points.

- **Parameters:**  
  - `coords` (`List[Tuple[float, float]]`):  
    An ordered list of `(latitude, longitude)` pairs. Any pairs containing `None` are ignored.

- **Returns:**  
  - A `float` between 0 and 1:
    - `0.0` indicates a straight path (total distance equals direct distance).
    - Values closer to `1.0` indicate paths that are more circuitous.

- **Logic Breakdown:**  
  1. **Filter missing points:** Remove any entries where latitude or longitude is `None`.  
  2. **Total distance:** Sum haversine distances between each consecutive pair of valid points.  
  3. **Direct distance:** Compute haversine distance between the first and last valid points.  
  4. **Compute ratio:** Return `1 - (direct_distance / total_distance)`, guarding against zero total length.

#### `calculate_bearing(lat1, lon1, lat2, lon2) -> float`

- **Purpose:**  
  Determine the initial compass bearing (in degrees) when traveling from  
  a start point `(lat1, lon1)` to an end point `(lat2, lon2)` along a great circle.

- **Parameters:**  
  - `lat1`, `lon1` (`float`): Latitude and longitude of the start point in decimal degrees.  
  - `lat2`, `lon2` (`float`): Latitude and longitude of the end point in decimal degrees.

- **Returns:**  
  - `float`: Bearing in degrees clockwise from true north (range `0°` to `<360°`).

- **Algorithm:**  
  1. Convert all latitudes and longitudes to radians.  
  2. Compute `x = sin(Δλ) · cos(φ₂)`.  
  3. Compute `y = cos(φ₁) · sin(φ₂) – sin(φ₁) · cos(φ₂) · cos(Δλ)`.  
  4. Calculate `θ = atan2(x, y)` and convert to degrees.  
  5. Normalize to `[0, 360)` by adding 360° and taking modulo 360°.

#### `compute_total_turning(coords: List[Tuple[float, float]]) -> float`

- **Purpose:**  
  Calculate the cumulative turning angle along a flight path or trajectory by summing  
  the absolute changes in bearing between each successive segment.

- **Parameters:**  
  - `coords` (`List[Tuple[float, float]]`):  
    An ordered list of `(latitude, longitude)` pairs. Any pairs containing `None` are ignored.

- **Returns:**  
  - `float`:  
    Total turning in degrees.  
    - Returns `0.0` if fewer than three valid points exist (no turn to compute).  
    - Each turn is normalized so that angles greater than 180° are converted  
      to their supplement (e.g., a 270° change becomes 90°).

- **Implementation Details:**  
  1. **Filter missing points:** Remove entries where lat or lon is `None`.  
  2. **Bearing calculation:** Use `calculate_bearing` to get the direction of each leg.  
  3. **Angular difference:** For each adjacent pair of bearings, compute `abs(b2 - b1)`.  
  4. **Normalization:** If `diff > 180°`, replace with `360° - diff` for the shorter turn.  
  5. **Accumulation:** Sum all normalized differences to get the total turning.

> **Note:** We will use only the **irregularity** metric—a detour index comparing total path length to great‑circle distance—to detect flights that take unusual, meandering routes.\
> **Details:** The irregularity metric (1 − direct/total distance), also known as circuitousness or the detour index, quantifies path tortuosity by measuring how far a route deviates from a straight line.  
> **Anomaly Detection:** Flights with high irregularity are flagged as anomalous, since a high value indicates significant detours or deviations from the expected great‑circle trajectory.  
> **Total Turning:** Although the total turning metric—summing absolute changes in bearing between successive points—accurately characterizes path curvature, we find that the irregularity metric alone suffices for isolating unusual flight patterns.

## 3.  Plotting the data

### GeoJSON Builder

We convert our flight‑segment DataFrame into a GeoJSON FeatureCollection for map rendering.

> **GeoJSON Standard:**  
> GeoJSON is defined by RFC 7946 and supports geometries like LineString within Feature and FeatureCollection objects.  
> **Safe Parsing:**  
> We use `ast.literal_eval` to convert string‑encoded Python literals (like lists of tuples) into actual lists without executing arbitrary code.  
> **Visualization:**  
> The resulting GeoJSON can be fed directly into mapping libraries (e.g., `ipyleaflet.GeoJSON`) for interactive display.

### Function: 

#### `build_geojson(df)`

- **Purpose:** Generate a GeoJSON dictionary:
  1. Iterate rows of `df`.  
  2. Parse `coordinates` (string or list) into a list of valid `[lon, lat]` pairs.  
  3. Skip segments with fewer than 2 points.  
  4. Build a Feature with a LineString geometry and attach properties:
     - `flight`: identifier.
     - `irregularity`: detour index for anomaly detection.
     - `popup`: optional HTML for map popups.
  5. Return a `FeatureCollection` of all features.

### Widget & Control Definitions

We define the sidebar controls for date selection, irregularity threshold,
an apply button, and placeholders for status and loading indicators.

- **DatePicker** (`date_picker`): Calendar popup to pick log date.  
- **FloatSlider** (`irreg_slider`): Filter threshold on path irregularity.  
- **Button** (`apply_btn`): Triggers data fetch & map update.  
- **HTML** (`download_html`): Shows Download CSV link or messages.  
- **HTML** (`loading_html`): Full-screen loading overlay.  
- Controls are grouped in a `VBox` and placed via `WidgetControl` in the top-right.


### Map & GeoJSON Layer Initialization

Initialize an `ipyleaflet.Map` and an empty `GeoJSON` layer that will later
be populated with flight path LineStrings.


### Hover Popup Handlers

These functions attach to the `GeoJSON` layer’s `on_hover` and `on_mouseout`
events to render a floating `Popup` showing the flight ID and
irregularity metric.


### Data Pipeline Definition

Encapsulates the end‑to‑end processing for a given date:

1. **Load & unzip**: `load_and_clean_adsb_zip_json`  
2. **Clean**: `clean_adsb_data`  
3. **Segment**: `segment_adsb`  
4. **Aggregate**: `aggregate_segments`  
5. **Metrics**: `compute_metrics`


### Apply Filters Callback

This function is bound to the **Apply Filters** button:

- Reloads (downloads & processes) data only if the selected date changed—otherwise skips the heavy pipeline steps and uses cached DataFrames.  
- Updates the slider range to reflect the new data distribution after a date change.  
- When only the irregularity threshold changes, it does not rerun the full pipeline, it simply refilters the already‑loaded metrics and redraws the map.  
- Filters segments by the slider’s irregularity threshold.
- Updates the map’s GeoJSON layer and recenters the view.
- Generates a CSV download link for the filtered segments.
- Catches `FileNotFoundError` to handle missing logs.


### Wiring & Initial Display

Finally, we connect the **Apply Filters** button to `apply_all_filters`,
invoke it once to load the default view, and render the map.

## Future Scope and Improvements

- **Parallel SFTP Fetching:** Implement concurrent downloads of multiple ZIP files to reduce total pipeline runtime.  
- **Incremental Ingestion:** Track and process only new or updated log dates, avoiding full dataset reprocessing on each run.  
- **Database Integration:** Persist cleaned and aggregated segments in a time-series or spatial database (e.g., InfluxDB, TimescaleDB, PostGIS) for efficient querying and historical analysis.  
- **Real‑time Streaming:** Transition from batch ZIP ingestion to a streaming pipeline (e.g., Kafka, AWS Kinesis) for near‑real‑time flight path visualization.  
- **Advanced Anomaly Detection:** Integrate machine learning models (e.g., DBSCAN clustering, Isolation Forest) to automatically flag unusual flight behavior.  
- **Enhanced UI Controls:** Add options to select multiple dates, define custom geographic bounding boxes, and overlay traffic‑density heatmaps.  
- **Automated Deployment:** Containerize the pipeline with Docker and orchestrate scheduled runs via cloud services (e.g., AWS Lambda, Azure Functions, Airflow).
