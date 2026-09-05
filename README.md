# map-bio-fr

A small GIS web application displaying agricultural parcels certified as organic in metropolitan France.

The purpose of this project is mainly educational: it demonstrates how several GIS technologies can be combined to build a web mapping application from a large geospatial dataset.

## Architecture

```text
GeoPackage (.gpkg)
        │
        │ GDAL / ogr2ogr
        ▼
PostgreSQL + PostGIS
        │
        │ Martin
        ▼
MVT / Vector Tiles
        │
        │ Leaflet + Leaflet.VectorGrid
        ▼
Web browser
```

### Technologies

- **GDAL / ogr2ogr** — import and inspect geospatial data
- **PostgreSQL** — relational database
- **PostGIS** — spatial database extension for PostgreSQL
- **DBeaver** — graphical database client
- **Martin** — vector tile server
- **MapLibre Martin** — serves PostGIS data as Mapbox Vector Tiles (MVT)
- **Leaflet** — web mapping library
- **Leaflet.VectorGrid** — displays vector tiles in Leaflet
- **Docker / Docker Compose** — runs PostgreSQL/PostGIS and Martin

---

# 1. Prerequisites

You need:

- macOS, Linux or Windows
- Docker
- Docker Compose
- Homebrew on macOS
- GDAL
- a web browser

DBeaver is optional but recommended for inspecting the PostgreSQL/PostGIS database.

---

# 2. Create the project

Create the project directory:

```bash
mkdir map-bio-fr
cd map-bio-fr
```

Create the frontend directory:

```bash
mkdir frontend
```

The project will eventually look approximately like this:

```text
map-bio-fr/
├── docker-compose.yml
├── README.md
├── cartobio-parcelles-2025-francemet-2154.gpkg
└── frontend/
    └── index.htm
```

The GeoPackage is not included in the Git repository.

---

# 3. PostgreSQL + PostGIS + Martin

Create `docker-compose.yml`:

```yaml
services:
  postgis:
    image: postgis/postgis:16-3.5
    platform: linux/amd64
    container_name: postgis_container
    restart: always
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgis_data:/var/lib/postgresql/data

  martin:
    image: ghcr.io/maplibre/martin:latest
    container_name: martin
    restart: always
    environment:
      DATABASE_URL: postgres://postgres:mypassword@postgis:5432/postgres
    ports:
      - "3000:3000"
    depends_on:
      - postgis

volumes:
  postgis_data:
```

The `platform: linux/amd64` setting allows the PostgreSQL/PostGIS container to run on Apple Silicon Macs through Docker's architecture emulation.

Martin is already part of the Docker Compose stack from the beginning. Its role will become useful once the PostGIS table has been populated.

---

# 4. Start the containers

Start PostgreSQL/PostGIS and Martin:

```bash
docker compose up -d
```

Check that both containers are running:

```bash
docker ps
```

You should see:

```text
postgis_container
martin
```

The services are exposed on:

```text
PostgreSQL/PostGIS → localhost:5432
Martin             → localhost:3000
```

---

# 5. Verify PostGIS

Check the installed PostGIS version:

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT PostGIS_Version();"
```

You can also check that the PostGIS extension is installed:

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT extname, extversion FROM pg_extension WHERE extname = 'postgis';"
```

---

# 6. Connect to PostgreSQL with DBeaver

DBeaver can be downloaded from the official website.

