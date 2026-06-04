# Final Project — Scoria Cones: stato del dataset e spunti

## Stato attuale del dataset (`CinderConesDataPack/`)

### Etna (Sicilia) — risoluzione massima, ground truth più sporca
- `DEM2m/DEM 2 m Etna-001.tif` — DEM raw 2 m (~2.86 GB)
- `Etna-Eduard-Analytical-NE.tiff`, `Etna-Eduard-Analytical-NW.tiff`, `Etna-Eduard-Texture.tiff` — render Eduard già pronti (shading direzionale + texture shading)
- `Coni/coni_monogenici.shp` + `crateri.shp` — **due shapefile separati** (basi e crateri), ~220 features, qualità bassa (falsi positivi/negativi, segmentazioni imprecise)

### El Hierro (Canarie) — risoluzione media, ground truth migliore
- `DEM/dem5m` (+ `.aux.xml`, `.ovr`) — DEM 5 m (~115 MB)
- `Renderings/Hierro-Eduard-Analytical-NE.tiff`, `-NW.tiff`, `-Texture.tiff`
- `Coni/Cones.shp` — **unico shapefile** con basi, crateri *e* fissure miste, ~180 feature, qualità medio-alta

### Jeju (Corea) — solo qualitativo / unsupervised
- `Dem10m/Raster Jeju 5k.tif` (10 m)
- Renderings Eduard NE/NW/Texture
- **Niente shapefile** → utilizzabile solo per test qualitativi o task unsupervised

### Aspetti pratici da considerare subito
- Etna DEM ~2.86 GB → serve gestione a tile/finestre, non si carica in RAM
- I `.prj`/`.tfw` danno la georeferenziazione, indispensabili per allineare shapefile↔raster (probabilmente `rasterio` + `geopandas`)
- Lo shapefile El Hierro mescola tipi di geometria: vanno separati per `tipo` (campo nel `.dbf`)
- Gli Eduard rendering sono già pronti → usabili come canali aggiuntivi senza ricalcolarli, ma generare anche slope/aspect/roughness dal DEM è quasi obbligatorio

## Spunti di progetto

### 1. Detection supervisionata di coni (strada più "standard")
Train su El Hierro (GT pulita) → test su Etna (GT rumorosa) e Jeju (qualitativo).
- Estrai patch positive dai poligoni shapefile, negative dal resto del DEM
- Input multicanale: DEM normalizzato + slope + aspect + NE/NW shading + texture shading
- Modello: YOLO/Faster-RCNN su tile, oppure U-Net per segmentazione semantica (base/cratere come due classi)
- Punto interessante: **transfer cross-vulcano** (generalizzazione tra Etna ed El Hierro, morfologie diverse)

### 2. Detection unsupervised / computer vision classica (più affine al corso)
I coni hanno firma morfologica forte: alto slope (25–35°), basso roughness, aspect radiale dal centro, cratere = depressione locale.
- Pipeline: smoothing → mappa slope → soglia → blob analysis → filtro per circolarità/area
- **Hough circolare** sui crateri nelle texture-shading
- Validazione quantitativa contro shapefile El Hierro (precision/recall, IoU)
- Mostri che metodi classici fanno l'80% del lavoro senza training — narrativa forte

### 3. Classificazione morfologica dei coni
Una volta detectati, classificarli in: *perfetto / parziale / sovrapposto / eroso*. Serve creare/etichettare le classi a mano (poche decine di esempi bastano).
- Feature: simmetria radiale, n. crateri, rapporto base/cratere, varianza dello slope, deviazione dall'ellisse
- Classificatore: anche solo Random Forest su feature handcrafted

### 4. Mining direzioni delle fissure (originale, sfrutta El Hierro)
El Hierro ha già le linee di fissura nello shapefile. Idea: predire la **direzione preferenziale** di allineamento dei coni da analisi spaziale (PCA su cluster di centroidi, Hough lineare su mappe di densità). Più "geologico" che CV, ma molto distintivo.

## Raccomandazione

Partire con **(2) + validazione su (1)**: pipeline classica di detection guidata dalla morfologia (slope/roughness/Hough), valutata quantitativamente sulla GT di El Hierro, con confronto contro un baseline CNN/U-Net addestrato sugli stessi dati. Così si coprono preprocessing, visualizzazione multicanale, detection classica *e* deep — tutti i bullet del PDF in un colpo. Etna come stress test (GT rumorosa → discutere perché P/R cala) e Jeju come demo qualitativa finale.