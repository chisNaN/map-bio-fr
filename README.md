# map-bio-fr

Application web cartographique permettant d'afficher les **parcelles déclarées en agriculture biologique en France métropolitaine**.

Ce projet met en œuvre une chaîne SIG web complète :

```text
GeoPackage
    │
    │ GDAL / ogr2ogr
    ▼
PostgreSQL + PostGIS
    │
    │ Martin
    ▼
Tuiles vectorielles MVT
    │
    │ Leaflet + VectorGrid
    ▼
Navigateur web
```

L'objectif du projet est principalement de mettre en pratique l'intégration de plusieurs briques d'un système SIG web :

- **PostgreSQL** : base de données relationnelle
- **PostGIS** : extension spatiale de PostgreSQL
- **DBeaver** : interface d'administration de la base
- **GDAL** : manipulation et import des données géographiques
- **Martin** : serveur de tuiles vectorielles
- **Leaflet** : bibliothèque cartographique côté navigateur
- **Leaflet.VectorGrid** : affichage des tuiles vectorielles MVT
- **Docker / Docker Compose** : exécution des services

---

# 1. Prérequis

Le projet est réalisé sur un **MacBook Pro M1**.

Les éléments suivants sont déjà installés :

- Docker
- Docker Compose
- Homebrew

---

# 2. Créer le projet

Créer le répertoire du projet :

```bash
mkdir map-bio-fr
cd map-bio-fr
```

Le projet aura progressivement cette structure :

```text
map-bio-fr/
├── docker-compose.yml
├── cartobio-parcelles-2025-francemet-2154.gpkg
└── frontend/
    └── index.htm
```

---

# 3. PostgreSQL + PostGIS + Martin avec Docker Compose

PostgreSQL et Martin sont exécutés dans des conteneurs Docker.

Le fichier `docker-compose.yml` utilisé par le projet est le suivant :

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

## 3.1 PostgreSQL + PostGIS

Le service :

```yaml
postgis:
  image: postgis/postgis:16-3.5
```

utilise l'image PostgreSQL avec PostGIS.

Sur le MacBook Pro M1, le projet force l'utilisation de l'architecture AMD64 :

```yaml
platform: linux/amd64
```

Docker Desktop utilise alors son mécanisme de virtualisation/émulation pour exécuter cette image sur Apple Silicon.

Le conteneur est nommé :

```text
postgis_container
```

PostgreSQL est exposé sur :

```text
localhost:5432
```

Les paramètres de connexion sont :

```text
Host       : localhost
Port       : 5432
Database   : postgres
User       : postgres
Password   : mypassword
```

Les données PostgreSQL sont stockées dans un volume Docker :

```text
postgis_data
```

Cela permet de conserver les données de la base indépendamment du cycle de vie du conteneur.

---

# 4. Démarrer les conteneurs

Depuis la racine du projet :

```bash
docker compose up -d
```

Vérifier les conteneurs :

```bash
docker ps
```

On doit retrouver notamment :

```text
postgis_container
martin
```

La stack est donc déjà composée des deux services :

```text
Docker Compose
│
├── postgis_container
│       └── PostgreSQL + PostGIS
│
└── martin
        └── serveur de tuiles
```

---

# 5. Vérifier que PostGIS est installé

Une première vérification consiste à demander directement à PostgreSQL la version de PostGIS :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT PostGIS_Version();"
```

Une réponse similaire à :

```text
 postgis_version
----------------
 3.5.x
(1 row)
```

confirme que PostGIS est disponible.

On peut également vérifier l'extension PostgreSQL :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT extname, extversion FROM pg_extension WHERE extname = 'postgis';"
```

---

# 6. Installer DBeaver

DBeaver permet de travailler graphiquement avec PostgreSQL.

Télécharger **DBeaver Community** depuis le site officiel :

https://dbeaver.io/download/

Pour le MacBook Pro M1, télécharger la version :

```text
macOS → Apple Silicon
```

