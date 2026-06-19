# Final Project — Scoria Cones: stato del dataset e spunti

> Versione rivista dopo l'analisi delle slide del corso (vedi `slides/Slides-Index.md`).
> Ogni idea riporta i collegamenti ai deck e alle pagine (numeri di pagina approssimativi).

## Stato attuale del dataset (`CinderConesDataPack/`)

### Etna (Sicilia) — risoluzione massima, ground truth più sporca
- `DEM2m/DEM 2 m Etna-001.tif` — DEM raw 2 m (~2.86 GB)
- `Etna-Eduard-Analytical-NE.tiff`, `-NW.tiff`, `-Texture.tiff` — render Eduard pronti (shading direzionale + texture shading)
- `Coni/coni_monogenici.shp` + `crateri.shp` — **due shapefile separati** (basi e crateri), ~220 features, qualità bassa (falsi positivi/negativi, segmentazioni imprecise)

### El Hierro (Canarie) — risoluzione media, ground truth migliore
- `DEM/dem5m` (+ `.aux.xml`, `.ovr`) — DEM 5 m (~115 MB)
- `Renderings/Hierro-Eduard-Analytical-NE.tiff`, `-NW.tiff`, `-Texture.tiff`
- `Coni/Cones.shp` — **unico shapefile** con basi, crateri *e* fissure miste, ~180 feature, qualità medio-alta

### Jeju (Corea) — solo qualitativo / unsupervised
- `Dem10m/Raster Jeju 5k.tif` (10 m)
- Renderings Eduard NE/NW/Texture
- **Niente shapefile** → solo test qualitativi o task unsupervised

### Aspetti pratici
- Etna DEM ~2.86 GB → gestione a tile/finestre, non si carica in RAM
- `.prj`/`.tfw` danno la georeferenziazione, indispensabili per allineare shapefile↔raster (`rasterio` + `geopandas`)
- Lo shapefile El Hierro mescola tipi di geometria: separare per `tipo` (campo nel `.dbf`)
- Render Eduard già pronti → usabili come canali aggiuntivi; comunque generare slope/aspect/roughness dal DEM è quasi obbligatorio

---

## Mappatura programma del corso

Copertura dei deck tecnici (dettaglio in `slides/Slides-Index.md`):

| Deck | Rilevanza | Cosa si usa |
|---|---|---|
| `01_image_processing` | ★★★ | gradiente (slope/aspect), smoothing, filtri orientabili (shading), Canny, Harris, scale-space |
| `03_image_matching` | ★★★ | detector scale-invariant **LoG/DoG (blob)**, SIFT, selezione di scala |
| `09_segmentation` | ★★★ | Otsu, **connected components (blob)**, color thresholding positive/negative, K-means, Davies-Bouldin |
| `SLIC_Superpixels` | ★★★ | superpixel su grayscale (clustering 5D) |
| `semantic_segmentation` | ★★★ | FCN, encoder-decoder/U-Net |
| `RCNN` | ★★★ | object detection, **valutazione IoU/Jaccard, P/R/F1, confusion matrix**, Faster R-CNN |
| `06_image_representations_intro` | ★★★ | classificazione binaria **positive/negative**, BoW, feature CNN |
| `04_3Dvision_small` | ★★ | inquadramento: il DEM è una **range/depth image** |
| `04_motion_analysis` | ✗ | richiede video/tempo — non applicabile |
| `human_pose`, `Human_in_the_image_action` | ✗ | specifici sull'uomo — non applicabili |

---

## Spunti di progetto (rivisti)

### 1. Detection supervisionata di coni — *modello aggiornato*
Train su El Hierro (GT pulita) → test su Etna (GT rumorosa) e Jeju (qualitativo).
- Patch positive dai poligoni shapefile, negative dal resto del DEM
- Input multicanale: DEM normalizzato + slope + aspect + NE/NW shading + texture shading
- Modello: **Faster R-CNN** su tile (`RCNN`, p.~10-14) *oppure* **U-Net/FCN** per segmentazione semantica base/cratere (`semantic_segmentation`, p.~7-11)
- ⚠️ Cambio rispetto alla v1: **Faster R-CNN al posto di YOLO** (YOLO non è nel programma; R-CNN/Faster R-CNN sì)
- Valutazione **come da slide**: IoU/Jaccard, precision/recall/F1, confusion matrix (`RCNN`, p.~3-9)
- Slide: `RCNN` (detection + valutazione), `semantic_segmentation` (U-Net/FCN), `06_image_representations` (feature)

