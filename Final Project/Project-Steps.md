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
- **Scelta provvisoria:** la configurazione di input si decide allo **Step 7.5** (ablation); se il 6
  non aggiunge recall unica si tiene solo Step 5.
- **SCARTATI (nel notebook+PDF):** centroidi bacino (→summit), soglia fissa 0.5 (→PR-AUC),
  split random (leakage →OOF), dedup 5px (rumoroso →12px).
- **Formula/slide:** classificazione positive/negative → `06_image_representations` **p.2-5**.
- **Test:** ✔ F1 ragionevole (0.225) e **texture la feature più importante**.

### Step 7.5 — Ablation: `solo Step 5` vs `Step 5+6` (scelta dell'input)
- **Fai:** rifai la classificazione (Step 7) con DUE pool di candidati — (a) **solo summit watershed**,
  (b) **summit + blob multiscala** — e confronta a parità di metodo. Metrica focalizzata: **coni
  unici recuperati** dal +6, **coppie annidate separate**, e **costo in falsi positivi** (precisione
  a parità di recall). Decide la configurazione finale della pipeline.
- **Sospetto da verificare:** il +6 recupera ~1 cono esclusivo (coppia annidata 150/151) ma porta
  ~13k blob ridondanti → potrebbe peggiorare la precisione più di quanto aggiunga in recall. Se così,
  si tiene **solo Step 5** (più semplice/difendibile) e lo Step 6 resta estensione già quantificata.
- **Formula/slide:** ablation study / model selection (confronto controllato).
- **Test:** numeri chiari su quale input vince; esito atteso = `solo Step 5` salvo guadagno netto del +6.

### Step 8 — Valutazione: prova del contributo  ✅ FATTO
- **Fai:** sulla pipeline **scelta allo Step 7.5** (`Step 5+6` RF-potata, `kept`) vs **baseline
  LoG+greedy** (Step 4 `cand`), a parità di regole e tolleranza (1 raggio cratere, min 80m):
  recall (coverage), precision (detection su un cratere), F1, **IoU medio** (cerchio detection vs
  cerchio GT), **adjacency gap** (greedy 1-1), e la metrica mirata **% coppie adiacenti/annidate
  separate**.
- **Esito reale:** baseline 22.065 dets, recall 0.776, **prec 0.024, F1 0.047**, IoU 0.124, gap 46,
  **coppie 3/4**. Nostra 3.264 dets, recall 0.801, **prec 0.098 (×4.1), F1 0.174 (×3.7)**, IoU 0.164,
  gap 39, **coppie 4/4**. → vince su OGNI metrica; il guadagno P/F1 è l'effetto RF, la separazione
  (gap 46→39, coppie 3/4→4/4) è il contributo. Zoom su coppia adiacente: baseline 1 blob fuso,
  nostra 2 detection distinte. Fig `fig_eval.png` + Table `tab:eval`.
- **Nota onestà:** Step 8 usa tolleranza stretta (1 raggio), NON i 200m del diagnostico Step 4 →
  gap assoluti più grandi qui; conta il confronto controllato (nostra vs baseline a parità di regole).
- **Formula/slide:** **IoU/Jaccard** `RCNN` p.2-4, **Precision/Recall/F1** `RCNN` **p.6**.
- **Test:** ✔ nostra > baseline su tutto; coppie difficili 4/4 vs 3/4.