Installer ensuite l'application normalement.

## 6.1 Créer la connexion PostgreSQL

Dans DBeaver, créer une connexion PostgreSQL avec :

```text
Host       : localhost
Port       : 5432
Database   : postgres
Username   : postgres
Password   : mypassword
```

DBeaver permet notamment de consulter les schémas, tables, colonnes et données de PostGIS.

---

# 7. Télécharger les données CartoBio

Le jeu de données utilisé est celui des parcelles certifiées en agriculture biologique disponibles sur CartoBio.

## Source officielle

La page officielle du jeu de données est :

https://www.data.gouv.fr/datasets/parcelles-certifiees-en-agriculture-biologique-sur-cartobio

La ressource utilisée pour le projet :

https://www.data.gouv.fr/api/1/datasets/r/2eb61d76-ebb1-49a9-aaaa-ccd4bb141f93

Le fichier utilisé est :

```text
cartobio-parcelles-2025-francemet-2154.gpkg
```

Placer le fichier à la racine du projet :

```text
map-bio-fr/
├── cartobio-parcelles-2025-francemet-2154.gpkg
└── docker-compose.yml
```

## Miroir de secours

Si la ressource n'est plus disponible sur data.gouv.fr, un miroir peut être utilisé :

https://mega.nz/file/V58DnKqa#jkh7pUqpFCRMSJ5gPMcOl3xXjtDdM6vd318U-uztdHQ

**La source officielle data.gouv.fr doit être privilégiée lorsque le fichier y est disponible.**

---

# 8. Installer GDAL

GDAL est utilisé pour manipuler les données géographiques.

Avec Homebrew :

```bash
brew install gdal
```

Vérifier l'installation :

```bash
ogrinfo --version
```

GDAL fournit notamment les deux commandes utilisées dans ce projet :

```text
ogrinfo
ogr2ogr
```

`ogrinfo` permet d'inspecter les données.

`ogr2ogr` permet notamment de convertir et d'importer les données dans PostgreSQL/PostGIS.

---

# 9. Inspecter le GeoPackage

Le fichier `.gpkg` est un **GeoPackage**.

Un GeoPackage est un format standard OGC basé sur SQLite permettant notamment de stocker des données vectorielles, leurs attributs, leurs géométries et leur système de coordonnées.

La couche utilisée dans ce projet est :

```text
cartobio_parcelles_2025_francemet_2154
```

On peut l'inspecter avec :

```bash
ogrinfo -so \
  cartobio-parcelles-2025-francemet-2154.gpkg \
  cartobio_parcelles_2025_francemet_2154
```

---

# 10. Compter les parcelles dans le GeoPackage

Le GeoPackage peut être interrogé directement avec le dialecte SQLite :

```bash
ogrinfo -dialect SQLite \
  -sql "SELECT COUNT(*) FROM cartobio_parcelles_2025_francemet_2154" \
  cartobio-parcelles-2025-francemet-2154.gpkg
```

Le fichier utilisé dans ce projet contient environ :

```text
1 232 528 parcelles
```

Cette étape permet notamment de connaître le volume de données avant de commencer l'import dans PostGIS.

---

# 11. Créer le schéma `bio`

Les données CartoBio seront stockées dans un schéma PostgreSQL dédié nommé :

```text
bio
```

