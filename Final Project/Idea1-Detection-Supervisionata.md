# Idea 1 — Detection supervisionata di coni

> Approfondimento dell'idea 1 di [`Project-Ideas.md`](./Project-Ideas.md).
> Deck di riferimento: `RCNN.pdf`, `semantic_segmentation.pdf`, `06_image_representations_intro.pdf`
> (dettaglio pagine in [`../slides/Slides-Index.md`](../slides/Slides-Index.md)).

## In una frase
Insegnare a un modello a **trovare automaticamente i coni** in una mappa di elevazione,
mostrandogli molti esempi già etichettati a mano dai geologi (gli shapefile). Il modello
impara la "firma" di un cono e poi la cerca da solo su terreni mai visti.

---

## 1. Che dati abbiamo ORA

Due tipi di dato, da incrociare.

**A) Il DEM (Digital Elevation Model)** — raster in cui ogni pixel **non è un colore ma
un'altitudine in metri**.
- Etna: `DEM 2 m Etna-001.tif` → 1 pixel = 2 m sul terreno
- El Hierro: `dem5m` → 1 pixel = 5 m
- Jeju: `Raster Jeju 5k.tif` → 1 pixel = 10 m

È un'immagine a **un solo canale** (grayscale), float, georeferenziata (i file `.prj`/`.tfw`
dicono *dove* sta sulla Terra ogni pixel).

**B) Gli shapefile (la ground truth)** — file vettoriali con i poligoni disegnati a mano
dai geologi:
- El Hierro `Cones.shp`: ~180 coni (basi + crateri + fissure), qualità medio-alta
- Etna `coni_monogenici.shp` + `crateri.shp`: ~220 coni, qualità bassa (errori, imprecisioni)
- Jeju: **niente** → non usabile per il supervisionato, solo test qualitativo finale

Punto chiave: lo shapefile dice **dove sono i coni** = sono le **risposte giuste** con cui
si addestra e si valuta il modello.

---

## 2. Da cosa si parte: preparazione del dato (la parte più lunga)

Fase richiesta esplicitamente dalla Consegna ("initial organization and preprocessing").

**a) Allineare raster e shapefile.** Con `rasterio` (DEM) e `geopandas` (shapefile).
Convertire i poligoni geografici in **coordinate-pixel** sul DEM → per ogni cono sai *in
quali pixel* cade.

**b) Costruire i canali di input (multicanale).** Il DEM grezzo "vede" male i coni
(Figura 5 del PDF: quasi piatta). Si derivano canali che evidenziano la morfologia:
- **slope** (pendenza) → fianchi a 25–35°
- **aspect** (direzione della pendenza) → su un cono punta radialmente verso l'esterno
- **hillshade NE / NW** (shading direzionale) → già forniti dai render Eduard, o ricalcolabili
- **texture shading** → già fornito

Lo stack diventa un'immagine a più canali (come una foto RGB ha 3 canali, qui 5-6).
È **l'input vero** del modello.

**c) Tagliare in tile.** Il DEM Etna è ~2.86 GB: si taglia in finestre (es. 512×512 px)
con un po' di sovrapposizione.

**d) Creare le etichette nel formato del modello:**
- **detection**: per ogni tile, la lista dei *bounding box* attorno ai coni
- **segmentazione**: una maschera dove ogni pixel è `0=sfondo / 1=base / 2=cratere`

---

## 3. Cosa si fa: l'esperimento

Idea forte: il **transfer cross-vulcano**.

```
TRAIN  →  El Hierro (GT pulita)
TEST   →  Etna (GT rumorosa)  +  Jeju (solo visivo, nessuna GT)
```

Si addestra dove le etichette sono buone (El Hierro) e si verifica se il modello
**generalizza** su un vulcano diverso, con risoluzione e morfologia diverse (Etna).
Jeju serve come demo qualitativa.

> Con ~180 esempi (pochi per il deep learning) servono **data augmentation** (rotazioni,
> flip — un cono ruotato è ancora un cono) e probabilmente un modello **pre-addestrato**
> (transfer learning da ImageNet).

---

## 4. Quali modelli — due strade

### Strada A — Object detection (bounding box)
Il modello produce **rettangoli** "qui c'è un cono" + un punteggio di confidenza.
- **Modello: Faster R-CNN** (deck `RCNN.pdf`).
  - Nota: nella v1 di `Project-Ideas.md` era indicato YOLO; corretto in Faster R-CNN
    perché **è quello trattato a lezione** → più difendibile all'orale.
- Pro: output semplice (una scatola per cono), valutazione standard.
- Contro: il box non segue la forma esatta del cono.

### Strada B — Segmentazione semantica (pixel per pixel)
Il modello etichetta **ogni pixel**: base / cratere / sfondo.
- **Modello: U-Net o FCN** (deck `semantic_segmentation.pdf`).
- Pro: contorni precisi, sfrutta proprio i poligoni dello shapefile; gestisce bene coni
  sovrapposti.
- Contro: per "contare i coni" serve un passo extra (connected components sulla maschera).

Per un progetto da 1 CFU: farne **una sola**, fatta bene. La **U-Net** è spesso la scelta
migliore qui (GT = poligoni ≈ maschere; forme irregolari che un box rappresenta male).

---

## 5. Come si misura se funziona (valutazione)

Metriche **dal deck `RCNN.pdf`** (allineate al programma):
- **IoU / Jaccard**: sovrapposizione tra predizione e ground truth
- predizione **TP** se IoU ≥ 0.5, altrimenti **FP**; cono vero non trovato = **FN**
- **Precision** (detection giuste / totali), **Recall** (coni veri trovati), **F1**

Narrativa del progetto: su **El Hierro** (test interno) P/R buoni; su **Etna** caleranno —
e si **spiega perché** (GT rumorosa, 2m vs 5m, morfologia diversa). Il "fallimento
spiegato" vale molto all'orale.

---

## 6. Cosa aspettarsi (realisticamente)

- La fase **1-2 (preprocessing/allineamento)** prende il **60-70% del tempo**. È normale
  e richiesta dalla Consegna.
- Con ~180 coni il deep learning è **al limite del poco-dato**: risultati decenti ma non da
  paper; augmentation + pretraining sono necessari, non opzionali.
- Su Etna il modello **sbaglierà di più** — è ok, fa parte del racconto.
- Strumenti: Python, `rasterio` + `geopandas` (dati geo), `numpy`, `PyTorch` (modello),
  `torchvision` (Faster R-CNN pronto) o `segmentation-models-pytorch` (U-Net).

---

## Relazione con le altre idee
È la più "ML supervisionato classico". L'**idea 2** (CV classica) richiede meno dati/GPU
ma più ingegneria a mano; l'**idea 3** (classificazione patch positive/negative) è la più
semplice da far partire ed è il ponte naturale verso l'idea 1.
