# Project-Steps — piano

> **Progetto:** detection di coni di scoria da DEM. **Contributo distintivo:** separare i coni
> **annidati/adiacenti** che il detector a blob fonde in un'unica rilevazione (limite sollevato
> dalla prof all'esame di Frattini — vedi `example/`).
>
> **Principio guida:** poche cose, fatte bene. Ogni step = *implementa → testa → slide*. Si
> avanza solo a Test verde. Se in corso d'opera diventa troppo lungo, si taglia dagli step
> contrassegnati **➕** (aggiunte) ripartendo dall'ultimo. Riferimenti:
> [`../slides/Slides-Index.md`](../slides/Slides-Index.md).
>
> **Esclusa per scelta:** CNN / U-Net (costo GPU/tempo, diluisce il focus).
>
> ⚠️ Prerequisiti: dataset in `CinderConesDataPack/` (in download) + `pip install rasterio
> geopandas` (mancano).

---

### Step 0 — Caricamento e ispezione DEM  ✅ FATTO
- **Fai:** apri i 3 DEM con `rasterio`; stampa `shape (H,W)`, `dtype`, risoluzione, nodata→NaN.
- **Formula/slide:** nessuna. Il DEM è una *range image* → `04_3Dvision_small` p.3.
- **Test:** risoluzioni = 2 / 5 / 10 m; nessun crash; quote in metri plausibili.

### Step 1 — Canali morfometrici: slope, aspect, roughness  ✅ FATTO
- **Fai:** gradiente Sobel → **slope** = `deg(arctan√(gx²+gy²))`, **aspect** = `arctan2(gy,gx)`,
  **roughness** = dev. standard locale (finestra k×k). Carica anche il canale **texture** Eduard.
- **Formula/slide:** gradiente `M=√(gx²+gy²)`, `θ=arctan(gy/gx)` → `01_image_processing` **p.13**.
- **Esito reale:** slope bordi **mediana 18.8°** vs sfondo **8.1°** (NON 25–35°); aspect radiale.

### Step 2 — Hillshade sintetico con filtri orientabili  ➕ ✅ FATTO *(contributo: rappresentazione)*
- **Fai:** derivate di gaussiana `G₁⁰`,`G₁⁹⁰`; per ogni θ combina
  `G₁^θ = cosθ·G₁⁰ + sinθ·G₁⁹⁰` → hillshade da qualsiasi direzione di luce (generato da te, non
  i render NE/NW forniti).
- **Scopo:** rappresentazione CV-fondata e tua → smarca da Frattini; risponde al 1° obiettivo
  del PDF.
- **Formula/slide:** filtri orientabili `G₁^θ=cosθ·G₁⁰+sinθ·G₁⁹⁰` → `01_image_processing` **p.22**.
- **Esito reale:** σ=2px; sintetico NE **correla 0.81** col render Eduard NE; NE/NW/SE ruotano
  coerenti. (Segno invertito vs Eduard → shading = −derivata direzionale.) Fig `fig_steerable.png`.

### Step 3 — Maschera morfometrica  ✅ FATTO (data-driven)
- **Fai:** maschera binaria = `slope > T` con **T scelto da Otsu** (non a mano) sul DEM ElHierro.
- **Esito reale:** Otsu = **17.6°** (≈ rim misurato 18.8°) → tiene **31%** dell'isola ma copre
  **96%** dei coni (recall per-cono: ≥10% del disco del cono è ripido). Piano fisso `[25,35°]`
  dava recall solo **21%** → scartato; AND con `roughness` non aggiunge nulla → tolto.
- **Limite noto:** la scogliera costiera (ripida) sopravvive → la scarterà il classificatore (Step 7).
- **Formula/slide:** thresholding + **Otsu** (between-class variance) → `09_segmentation` **p.13-18**.
- **Test:** ✔ maschera selettiva (31% isola) ma alta recall sui coni (96%). Fig `fig_mask.png`.

### Step 4 — Candidati LoG (baseline che mostra il problema)  ✅ FATTO
- **Fai:** `skimage.feature.blob_log` sul **texture mascherato** (DS×2, `min_sigma=1.5,
  max_sigma=12, num_sigma=6, threshold=0.08, overlap=0.3`); candidati = blob col centro in maschera.
- **Esito reale (=Frattini):** 22.065 candidati, **recall 0.964** (189/196, 7 persi piccoli/erosi),
  **precision 0.009** (migliaia di FP → li pulisce la RF, Step 7). Il **gap di adiacenza**: con
  assegnamento **one-to-one** solo 186/196 hanno una rilevazione unica → **10 coni persi** (Frattini: 11)
  perché coni vicini competono per gli stessi blob. Fig `fig_log.png`.
