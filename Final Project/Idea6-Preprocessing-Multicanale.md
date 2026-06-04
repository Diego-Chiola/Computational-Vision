# Idea 6 — Preprocessing/visualizzazione multicanale "CV-grounded"

> Approfondimento dell'idea 6 di [`Project-Ideas.md`](./Project-Ideas.md).
> Deck di riferimento: `01_image_processing.pdf` (gradiente, filtri orientabili, smoothing,
> scale-space). Dettaglio pagine in [`../slides/Slides-Index.md`](../slides/Slides-Index.md).

Risponde direttamente al **primo obiettivo** del `cinder_cones.pdf`: *"Computer vision work
to find new visualization and/or preprocessing modalities"*. Non è un detector: è il
**lavoro a monte** che rende i coni visibili e che alimenta tutte le altre idee.

## In una frase
Trasformare il DEM grezzo (quasi piatto e illeggibile) in una serie di **canali derivati**
che evidenziano la morfologia dei coni, usando gli strumenti di image processing del corso —
in particolare i **filtri orientabili (steerable)** per generare lo shading da qualsiasi
direzione di luce.

---

## 1. Su che dati si lavora

- Solo il **DEM** (Etna 2m, El Hierro 5m, Jeju 10m) → nessuno shapefile necessario, quindi
  vale su **tutti e tre** i dataset.
- Output: uno **stack multicanale** per ogni area, che diventa l'**input** delle idee 1, 2, 3,
  5. Questa idea è quindi anche la **fase di preprocessing condivisa** di tutto il progetto.

Motivazione (dal PDF): il DEM grezzo "vede" male i coni (Figura 5: quasi piatta), mentre i
geologi lavorano su dati derivati — shading, aspect, roughness, texture shading.

---

## 2. Cosa si fa: i canali da generare

| Canale | Come | Deck |
|---|---|---|
| **slope** (pendenza) | magnitudo del gradiente `√(gx²+gy²)` | `01`, p.~13 |
| **aspect** (direzione pendenza) | orientazione del gradiente `arctan(gy/gx)` | `01`, p.~13 |
| **roughness** (rugosità) | varianza/deviazione standard locale | — |
| **scale-space** | smoothing gaussiano a σ crescenti (piramide) | `01`, p.~37-40 |
| **hillshade multi-direzione** | **filtri derivativi orientabili (steerable `G₁^θ`)** | `01`, p.~22 |

Il pezzo forte e distintivo è l'**hillshade con i filtri orientabili**:
- il deck mostra che la derivata di gaussiana a un angolo θ qualsiasi si ottiene combinando
  due filtri base: `G₁^θ = cos(θ)·G₁⁰ + sin(θ)·G₁⁹⁰`
- questo permette di **sintetizzare lo shading da qualunque direzione di illuminazione**
  calcolando solo due filtri (invece di rifare l'hillshade da zero per ogni angolo)
- è la versione **fondata sulla computer vision** dello "shading" che i geologi usano a mano,
  e che nei dati è fornito solo per NE/NW (render Eduard)

---

## 3. Quali strumenti

- `numpy`/`scipy.ndimage` per gradiente, filtri gaussiani e loro derivate
- `rasterio` per leggere/scrivere i raster mantenendo la georeferenziazione
- (confronto) `richdem`/`gdaldem` producono slope/aspect "standard" → utili come riferimento
  per validare i tuoi canali fatti a mano
- nessuna GPU

---

## 4. Come si valuta

Non avendo un output "detection", la valutazione è diversa:
- **qualitativa**: confronto visivo dei canali, mostrando che i coni risaltano molto più che
  nel DEM grezzo (figure prima/dopo)
- **quantitativa indiretta**: misuri quanto **migliorano le altre idee** quando aggiungi un
  canale. Esempio: precision/recall dell'idea 2 con solo DEM vs DEM+slope vs DEM+slope+shading
  → un **ablation study** che dimostra il valore del preprocessing

---

## 5. Cosa aspettarsi

- È la base **indispensabile** per tutto il resto: conviene farla per prima, perché serve a
  ogni altra idea.
- Da sola **non "trova" i coni** → all'orale va presentata come contributo di
  preprocessing/visualizzazione (cosa che la Consegna chiede esplicitamente), non come
  detector.
- Il contributo originale e difendibile è il **multi-direction hillshade via steerable
  filters**: lega un metodo del corso (filtri orientabili) a un'esigenza reale dei geologi.
- Veloce, su CPU, nessuna etichetta richiesta.

---

## Dove si colloca
È la **fondazione condivisa** del progetto: produce i canali che alimentano le idee 1
(input multicanale del modello), 2 (mappe di slope/roughness), 3 (canali delle patch) e 5
(feature dei superpixel). Nella relazione è il capitolo "preprocessing", con l'ablation study
come prova del suo valore.