[DBeaver official download page](https://dbeaver.io/download/?utm_source=chatgpt.com)

Create a new PostgreSQL connection with:

```text
Host:       localhost
Port:       5432
Database:   postgres
Username:   postgres
Password:   mypassword
```

DBeaver is useful for visually inspecting:

- schemas
- tables
- columns
- geometry columns
- indexes
- spatial data

---

# 7. Download the CartoBio dataset

The project uses the 2025 metropolitan-France CartoBio dataset.

The dataset is provided by the French government through data.gouv.fr.

[CartoBio dataset on data.gouv.fr](https://www.data.gouv.fr/datasets/parcelles-certifiees-en-agriculture-biologique-sur-cartobio?utm_source=chatgpt.com)

The specific resource used by this project is:

[2025 metropolitan-France GeoPackage resource](https://www.data.gouv.fr/api/1/datasets/r/2eb61d76-ebb1-49a9-aaaa-ccd4bb141f93?utm_source=chatgpt.com)

The resulting file is:

```text
cartobio-parcelles-2025-francemet-2154.gpkg
```

The file contains the following layer:

```text
cartobio_parcelles_2025_francemet_2154
```

The dataset uses **EPSG:2154 (Lambert-93)**.

> The large GeoPackage is intentionally not committed to this repository.

---

# 8. Install GDAL

On macOS:

```bash
brew install gdal
```

Verify the installation:

```bash
ogrinfo --version
```

---

# 9. Inspect the GeoPackage

Inspect the layer:

```bash
ogrinfo -so \
  cartobio-parcelles-2025-francemet-2154.gpkg \
  cartobio_parcelles_2025_francemet_2154
```

The dataset contains approximately:

```text
1,232,528 features
```

The geometry type is:

```text
Polygon
```

and the CRS is:

```text
EPSG:2154
```

---

# 10. Count the features with GDAL

GDAL can use SQLite to query the GeoPackage directly.

```bash
ogrinfo -dialect SQLite \
  -sql "SELECT COUNT(*) FROM cartobio_parcelles_2025_francemet_2154" \
  cartobio-parcelles-2025-francemet-2154.gpkg
```

This should return approximately:

```text
1232528
```

This is a useful way to verify the source dataset before importing it into PostgreSQL.

---

# 11. Create the PostGIS schema

Create a dedicated `bio` schema:

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "CREATE SCHEMA IF NOT EXISTS bio;"
```

The final table will be:

```text
bio.parcelles
```

with the geometry column:

```text
geom
```

---

# 12. Test the import with 100 features

Before importing more than one million polygons, perform a small test import.

```bash
ogr2ogr \
  -f PostgreSQL \
  "PG:host=localhost port=5432 dbname=postgres user=postgres password=mypassword" \
  "cartobio-parcelles-2025-francemet-2154.gpkg" \
  "cartobio_parcelles_2025_francemet_2154" \
  -nln bio.test_parcelles \
  -lco GEOMETRY_NAME=geom \
  -lco SPATIAL_INDEX=GIST \
  --config PG_USE_COPY YES \
  -gt 200000 \
  -limit 100 \
  -progress
```

### Important

`-limit 100` is what limits the import to 100 features.

By contrast:

```text
-gt 200000
```

does **not** limit the number of imported features.

It controls the transaction group size used during the import.

---

# 13. Verify the test table

Count the imported features:

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT count(*) FROM bio.test_parcelles;"
```

Expected result:

```text
100
```

Check the geometry type:

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT GeometryType(geom), COUNT(*) FROM bio.test_parcelles GROUP BY GeometryType(geom);"
```

Check the SRID:

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT ST_SRID(geom), COUNT(*) FROM bio.test_parcelles GROUP BY ST_SRID(geom);"
```

The expected geometry is:

```text
ST_Polygon
```

with SRID:

```text
2154
```

---

# 14. Remove the test table

Once the test import has been successfully validated:

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "DROP TABLE bio.test_parcelles;"
```

---

# 15. Import the complete dataset

Now import all approximately 1.23 million parcels.

```bash
ogr2ogr \
  -f PostgreSQL \
  "PG:host=localhost port=5432 dbname=postgres user=postgres password=mypassword" \
  "cartobio-parcelles-2025-francemet-2154.gpkg" \
  "cartobio_parcelles_2025_francemet_2154" \
  -nln bio.parcelles \
  -lco GEOMETRY_NAME=geom \
  -lco SPATIAL_INDEX=GIST \
  --config PG_USE_COPY YES \
  -gt 200000 \
  -progress
```

Notice that there is **no**:

```text
-limit 100
```

This time the entire layer is imported.

Also, no custom `FID` is specified.

---

# 16. Verify the final import

Count the rows:

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT count(*) FROM bio.parcelles;"
```

Expected result:

```text
1232528
```

The exact number may vary if the source dataset is updated.

---

# 17. Inspect the table structure

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "\d bio.parcelles"
```

The table should contain the original attributes and a geometry column named:

```text
geom
```

---

# 18. Check the indexes

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT indexname FROM pg_indexes WHERE tablename = 'parcelles';"
```

A spatial GiST index should be present.

The spatial index is important because PostgreSQL/PostGIS needs to efficiently locate geometries spatially.

---

# 19. Martin

Martin is a vector tile server designed to serve geospatial data, including PostGIS data, as vector tiles.

In this project, Martin sits between PostgreSQL/PostGIS and the web browser:

```text
PostGIS
   │
   ▼
 Martin
   │
   ▼
MVT / Vector Tiles
   │
   ▼
Leaflet
```

Martin automatically discovers suitable PostGIS tables.

The PostGIS table:

```text
bio.parcelles
```

therefore becomes available through Martin.

---

# 20. Check the Martin catalog

Open:

```text
http://localhost:3000/catalog
```

The catalog should contain an entry corresponding to:

```text
parcelles
```

The important part is:

```text
bio.parcelles.geom
```

This confirms that Martin has discovered the PostGIS table and its geometry column.

---

# 21. Vector tile endpoint

Martin exposes the `parcelles` layer as vector tiles.

The template URL is:

```text
http://localhost:3000/parcelles/{z}/{x}/{y}
```

where:

- `z` = zoom level
- `x` = tile column
- `y` = tile row

For example, a concrete tile request could look like:

```text
http://localhost:3000/parcelles/6/32/22
```

The response is an MVT / Protocol Buffer binary payload.

The `{z}/{x}/{y}` placeholders are normally replaced by the mapping library rather than typed literally into the browser.

---

# 22. Create the frontend

Create:

```text
frontend/index.htm
```

The frontend uses:

- Leaflet
- Leaflet.VectorGrid
- OpenStreetMap
- Martin

---

# 23. Leaflet

Include Leaflet from the CDN:

```html
<link
  rel="stylesheet"
  href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
>

<script
  src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js">
</script>

<script
  src="https://unpkg.com/leaflet.vectorgrid@1.3.0/dist/Leaflet.VectorGrid.bundled.min.js">
</script>
```

No `integrity` or `crossorigin` attributes are required here.

---

# 24. Initialize the map

```javascript
const map = L.map('map').setView([46.5, 2.5], 6);

L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
  attribution: '&copy; OpenStreetMap contributors'
}).addTo(map);
```

The initial map is centered approximately on metropolitan France.

---

# 25. Display the Martin vector tiles

Leaflet.VectorGrid can consume Martin's MVT endpoint:

```javascript
const parcelles = L.vectorGrid.protobuf(
  'http://localhost:3000/parcelles/{z}/{x}/{y}',
  {
    vectorTileLayerStyles: {
      parcelles: {
        fill: true,
        fillOpacity: 0.15,
        weight: 0.5
      }
    },
    minZoom: 8,
    interactive: true
  }
).addTo(map);
```

### Why `minZoom: 8`?

The dataset contains more than one million polygons.

Displaying every parcel at national scale would produce an extremely dense visualization.

The layer therefore only becomes visible from zoom level 8:

```javascript
minZoom: 8
```

The reduced opacity also makes the visualization easier to read:

```javascript
fillOpacity: 0.15
```

---

# 26. Display parcel attributes

VectorGrid makes the feature properties available when a feature is clicked.

```javascript
parcelles.on('click', (e) => {

  const properties = e.layer.properties;

  let contenu = '<strong>Parcelle</strong><br><br>';

  for (const [cle, valeur] of Object.entries(properties)) {
    contenu += `<strong>${cle}</strong> : ${valeur}<br>`;
  }

  L.popup()
    .setLatLng(e.latlng)
    .setContent(contenu)
    .openOn(map);

});
```

The callback uses an anonymous arrow function:

```javascript
(e) => { ... }
```

This is a standard JavaScript callback passed to:

```javascript
parcelles.on('click', ...)
```

The popup dynamically displays the attributes contained in the selected vector-tile feature.

---

# 27. Complete frontend example

A minimal `frontend/index.htm` can therefore look like this:

```html
<!DOCTYPE html>
<html lang="en">

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">

  <title>Organic Farming Parcels — France</title>

  <link
    rel="stylesheet"
    href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
  >

  <style>
    html, body {
      height: 100%;
      margin: 0;
    }

    #map {
      height: 100%;
    }
  </style>
</head>

<body>

  <div id="map"></div>

  <script
    src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js">
  </script>

  <script
    src="https://unpkg.com/leaflet.vectorgrid@1.3.0/dist/Leaflet.VectorGrid.bundled.min.js">
  </script>

  <script>

    const map = L.map('map').setView([46.5, 2.5], 6);

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; OpenStreetMap contributors'
    }).addTo(map);


    const parcelles = L.vectorGrid.protobuf(
      'http://localhost:3000/parcelles/{z}/{x}/{y}',
      {
        vectorTileLayerStyles: {
          parcelles: {
            fill: true,
            fillOpacity: 0.15,
            weight: 0.5
          }
        },
        minZoom: 8,
        interactive: true
      }
    ).addTo(map);


    parcelles.on('click', (e) => {

      const properties = e.layer.properties;

      let contenu = '<strong>Parcelle</strong><br><br>';

      for (const [cle, valeur] of Object.entries(properties)) {
        contenu += `<strong>${cle}</strong> : ${valeur}<br>`;
      }

      L.popup()
        .setLatLng(e.latlng)
        .setContent(contenu)
        .openOn(map);

    });

  </script>

</body>

</html>
```

---

# 28. Run the frontend

From the project directory:

```bash
cd frontend
npx serve -l 5500
```

The frontend is then available at:

```text
http://localhost:5500
```

The port is deliberately different from Martin:

```text
Frontend → 5500
Martin   → 3000
PostGIS  → 5432
```

---

# 29. Final architecture

Once everything is running, the complete application looks like this:

```text
                    ┌─────────────────────────┐
                    │       GeoPackage        │
                    │                         │
                    │  1.23M organic parcels  │
                    └────────────┬────────────┘
                                 │
                                 │ GDAL / ogr2ogr
                                 ▼
                    ┌─────────────────────────┐
                    │ PostgreSQL + PostGIS    │
                    │                         │
                    │      bio.parcelles      │
                    │          geom           │
                    │       GiST index        │
                    └────────────┬────────────┘
                                 │
                                 │ SQL / PostGIS
                                 ▼
                    ┌─────────────────────────┐
                    │         Martin          │
                    │                         │
                    │     Vector tiles       │
                    │         (MVT)           │
                    └────────────┬────────────┘
                                 │
                                 │ HTTP
                                 ▼
                    ┌─────────────────────────┐
                    │         Leaflet         │
                    │   + Leaflet.VectorGrid  │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │       Web browser       │
                    └─────────────────────────┘
```

---

# 30. Role of each component

### GeoPackage

The original geospatial dataset.

```text
.gpkg
```

It contains the agricultural parcel geometries and their attributes.

### GDAL / ogr2ogr

The bridge between the GeoPackage and PostgreSQL.

It is used to:

- inspect the dataset
- count features
- import the data
- create the PostGIS table
- create the spatial index

### PostgreSQL

The relational database storing the data.

### PostGIS

The spatial extension that adds:

- geometry types
- spatial reference systems
- spatial functions
- spatial indexes
- spatial queries

### Martin

The vector tile server.

It reads the PostGIS data and exposes it as MVT tiles.

### Leaflet

The browser-side mapping library.

It handles:

- the map
- the basemap
- user interaction
- popups

### Leaflet.VectorGrid

The Leaflet plugin responsible for displaying vector tiles.

### Docker

Runs the database and tile server in isolated containers.

---

# 31. Ports

| Service | Port | Purpose |
|---|---:|---|
| PostgreSQL/PostGIS | `5432` | Database |
| Martin | `3000` | Vector tile server |
| Frontend | `5500` | Web application |

---

# 32. Summary

This project demonstrates a complete GIS web pipeline:

```text
GeoPackage
    ↓
GDAL
    ↓
PostGIS
    ↓
Martin
    ↓
MVT
    ↓
Leaflet.VectorGrid
    ↓
Leaflet
    ↓
Browser
```

The main idea is to keep each component responsible for a specific task:

```text
GDAL       → data conversion/import
PostGIS    → spatial storage and queries
Martin     → vector tile generation/serving
Leaflet    → interactive web map
```

This architecture provides a practical foundation for experimenting with larger GIS web applications and for understanding how desktop geospatial data can be transformed into data that can efficiently be displayed in a web browser.