- **Formula/slide:** **LoG = blob detector scale-invariant** → `03_image_matching` **p.14-16**.
- **Test:** ✔ recall ≈ 0.96 (come riferimento); il merge NON appare a alta-recall ma sotto one-to-one
  (gap 10 ≈ 11) → motiva il watershed (Step 5).

### Step 5 — Watershed: separa coni ADIACENTI  ✅ FATTO *(cuore del contributo)*
- **Fai:** DEM **detrendato** (high-pass DoG: `gauss σ=1.5 − gauss σ=15` su DS×2, così un cono
  piccolo su pendio regionale diventa un dosso locale) → **marker** = massimi locali significativi
  (`maximum_filter size=5`, `detrend>2.5`) ristretti al vicinato della maschera Step 3
  (`binary_dilation(mask, disk(3))`) → `watershed(-detrend, markers, mask=valid)` marker-controlled.
- **Esito reale:** **4.427 bacini** (over-segmenta come il pool LoG → li pota la RF, Step 7);
  **184/196** coni in un bacino proprio; **TEST: 3/4 coppie GT adiacenti** finiscono in bacini
  distinti; degli **10 coni persi** dal LoG one-to-one, **8** ora hanno un bacino separato dal
  vicino con cui competevano. La 4ª coppia è strettamente **annidata** (centroidi ~10px, niente
  sella) → demandata allo Step 6. Fig `fig_watershed.png`.
- **Formula/slide:** **watershed** (region-based) → `09_segmentation` **p.8**; descrizione
  "separa catchment basins" → `SLIC_Superpixels` **p.4**.
- **Test (decisivo):** ✔ dove il LoG dava 1 blob su coni attaccati, il watershed dà **2 regioni**
  (3/4 coppie; 8/10 coni recuperati).

### Step 6 — LoG multi-scala: separa coni ANNIDATI  ➕ ✅ FATTO *(completa il contributo)*
- **Fai:** `blob_log` IDENTICO allo Step 4 ma `overlap=1.0` invece di `0.3` → le risposte LoG a
  **scale diverse** non vengono fuse/soppresse (il cratere piccolo annidato non viene scartato dal
  blob più grande). Stesso assegnamento one-to-one (k=25, 200m) dello Step 4 per il confronto.
- **Esito reale:** pool candidati $22.065\to27.112$; **gap one-to-one $10\to5$** (recupera 5 coni:
  ids [7,52,64,133,153]). I 5 ancora persi non hanno scala separabile (hard core).
- **Onestà (nel notebook+PDF):** i 5 recuperati sono un MIX (annidati veri + coni che beneficiano
  solo del pool più denso) → guadagno reale ma non esclusivamente "annidato". SCARTATO il match
  **scale-aware** (σ blob ≈ raggio cratere): peggiora tantissimo il gap (43-53) perché i raggi GT
  vengono dai contorni aperti/rumorosi → si tiene l'assegnamento semplice. Il segnale a scala dei
  crateri sul texture/DEM è debole (LoG ~σ3, niente firma di scala pulita per la coppia 150/151).
- **Formula/slide:** selezione di scala LoG/DoG → `03_image_matching` **p.14-16**. Fig `fig_multiscale.png`.
- **Test:** ✔ tenendo le scale annidate il gap one-to-one scende (10→5). Step 5 (regione, adiacenti)
  + Step 6 (scala, annidati) = i due modi in cui i vicini si fondono.