Créer le schéma :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "CREATE SCHEMA IF NOT EXISTS bio;"
```

La table finale sera :

```text
bio.parcelles
```

---

# 12. Faire un import de test avec 100 parcelles

Avant de lancer l'import complet de plus d'un million de parcelles, on effectue un test avec seulement 100 entités.

La commande utilisée est :

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

## 12.1 Paramètres importants

### `-f PostgreSQL`

Indique que PostgreSQL est la destination.

### `PG:host=localhost...`

Connexion à PostgreSQL :

```text
Host     : localhost
Port     : 5432
Database : postgres
User     : postgres
Password : mypassword
```

### `-nln bio.test_parcelles`

Définit le nom de la table de destination :

```text
bio.test_parcelles
```

### `-lco GEOMETRY_NAME=geom`

Définit le nom de la colonne géométrique :

```text
geom
```

### `-lco SPATIAL_INDEX=GIST`

Demande la création d'un index spatial GiST.

### `--config PG_USE_COPY YES`

Demande à GDAL d'utiliser PostgreSQL `COPY` pour accélérer l'import.

### `-gt 200000`

Définit la taille des groupes de transactions GDAL.

**Ce paramètre ne limite pas le nombre de parcelles importées.**

### `-limit 100`

C'est lui qui limite volontairement l'import à :

```text
100 entités
```

### `-progress`

Affiche la progression de l'opération.

---

# 13. Vérifier l'import des 100 parcelles

Une fois l'import terminé :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT count(*) FROM bio.test_parcelles;"
```

Résultat attendu :

```text
 count
-------
   100
(1 row)
```

On peut également examiner la table dans DBeaver.

---

# 14. Vérifier les géométries

On peut vérifier le type des géométries :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT GeometryType(geom), COUNT(*) FROM bio.test_parcelles GROUP BY GeometryType(geom);"
```

On peut vérifier le SRID :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT ST_SRID(geom), COUNT(*) FROM bio.test_parcelles GROUP BY ST_SRID(geom);"
```

Pour les données utilisées dans ce projet, les géométries sont des polygones et utilisent le système de coordonnées :

```text
EPSG:2154
```

---

# 15. Supprimer la table de test

Une fois le test validé :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "DROP TABLE bio.test_parcelles;"
```

La table temporaire n'est plus nécessaire.

---

# 16. Importer toutes les parcelles

Une fois le test validé, on lance l'import complet.

La différence essentielle avec la commande précédente est la suppression de :

```text
-limit 100
```

Commande finale :

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

La destination est maintenant :

```text
bio.parcelles
```

et non plus :

```text
bio.test_parcelles
```

L'opération importe les quelque :

```text
1 232 528 parcelles
```

dans PostGIS.

---

# 17. Vérifier le nombre final de parcelles

Une fois l'import terminé :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT count(*) FROM bio.parcelles;"
```

Le nombre doit être cohérent avec celui obtenu précédemment dans le GeoPackage :

```text
1 232 528
```

---

# 18. Vérifier la structure de la table

Afficher les colonnes :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "\d bio.parcelles"
```

La colonne géométrique doit notamment être :

```text
geom
```

L'index spatial créé par GDAL peut également être vérifié avec :

```bash
docker exec -it postgis_container \
psql -U postgres -d postgres \
-c "SELECT indexname FROM pg_indexes WHERE tablename = 'parcelles';"
```

---

# 19. Martin : serveur de tuiles vectorielles

Une fois les données disponibles dans PostGIS, il faut pouvoir les transmettre efficacement au navigateur.

Le projet utilise **Martin**, le serveur de tuiles de MapLibre.

Site officiel :

https://maplibre.org/martin/

Martin se connecte à PostgreSQL/PostGIS et permet d'exposer les données sous forme de **tuiles vectorielles Mapbox Vector Tiles (MVT)**.

Le principe devient :

```text
PostGIS
   │
   │ requête
   ▼
Martin
   │
   │ MVT
   ▼