### Step 9 — Jeju unsupervised: SLIC + clustering  ➕ ✅ FATTO *(dataset senza ground truth)*
- **Domanda giusta (deciso con l'utente):** NON "il modello di El Hierro generalizza a Jeju?"
  (zero-shot, NON misurabile senza GT) ma "**senza etichette, la stessa rappresentazione CV
  (slope/texture/rilievo locale) fa emergere i coni come cluster?**" → prova QUALITATIVA di
  robustezza della *rappresentazione*, non transfer del classificatore.
- **Fai:** **SLIC** superpixel (DS×3, ~5.4k superpixel, su immagine appearance [texture,slope,rough],
  `mask`=terra) → feature per superpixel [texture, slope, roughness, **rilievo locale DoG**] →
  **K-means** con **K via Davies-Bouldin** (K=2..6) → cluster cono-like = max (slope+tex+relief).
- **Esito reale:** K=2 (DB min 0.71); il cluster cono-like copre **~6% della terra**, slope media
  **17.7°** (≈ rim El Hierro 18.8° → coerenza della rappresentazione!), rilievo locale +22.8m (veri
  dossi). Zoom sulle piane: i blob del cluster cadono **sugli oreum**. Fig `fig_jeju.png`. Onestà
  scritta: qualitativo (no GT da scorare); K grezzo → il cluster prende anche Hallasan/scogliera
  (stessa confusione ripido-non-cono che la RF supervisionata toglieva, qui no).
- **Formula/slide:** SLIC `Ds=d_lab+(m/S)·d_xy` → `SLIC_Superpixels` **p.5-6**; K-means +
  Davies-Bouldin → `09_segmentation` **p.28-31**.
- **Test:** ✔ un cluster isola la terra cono-like (6%), slope ≈ El Hierro; gli oreum emergono nel zoom.

### Step 9.5 — Jeju zero-shot: RF di El Hierro applicato a Jeju  ➕ ✅ FATTO *(aggiunto per allinearsi a Frattini)*
- **Perché:** Frattini fa lo zero-shot su Jeju (RF trasferito), io no → aggiunto per coprire anche
  l'asse "generalizzazione del modello" tenendo il vantaggio della separazione.
- **Fai:** stesso `rf` addestrato su El Hierro (Step 7) applicato ai candidati Jeju (costruiti col
  metodo Step 5+6), 8 feature sui canali di Jeju, **soglia di El Hierro INVARIATA** (come Frattini
  con 0.64).
- **Esito reale:** 10.956 candidati Jeju; a soglia 0.10 (El Hierro) **1.891 detection → SOVRA-rileva**
  (vs ~360 oreum letteratura). Specularmente a Frattini che a 0.64 **sotto-rilevava** (60). → **il
  punto operativo NON si trasferisce** (probabilità calibrate su El Hierro). Ricalibrando al conteggio
  (soglia 0.23 → 363 detection) i punti cadono sulle piane periferiche (oreum), lontano dall'Hallasan
  → **il ranking sì si trasferisce**. Fig `fig_zeroshot.png`.
- **Messaggio (coi due step Jeju):** la **rappresentazione generalizza** (Step 9, cluster 17.7°), la
  **calibrazione del classificatore no** (Step 9.5) → su un'isola senza etichette l'unsupervised è la
  via più sicura.
- **Test:** ✔ zero-shot mostrato; conferma che serve ricalibrare/etichettare per detection precise.

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
(classifica) → `7.5` (ablation: scegli l'input) → **`8`** (numeri che dimostrano la tesi) → `9`
(Jeju unsupervised).
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

---

## Conclusioni (dai dati ottenuti)

1. **Baseline riprodotta.** Il LoG dà recall **0.964** (= riferimento Frattini) ma con assegnamento
   one-to-one perde coni vicini → il **problema del merge** (il limite sollevato dalla prof) è reale e
   misurabile, non un'opinione.
2. **Il contributo funziona.** La pipeline finale (watershed + multiscala + RF) batte la baseline
   LoG+greedy su **ogni** metrica (precision ×4.1, F1 ×3.7, IoU migliore) e soprattutto separa
   **4/4 coppie adiacenti/annidate** vs 3/4 della baseline (adjacency gap 46→39). La separazione —
   non la classificazione — è il valore aggiunto.
3. **Classificatore debole-ma-reale, e onesto.** Sul pool unione 5+6 la RF ha OOF AP 0.13 (×3.6 sul
   prior); l'ablation (7.5) mostra che un pool pulito "solo-5" ordina molto meglio (AP 0.31) ma è
   l'unione a portare recall+separazione. Trade-off voluto: recall/separazione vs precisione, gestito
   dal matching uno-a-molti dello Step 8. Feature top = **texture + coerenza radiale** (la firma CV del
   cono), come da ipotesi.
4. **Generalizzazione (Jeju, no GT): la rappresentazione sì, il modello no.** Unsupervised (Step 9) un
   cluster isola la terra cono-like a **17.7° ≈ 18.8°** di El Hierro → le *feature* generalizzano.
   Zero-shot (Step 9.5) il classificatore alla soglia di El Hierro **sovra-rileva** (1891 vs ~360);
   il **ranking** si trasferisce ma il **punto operativo** no → su un'isola senza etichette
   l'unsupervised è la via più sicura.
5. **Vs Frattini.** Comparabili sul nucleo condiviso (stesso LoG, RF nello stesso ordine), io **avanti
   sulla separazione** (il limite che a lui mancava), **pari sul transfer** (zero-shot aggiunto).
   **Etna** non usata da nessuno (annotazioni sporche).
6. **Limiti.** Precisione ancora bassa (terreno vulcanico difficile); Jeju solo qualitativo (niente
   etichette per scorare); per detection precise cross-isola servirebbero più dati/etichette.