### Step 7 — Classificazione cono / non-cono  ✅ FATTO
- **Input (deciso con l'utente):** candidati = **UNIONE Step 5 + Step 6**, ma in forma di PUNTI:
  **summit markers** del watershed (cime dei coni — NON i centroidi dei bacini, che coprivano solo
  ~48/196) + **blob multiscala** (`cand_ms`). Dedup NMS a **12px** (watershed tenuto per primo).
  → **17.296 candidati** (WS 4.258 + LoG 13.038), soffitto di recall del pool **183/196 (0.93)**.
- **Fai:** 8 feature per candidato (mean slope/rough/tex, **coerenza radiale dell'aspect**,
  contrasto pendenza anello-cima, std pendenza anello, range quota, raggio) → **Random Forest**
  (`class_weight=balanced_subsample`). Valutazione **out-of-fold** (5-fold `cross_val_predict`,
  niente leakage spaziale). Punto operativo = soglia più alta che tiene ≥85% dei coni recuperabili.
- **Esito reale:** **OOF AP 0.126** vs prior 0.035 (**×3.6**); best-F1 **0.225**. Top feature:
  **texture 0.18, coerenza radiale 0.15** ✓. A thr 0.10 il pool scende **17.296→3.264 (×5)**
  tenendo **157/196 coni (0.80)**. Fig `fig_rf.png`. (Classificatore *debole-ma-reale*, onesto
  per un problema sbilanciato al 3%.)
- **Scelta provvisoria:** lo **Step 8 confronterà `solo Step 5` vs `Step 5+6`**; se il 6
  non aggiunge recall unica si tiene solo Step 5.
- **SCARTATI (nel notebook+PDF):** centroidi bacino (→summit), soglia fissa 0.5 (→PR-AUC),
  split random (leakage →OOF), dedup 5px (rumoroso →12px).
- **Formula/slide:** classificazione positive/negative → `06_image_representations` **p.2-5**.
- **Test:** ✔ F1 ragionevole (0.225) e **texture la feature più importante**.

### Step 8 — Valutazione: prova del contributo
- **Fai:** matching candidati↔GT con **assegnamento uno-a-molti** (non greedy 1-1); calcola
  **IoU, Precision/Recall/F1**; metrica mirata = **% di coppie vicine/annidate separate**
  (nostra pipeline vs baseline LoG+greedy).
- **Formula/slide:** **IoU/Jaccard** `RCNN` p.2-4, **Precision/Recall/F1** `RCNN` **p.6**.
- **Test:** sulle coppie difficili recall nostra > baseline; sul resto ≈ pari.

### Step 9 — Jeju unsupervised: SLIC + clustering  ➕ *(dataset senza ground truth)*
- **Fai:** **SLIC** superpixel sul DEM/shading → **K-means** sui superpixel (K via
  **Davies-Bouldin**) → regioni cono/non-cono senza etichette. Valutazione qualitativa.
- **Scopo:** coprire Jeju (no GT) in modo unsupervised, non solo zero-shot.
- **Formula/slide:** distanza 5D SLIC `Ds=d_lab+(m/S)·d_xy` → `SLIC_Superpixels` **p.5-6**;
  K-means + Davies-Bouldin → `09_segmentation` **p.28-31**.
- **Test:** i superpixel aderiscono ai bordi dei coni; i cluster isolano gli oreum dalle piane.

### Step 10 (bonus) — Allineamento coni–fissure  ➕⭐ *(extra qualitativo, dopo la valutazione)*
- **Fai:** sovrapporre gli azimut delle fissure (El Hierro `Id=3`, campo `Prj_Az`) ai coni
  rilevati e verificare visivamente che i coni si dispongano lungo le fissure.
- **Scopo:** conferma geologica del risultato ("i coni si allineano lungo le fissure", come da
  consegna sui *linear geological patterns*) — bella slide all'orale, non parte della pipeline.
- **Nota:** solo qualitativo; il *mining* delle direzioni (PCA/Hough) è fuori programma. Il
  campo `pericolosi` (livello di rischio) **non** si usa: è tematico, non visivo.
- **Test:** una frazione consistente di coni cade entro pochi gradi/metri da una fissura.

---

## Ordine
`0 → 1 → 2 → 3 → 4` (preprocessing + baseline) → **`5 → 6`** (separazione = contributo) → `7`
(classifica) → **`8`** (numeri che dimostrano la tesi) → `9` (Jeju unsupervised).
**Se diventa lungo**, taglia in quest'ordine: prima `9`, poi `6`, poi `2`.

## Mappa step → slide → formula
| Step | Slide | Formula |
|---|---|---|
| 0 | `04_3Dvision_small` p.3 | — |
| 1 | `01_image_processing` p.13 | `M=√(gx²+gy²)`, `θ=arctan(gy/gx)` |
| 2 ➕ | `01_image_processing` p.22 | `G₁^θ=cosθ·G₁⁰+sinθ·G₁⁹⁰` |
| 3 | `09_segmentation` p.13-18 | thresholding / Otsu |
| 4 | `03_image_matching` p.14-16 | LoG (scale-invariant blob) |
| 5 | `09_segmentation` p.8 · `SLIC` p.4 | watershed (separazione bacini) |
| 6 ➕ | `03_image_matching` p.14-16 | LoG multi-scala (selezione di scala) |
| 7 | `06_image_representations` p.2-5 | classificazione positive/negative |
| 8 | `RCNN` p.2-6 | IoU/Jaccard, Precision/Recall/F1 |
| 9 ➕ | `SLIC` p.5-6 · `09` p.28-31 | SLIC `Ds=d_lab+(m/S)·d_xy`, K-means + Davies-Bouldin |
