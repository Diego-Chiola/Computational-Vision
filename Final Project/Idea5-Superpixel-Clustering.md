# Idea 5 — Superpixel + clustering (SLIC + K-means)

> Approfondimento dell'idea 5 di [`Project-Ideas.md`](./Project-Ideas.md).
> Deck di riferimento: `SLIC_Superpixels.pdf`, `09_segmentation.pdf`
> (dettaglio pagine in [`../slides/Slides-Index.md`](../slides/Slides-Index.md)).

Idea **unsupervised**: niente etichette per addestrare. Si raggruppa il terreno in regioni
omogenee e si lascia che i coni "emergano" come cluster con caratteristiche proprie.

## In una frase
Spezzettare la mappa in tante **regioni piccole e omogenee (superpixel)** con SLIC, descrivere
ogni regione con poche feature morfologiche, e poi **raggruppare** le regioni simili con
K-means — sperando che i coni finiscano in cluster distinti dal terreno di sfondo.

---

## 1. Su che dati si lavora

- Funziona su **qualsiasi DEM**, anche senza shapefile → è l'idea ideale per **Jeju**
  (che non ha ground truth) e per il task unsupervised richiesto dalla Consegna.
- Input: il DEM e i suoi canali derivati (slope, roughness, shading). SLIC lavora bene anche
  su immagini **a un solo canale** (grayscale), come nota il paper.
- Shapefile (El Hierro/Etna): usato **solo a posteriori** per dare una valutazione numerica,
  non per guidare il metodo.

---

## 2. Cosa si fa

```
DEM (+ slope/roughness/shading)
 │
 ├─(a) SLIC          → divide la mappa in ~migliaia di superpixel    [SLIC_Superpixels]
 │                      compatti e aderenti ai bordi (clustering 5D lab+xy)
 │
 ├─(b) feature per superpixel → per ogni regione calcoli:
 │      slope medio, roughness, varianza, elevazione relativa, forma...
 │
 ├─(c) K-means        → raggruppa i superpixel in K cluster          [09_segmentation, p.~28-31]
 │      (es. "fianco di cono", "cratere", "pianura", "valle")
 │
 ├─(d) scelta di K    → indice di Davies-Bouldin                     [09_segmentation, p.~31]
 │      provi più K e tieni quello con cluster più netti
 │
 └─(e) interpretazione → identifichi quale/i cluster corrispondono ai coni
       e (opzionale) raggruppi superpixel adiacenti dello stesso cluster
```

Punto concettuale: SLIC riduce **milioni di pixel** a **poche migliaia di superpixel**,
rendendo il clustering veloce e i descrittori più stabili (un superpixel ha già una sua
statistica interna, meno rumorosa del singolo pixel).

---

## 3. Quali strumenti

- `scikit-image` → `slic()` per i superpixel e `regionprops()` per le feature
- `scikit-learn` → `KMeans`, `davies_bouldin_score`
- `rasterio`/`numpy` per il DEM e i canali
- nessuna GPU, tutto su CPU

---

## 4. Come si valuta

Essendo unsupervised, due livelli:
- **interno** (senza ground truth): qualità dei cluster con **Davies-Bouldin** → vale anche
  su Jeju
- **esterno** (dove c'è lo shapefile): confronti la maschera "cluster-cono" con i poligoni
  veri di El Hierro → IoU / precision / recall, come nelle altre idee

---

## 5. Cosa aspettarsi

- **Nessuna etichetta richiesta** → unico modo per produrre risultati quantificabili anche su
  Jeju; copre il task unsupervised della Consegna.
- Il rischio tipico del clustering: i cluster potrebbero **non coincidere** con la semantica
  "cono/non-cono" (K-means raggruppa per somiglianza di feature, non per concetto). Spesso
  serve scegliere bene le feature e magari unire più cluster.
- La scelta di **K** è delicata: Davies-Bouldin aiuta ma non risolve del tutto (lo dice anche
  la slide).
- Ottima come **terza gamba** del progetto accanto a una pipeline classica (idea 2) e/o a un
  modello supervisionato (idea 1/3): mostra l'approccio non supervisionato e la **generalità**
  su un dataset senza GT.

---

## Dove si colloca
È l'approccio **region-based / unsupervised** del progetto. Si combina naturalmente con
l'idea 2 (entrambe "classiche", senza training) e copre l'unico scenario in cui le altre idee
supervisionate non possono dare numeri: **Jeju**.