### 2. Detection con CV classica — *ricalibrata*
Firma morfologica forte: slope alto (25–35°), roughness basso, aspect radiale, cratere = depressione locale.
- Pipeline: smoothing gaussiano → mappa **slope** (= magnitudo gradiente) → soglia **Otsu/adattiva** → **connected components** → filtro per circolarità/area
- **Blob detection LoG/DoG multi-scala** per i crateri circolari a scale diverse
- ⚠️ Cambio rispetto alla v1: **declassato Hough** (non è nel programma) → si usa **LoG/DoG (matching)** + **connected components (segmentation)**, che sono insegnati. Hough resta come extra facoltativo.
- Validazione quantitativa contro shapefile El Hierro (precision/recall, IoU)
- Slide: `01_image_processing` (gradiente p.~13, smoothing p.~17, scale-space p.~38-40), `09_segmentation` (Otsu p.~15-18, connected components p.~13, adattivo p.~20-22), `03_image_matching` (LoG/DoG + scala p.~14-16)

### 3. Classificazione patch positive/negative — *promossa* ⬆️
La più allineata in assoluto: combacia con la Consegna *e* con due deck.
- Patch positive dai poligoni, negative dal resto → classificatore
- `06_image_representations` mostra la classificazione binaria **face/not-face** (p.~4) = identica logica positive/negative
- `09_segmentation` mostra la **skin segmentation con esempi positive/negative** (p.~25-27)
- Feature: **BoW/coding-pooling SIFT** (`06`, p.~10-11) o **feature CNN** (`06`, p.~13-14) → **SVM**
- Slide: `06_image_representations` (classificazione, BoW, CNN), `09_segmentation` (paradigma positive/negative)

### 4. Mining direzioni delle fissure — *extra opzionale* ➡️
El Hierro ha le fissure nello shapefile. Predire la direzione preferenziale di allineamento (PCA su centroidi, Hough lineare su densità).
- ⚠️ PCA/Hough lineare **non sono nel programma** → idea distintiva geologicamente ma meno difendibile all'orale. Tenere come estensione facoltativa.

### 5. Superpixel + clustering (SLIC + K-means) — *nuova* 🆕
SLIC funziona su grayscale → segmentare DEM/shading in superpixel, poi classificare/clusterizzare i superpixel come cono/non-cono.
- Scelta di K con **Davies-Bouldin** (`09_segmentation`, p.~31)
- Ottimo per il task **unsupervised su Jeju** (niente shapefile)
- Slide: `SLIC_Superpixels` (intero, distance measure p.5, algoritmo p.6), `09_segmentation` (K-means p.~28-31)

### 6. Preprocessing/visualizzazione multicanale "CV-grounded" — *nuova* 🆕
Risponde al primo obiettivo del PDF ("new visualization/preprocessing modalities").
- Generare dal DEM: **slope, aspect** (= magnitudo/orientazione gradiente), **roughness** (varianza locale), **scale-space gaussiano**
- Usare i **filtri derivativi orientabili (steerable `G₁^θ`)** per sintetizzare l'hillshade da **qualsiasi direzione di illuminazione** → versione CV-fondata dello "shading" dei geologi
- Slide: `01_image_processing` (gradiente p.~13, filtri orientabili/steerable p.~22, smoothing p.~17, scale-space p.~38-40)

### Inquadramento bonus (3D vision)
Il DEM **è** una range/depth image — l'output dei sensori 3D (LIDAR/RGBD). Si può presentare il progetto come "lavoriamo sulla ricostruzione 3D già disponibile". Slide: `04_3Dvision_small` (sensori attivi p.3, geometria p.4).

---

## Raccomandazione rivista

Combinazione che massimizza la copertura del programma (6 deck tecnici su 8):
1. **Preprocessing multicanale** (idea 6) — `01_image_processing`
2. **Detection classica** (idea 2 ricalibrata: slope/roughness → Otsu + connected components → blob LoG/DoG) — `09_segmentation` + `03_image_matching`
3. **Baseline DL**: **Faster R-CNN** (`RCNN`) o **U-Net/FCN** (`semantic_segmentation`)
4. **Valutazione** IoU/precision/recall/F1 su El Hierro (`RCNN`), **SLIC + K-means** come variante region-based e demo unsupervised su Jeju

La classificazione positive/negative (idea 3) fa da ponte tra detection classica e DL. Etna come stress test (GT rumorosa), Jeju come demo qualitativa finale.

### Non applicabili
`04_motion_analysis` (serve il video/tempo), `human_pose`, `Human_in_the_image_action` (specifici sull'uomo).
