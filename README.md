# Geocoding Florida Healthcare Facilities with ArcGIS API for Python & Folium

**Author:** Rahman  
**Tools:** Python · ArcGIS API for Python · Folium · GeoPandas · Pandas

---
 ## Live Maps
- [🗺️ CartoDB Basemap](https://siddik95.github.io/GeocodingWithArcGISAPI/geocoded_map.html)
- [🛰️ Esri Satellite](https://siddik95.github.io/GeocodingWithArcGISAPI/geocoded_map_esri.html)
- [🎛️ Multi-Basemap (Interactive)](https://siddik95.github.io/GeocodingWithArcGISAPI/geocoded_map_multi.html)
## Overview

**Geocoding** is the process of converting a place name or street address into geographic coordinates (latitude and longitude) that can be plotted on a map. In this project, the ArcGIS API for Python is used to geocode the locations of **critical community and emergency healthcare facilities across Florida**, and the results are visualized as interactive web maps using the **Folium** library.

The workflow covers three stages:

1. **Single-point geocoding** — testing the API with one address
2. **Batch geocoding** — geocoding an entire CSV dataset of Florida facilities (5,794 locations)
3. **Interactive map visualization** — rendering the results with multiple basemap options

---

## Part 1 — Single Point Geocoding

Before processing the full dataset, the workflow is validated by geocoding a single well-known address: the **University of Central Florida**.

```python
from arcgis.gis import GIS
from arcgis.geocoding import geocode

# Connect to ArcGIS Online (anonymous)
gis = GIS()

address = "University of central florida"
results = geocode(address)

if results:
    best_match = results[0]
    location = best_match['location']
    print(f"Full Address: {best_match['attributes']['LongLabel']}")
    print(f"Longitude (X): {location['x']}")
    print(f"Latitude (Y):  {location['y']}")
    print(f"Score: {best_match['score']}/100")
```

The `geocode()` function returns a ranked list of candidate matches. The **first result** (`results[0]`) is the best match, and from it we extract:

| Field | Description |
|---|---|
| `LongLabel` | Full formatted address returned by the geocoder |
| `location['x']` | Longitude |
| `location['y']` | Latitude |
| `score` | Confidence score out of 100 |

---

## Part 2 — Loading the Dataset

The dataset is the **Critical Community and Emergency Facilities** CSV for Florida, loaded with Pandas.

```python
import pandas as pd

df = pd.read_csv("Critical_Community_and_Emergency_Facilities.csv")
df.head()
```

A quick data quality check using `df.info()` revealed **missing values in the `City` column**. Since City is not essential — the geocoder uses `Address`, `City`, and `Zip` together — these gaps do not significantly affect geocoding accuracy. The state abbreviation `FL` is appended explicitly to every query to further constrain results to Florida.

---

## Part 3 — Batch Geocoding All Facilities

The core geocoding loop iterates over every row in the DataFrame, constructs a full address string, calls the ArcGIS geocoder, and stores the resulting coordinates back into the DataFrame.

```python
latitudes, longitudes, long_labels = [], [], []

for index, row in df.iterrows():
    full_address = f"{row['Address']}, {row['City']}, FL {row['Zip']}"
    results = geocode(full_address)

    if results:
        best_match = results[0]
        longitudes.append(best_match['location']['x'])
        latitudes.append(best_match['location']['y'])
        long_labels.append(best_match['attributes']['LongLabel'])
    else:
        longitudes.append(None)
        latitudes.append(None)
        long_labels.append("Not Found")

df['Longitude'] = longitudes
df['Latitude']  = latitudes
df['Geocoded_LongLabel'] = long_labels
```

**Result:** All **5,794 facilities** were geocoded successfully — no failures. The enriched DataFrame was saved back to CSV:

```python
df.to_csv("Critical_Community_and_Emergency_Facilities_geocoded.csv", index=False)
```

---

## Part 4 — Visualizing the Results

All three maps are rendered with **Folium**, a Python library that wraps Leaflet.js to produce interactive HTML maps. Points are drawn as `CircleMarker` elements with a color gradient applied via a `LinearColormap`, and each marker displays the **facility name** on hover (tooltip) and click (popup).

A **Florida bounding box** is used to filter out any geocoding artifacts that may have landed outside the state:

```python
bbox = [-87.634938, 24.523096, -80.031362, 31.000888]
#        West lon    South lat   East lon    North lat
```

---

### Map 1 — CartoDB Positron Basemap

The first map uses the clean, minimal **CartoDB Positron** basemap, which provides a neutral light-gray background that allows the colored healthcare facility markers to stand out clearly.

```python
import folium
from folium import CircleMarker
from branca.colormap import LinearColormap

m = folium.Map(location=[28, -81], tiles='cartodbpositron', zoom_start=7, min_zoom=3)
colormap = LinearColormap(['red', 'orange', 'yellow', 'green', 'blue', 'purple'])

for idx, row in df.iterrows():
    if bbox[0] <= row['Longitude'] <= bbox[2] and bbox[1] <= row['Latitude'] <= bbox[3]:
        CircleMarker(
            [row['Latitude'], row['Longitude']],
            popup=row['Name'],
            fill=True, fill_opacity=0.5,
            color=colormap(idx),
            tooltip=row['Name'],
            radius=3
        ).add_to(m)

m.save("geocoded_map.html")
```

**Output:** `geocoded_map.html`
- [🗺️ CartoDB Basemap](https://siddik95.github.io/GeocodingWithArcGISAPI/geocoded_map.html)

> 🗺️ *5,794 healthcare facility points plotted across Florida on a light CartoDB basemap. Each dot is colored along a red → purple gradient. Hovering over a point shows the facility name; clicking opens a popup.*

---

### Map 2 — Esri World Imagery Basemap

The second map replaces the default tile layer with **Esri's satellite imagery**, giving geographic context by showing real terrain, roads, and urban areas beneath the facility markers.

```python
m = folium.Map(location=[28, -81], tiles='cartodbpositron', zoom_start=7, min_zoom=3)

for idx, row in df.iterrows():
    if bbox[0] <= row['Longitude'] <= bbox[2] and bbox[1] <= row['Latitude'] <= bbox[3]:
        CircleMarker(...).add_to(m)

# Overlay Esri satellite imagery
folium.TileLayer(
    tiles='https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
    attr='Esri',
    name='Esri Imagery',
    overlay=False,
    control=True
).add_to(m)

m.save("geocoded_map_esri.html")
```

**Output:** `geocoded_map_esri.html`
- [🛰️ Esri Satellite](https://siddik95.github.io/GeocodingWithArcGISAPI/geocoded_map_esri.html)
> 🛰️ *Same 5,794 facility points on an Esri satellite imagery basemap. The aerial view makes it easy to visually correlate healthcare facility density with urban vs. rural areas of Florida.*

---

### Map 3 — Multiple Switchable Basemaps

The final, most feature-rich map adds a **Folium Layer Control** widget that lets users switch between four different basemaps interactively — no code changes required:

| Basemap | Style |
|---|---|
| CartoDB Positron | Light, minimal (default) |
| Esri Satellite | Aerial/satellite imagery |
| OpenStreetMap | Classic street map |
| CartoDB Dark Matter | Dark mode — makes colored points pop |

```python
m = folium.Map(location=[28, -81], tiles='cartodbpositron', zoom_start=7, min_zoom=3)
colormap = LinearColormap(['red', 'orange', 'yellow', 'green', 'blue', 'purple'])

# Add data points
for idx, row in df.iterrows():
    if bbox[0] <= row['Longitude'] <= bbox[2] and bbox[1] <= row['Latitude'] <= bbox[3]:
        CircleMarker(
            location=[row['Latitude'], row['Longitude']],
            popup=row['Name'], fill=True, fill_opacity=0.7,
            color=colormap(idx), tooltip=row['Name'], radius=4
        ).add_to(m)

# Add extra basemap layers
folium.TileLayer(
    tiles='https://server.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer/tile/{z}/{y}/{x}',
    attr='Esri', name='Esri Satellite'
).add_to(m)

folium.TileLayer(tiles='openstreetmap', name='Open Street Map').add_to(m)
folium.TileLayer(tiles='cartodbdark_matter', name='Dark Mode').add_to(m)

# Layer switcher widget — must be added last
folium.LayerControl().add_to(m)

m.save("geocoded_map_multi.html")
```

**Output:** `geocoded_map_multi.html`
- [🎛️ Multi-Basemap (Interactive)](https://siddik95.github.io/GeocodingWithArcGISAPI/geocoded_map_multi.html)
> 🎛️ *The layer control panel (top-right corner of the map) lets users toggle between CartoDB Positron, Esri Satellite, OpenStreetMap, and Dark Mode. Markers are slightly larger (radius=4) and more opaque (fill_opacity=0.7) in this version for better visibility across all basemap styles.*

---



## Dependencies

```bash
pip install arcgis folium geopandas pandas branca
```

| Library | Purpose |
|---|---|
| `arcgis` | ArcGIS API for Python — provides the `geocode()` function |
| `folium` | Interactive Leaflet.js maps from Python |
| `geopandas` | Geospatial data handling (imported but available for extensions) |
| `pandas` | Reading/writing CSV, DataFrame operations |
| `branca` | Color maps (`LinearColormap`) for Folium |

---

## Summary

| Step | Input | Output |
|---|---|---|
| Single geocode | One address string | Lat/lon + confidence score |
| Batch geocode | CSV with addresses (5,794 rows) | CSV enriched with Latitude, Longitude, LongLabel |
| Map 1 | Geocoded CSV | `geocoded_map.html` — CartoDB light basemap |
| Map 2 | Geocoded CSV | `geocoded_map_esri.html` — Esri satellite basemap |
| Map 3 | Geocoded CSV | `geocoded_map_multi.html` — 4 switchable basemaps |

All 5,794 critical community and emergency healthcare facilities in Florida were successfully geocoded and visualized as interactive web maps.