Navigateur
```

---

# 20. Martin dans Docker Compose

Martin est déjà présent dans le `docker-compose.yml` du projet :

```yaml
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
```

Martin écoute donc sur :

```text
http://localhost:3000
```

### Connexion à PostgreSQL

Martin utilise :

```text
postgres://postgres:mypassword@postgis:5432/postgres
```

Le point important est :

```text
@postgis:5432
```

`postgis` est le nom du **service Docker Compose**.

Martin et PostgreSQL communiquent donc à l'intérieur du réseau Docker.

Il ne faut pas utiliser `localhost` dans cette URL pour désigner le conteneur PostgreSQL.

---

# 21. Vérifier le catalogue Martin

Ouvrir :

```text
http://localhost:3000/catalog
```

Martin doit détecter la table :

```text
bio.parcelles
```

et proposer notamment une source :

```text
parcelles
```

correspondant à :

```text
bio.parcelles.geom
```

Le catalogue permet ainsi de vérifier que Martin voit correctement la table PostGIS.

---

# 22. URL des tuiles vectorielles

La source Martin :

```text
parcelles
```

est accessible sous forme de tuiles :

```text
http://localhost:3000/parcelles/{z}/{x}/{y}
```

Les trois paramètres représentent :

```text
z = niveau de zoom
x = colonne de tuile
y = ligne de tuile
```

Les valeurs sont normalement remplacées automatiquement par le client cartographique.

Par exemple, une requête réelle peut ressembler à :

```text
http://localhost:3000/parcelles/6/32/22
```

Le résultat est une tuile vectorielle binaire au format MVT.

---

# 23. Créer le frontend Leaflet

Créer le répertoire :

```bash
mkdir frontend
```

Puis :

```bash
cd frontend
```

Créer :

```text
index.htm
```

La structure devient :

```text
map-bio-fr/
├── docker-compose.yml
├── cartobio-parcelles-2025-francemet-2154.gpkg
└── frontend/
    └── index.htm
```

---

# 24. Instancier Leaflet

Le fichier `index.htm` utilise Leaflet depuis le CDN `unpkg`.

Aucun attribut `integrity` ou `crossorigin` n'est utilisé dans cette version.

```html
<!DOCTYPE html>
<html lang="fr">

<head>

  <meta charset="UTF-8">

  <meta
    name="viewport"
    content="width=device-width, initial-scale=1.0"
  >

  <title>CartoBio</title>

  <link
    rel="stylesheet"
    href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
  >

  <style>

    html,
    body {
      margin: 0;
      padding: 0;
      width: 100%;
      height: 100%;
    }

    #map {
      width: 100%;
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

    const map = L.map('map').setView(
      [46.5, 2.5],
      6
    );

    L.tileLayer(
      'https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
      {
        attribution: '&copy; OpenStreetMap contributors'
      }
    ).addTo(map);


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

  </script>

</body>

</html>
```

---

# 25. Comprendre `L.vectorGrid.protobuf`

Le code :

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

connecte directement Leaflet à Martin.

La chaîne est :

```text
Leaflet
   │
   │ HTTP
   ▼
Martin :3000
   │
   ▼
PostGIS
   │
   ▼
bio.parcelles
```

Martin renvoie une tuile MVT correspondant à la zone actuellement visible.

---

# 26. Limiter l'affichage avec `minZoom`

Le projet utilise :

```javascript
minZoom: 8
```

Cela signifie que les parcelles ne sont affichées qu'à partir du niveau de zoom 8.

Cette limitation est importante car la table contient plus d'un million de polygones.

À faible niveau de zoom, afficher simultanément toutes les parcelles pourrait produire une représentation cartographique difficile à interpréter.

Le réglage :

```javascript
fillOpacity: 0.15
```

permet également de limiter l'effet visuel d'accumulation des polygones.

---

# 27. Afficher les attributs d'une parcelle

La propriété :

```javascript
interactive: true
```

permet de rendre les entités vectorielles interactives.

On peut alors écouter l'événement `click`.

Ajouter :

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

La fonction :

```javascript
(e) => {
  ...
}
```

est une **fonction fléchée anonyme** utilisée comme callback.

Elle est passée directement à :

```javascript
parcelles.on('click', ...)
```

---

# 28. Fonctionnement du popup

Lorsqu'une parcelle est sélectionnée :

```text
Clic sur une parcelle
       │
       ▼
événement "click"
       │
       ▼
e.layer
       │
       ▼
e.layer.properties
       │
       ▼
attributs de la parcelle
       │
       ▼
