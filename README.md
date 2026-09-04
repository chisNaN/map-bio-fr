https://www.data.gouv.fr/datasets/parcelles-certifiees-en-agriculture-biologique-sur-cartobio?utm_source=chatgpt.com


https://www.data.gouv.fr/api/1/datasets/r/2eb61d76-ebb1-49a9-aaaa-ccd4bb141f93



ogrinfo -so "cartobio-parcelles-2025-francemet-2154.gpkg" 2>&1 | head -40


ogrinfo -dialect SQLite \               
  -sql "SELECT COUNT(*) FROM cartobio_parcelles_2025_francemet_2154" \
  cartobio-parcelles-2025-francemet-2154.gpkg

ogrinfo -so \                           
  "cartobio-parcelles-2025-francemet-2154.gpkg" \
  "cartobio_parcelles_2025_francemet_2154"    

ogr2ogr \
  -f PostgreSQL \
  "PG:host=localhost port=5432 dbname=postgres user=postgres password=mypassword" \
  "cartobio-parcelles-2025-francemet-2154.gpkg" \
  "cartobio_parcelles_2025_francemet_2154" \
  -nln bio.parcelles \
  -lco GEOMETRY_NAME=geom \
  -lco FID=id \
  --config PG_USE_COPY YES \
  -progress

  docker exec -it postgis_container psql -U postgres -d postgres -c "CREATE SCHEMA IF NOT EXISTS bio;"

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

  docker exec -it postgis_container psql -U postgres -d postgres -c "select count(*) from bio.test_parcelles;"

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


  https://maplibre.org/martin/

  https://leafletjs.com/


  https://dbeaver.io/

  https://gdal.org/en/stable/programs/ogr2ogr.html