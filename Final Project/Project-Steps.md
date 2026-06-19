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

### Step 0 — Caricamento e ispezione DEM
- **Fai:** apri i 3 DEM con `rasterio`; stampa `shape (H,W)`, `dtype`, risoluzione, nodata→NaN.
- **Formula/slide:** nessuna. Il DEM è una *range image* → `04_3Dvision_small` p.3.
- **Test:** risoluzioni = 2 / 5 / 10 m; nessun crash; quote in metri plausibili.

### Step 1 — Canali morfometrici: slope, aspect, roughness
- **Fai:** gradiente Sobel → **slope** = `deg(arctan√(gx²+gy²))`, **aspect** = `arctan2(gy,gx)`,
  **roughness** = dev. standard locale (finestra k×k). Carica anche il canale **texture** Eduard.
- **Formula/slide:** gradiente `M=√(gx²+gy²)`, `θ=arctan(gy/gx)` → `01_image_processing` **p.13**.
- **Test:** slope alto (25–35°) sui fianchi; aspect radiale attorno ai coni.

### Step 2 — Hillshade sintetico con filtri orientabili  ➕ *(contributo: rappresentazione)*
- **Fai:** derivate di gaussiana `G₁⁰`,`G₁⁹⁰`; per ogni θ combina
  `G₁^θ = cosθ·G₁⁰ + sinθ·G₁⁹⁰` → hillshade da qualsiasi direzione di luce (generato da te, non
  i render NE/NW forniti).
- **Scopo:** rappresentazione CV-fondata e tua → smarca da Frattini; risponde al 1° obiettivo
  del PDF.
- **Formula/slide:** filtri orientabili `G₁^θ=cosθ·G₁⁰+sinθ·G₁⁹⁰` → `01_image_processing` **p.22**.
- **Test:** l'hillshade θ=NE somiglia al render Eduard NE; ruotando θ l'ombra ruota coerente.

### Step 3 — Maschera morfometrica
- **Fai:** maschera binaria `slope∈[25°,35°]` AND `roughness < soglia` (soglia da percentile).
- **Formula/slide:** thresholding → `09_segmentation` **p.13**; (Otsu p.15-18 come alternativa).
- **Test:** la maschera copre i fianchi dei coni noti.

### Step 4 — Candidati LoG (baseline che mostra il problema)
- **Fai:** `blob_log` sul texture mascherato, config ad alto recall → candidati `(row,col,σ)`.
- **Formula/slide:** **LoG = blob detector scale-invariant** → `03_image_matching` **p.14-16**.
- **Test:** recall ≈ 0.96 sui crateri; **osserva che due crateri vicini = 1 solo blob**.

### Step 5 — Watershed: separa coni ADIACENTI  *(cuore del contributo)*
- **Fai:** crateri = depressioni → marcatori sui minimi locali → **watershed** → 1 bacino per
  cratere, anche se affiancati (marker-controlled, per evitare over-segmentation).
- **Formula/slide:** **watershed** (region-based) → `09_segmentation` **p.8**; descrizione
  "separa catchment basins" → `SLIC_Superpixels` **p.4**.
- **Test (decisivo):** dove il LoG dava 1 blob su coni attaccati, il watershed dà **2 regioni**.

### Step 6 — LoG multi-scala: separa coni ANNIDATI  ➕ *(completa il contributo)*
- **Fai:** tieni le risposte LoG a **scale diverse** nella stessa posizione, senza fonderle tra
  scale → cratere piccolo *dentro* base grande = due σ = due rilevamenti.
- **Scopo:** copre il caso annidato-per-scala che il watershed (adiacenti) non gestisce →
  contributo completo: "separo adiacenti **e** annidati".
- **Formula/slide:** selezione di scala LoG/DoG → `03_image_matching` **p.14-16**.
- **Test:** un cono con cratere annidato dà 2 rilevamenti a σ diversi anziché 1.

### Step 7 — Classificazione cono / non-cono
- **Fai:** poche feature per candidato (slope, roughness, texture, circolarità) →
  **Random Forest** → tieni i candidati classificati come cono.
- **Formula/slide:** classificazione positive/negative → `06_image_representations` **p.2-5**.
- **Test:** F1 ragionevole; texture tra le feature più importanti.

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