L.popup()
```

Les propriétés de la parcelle sont parcourues avec :

```javascript
Object.entries(properties)
```

Chaque couple :

```text
clé → valeur
```

est ensuite ajouté au contenu du popup.

---

# 29. Lancer le frontend

Depuis le répertoire `frontend` :

```bash
npx serve -l 5500
```

Le serveur web statique écoute alors sur :

```text
http://localhost:5500
```

Ouvrir :

```text
http://localhost:5500
```

Martin continue quant à lui d'écouter sur :

```text
http://localhost:3000
```

---

# 30. Architecture finale

L'architecture complète du projet est :

```text
                         INTERNET
                            │
                            ▼
                    ┌───────────────┐
                    │  data.gouv.fr │
                    └───────┬───────┘
                            │
                            │ GeoPackage
                            ▼
              cartobio-parcelles-2025...
                         .gpkg
                            │
                            │ GDAL / ogr2ogr
                            ▼
                 ┌─────────────────────┐
                 │ PostgreSQL + PostGIS│
                 │                     │
                 │    bio.parcelles    │
                 └──────────┬──────────┘
                            │
                            │ SQL / spatial data
                            ▼
                    ┌───────────────┐
                    │    Martin     │
                    │    :3000      │
                    └───────┬───────┘
                            │
                            │ MVT
                            ▼
                    ┌───────────────┐
                    │    Leaflet    │
                    │ VectorGrid    │
                    └───────┬───────┘
                            │
                            ▼
                       Navigateur
                       :5500
```

---

# 31. Rôle de chaque brique

| Brique | Rôle |
|---|---|
| GeoPackage | Fichier contenant les données géographiques d'origine |
| GDAL | Lecture, inspection et import des données |
| PostgreSQL | Base de données relationnelle |
| PostGIS | Gestion des données et requêtes spatiales |
| DBeaver | Administration et exploration de PostgreSQL |
| Martin | Transformation des données PostGIS en tuiles vectorielles |
| MVT | Format des tuiles vectorielles transmises au navigateur |
| Leaflet | Bibliothèque cartographique JavaScript |
| VectorGrid | Lecture et affichage des tuiles MVT dans Leaflet |
| Docker | Isolation et exécution des services |
| Docker Compose | Orchestration de PostgreSQL/PostGIS et Martin |
| `npx serve` | Serveur HTTP local du frontend |

---

# 32. Les trois ports du projet

```text
5432
 │
 └── PostgreSQL/PostGIS

3000
 │
 └── Martin

5500
 │
 └── Frontend Leaflet
```

La communication principale est :

```text
Navigateur :5500
       │
       │ HTTP / MVT
       ▼
Martin :3000
       │
       │ PostgreSQL
       ▼
PostGIS :5432
```

---

# 33. Résumé de la mise en œuvre

Le projet suit finalement une architecture SIG web classique :

```text
DONNÉES
   ↓
GeoPackage
   ↓
GDAL
   ↓
BASE DE DONNÉES SPATIALE
   ↓
PostgreSQL + PostGIS
   ↓
SERVEUR DE TUILES
   ↓
Martin
   ↓
TUILES VECTORIELLES
   ↓
MVT
   ↓
CLIENT CARTOGRAPHIQUE
   ↓
Leaflet + VectorGrid
```

L'intérêt de cette architecture est de séparer clairement les responsabilités :

- le **GeoPackage** constitue la donnée source ;
- **GDAL** permet de la transformer et de l'importer ;
- **PostGIS** stocke et interroge les données spatiales ;
- **Martin** sert les données sous une forme optimisée pour la cartographie web ;
- **Leaflet** assure la visualisation et l'interaction côté navigateur.

Cette séparation permet ensuite de faire évoluer le projet : requêtes spatiales PostGIS, filtrage des parcelles, agrégations par commune ou département, styles cartographiques dynamiques, différentes sources de données, ou encore remplacement de Leaflet par MapLibre GL JS ou une autre bibliothèque cartographique.