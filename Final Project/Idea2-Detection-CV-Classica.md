# Idea 2 — Detection con Computer Vision classica

> Approfondimento dell'idea 2 di [`Project-Ideas.md`](./Project-Ideas.md).
> Deck di riferimento: `01_image_processing.pdf`, `09_segmentation.pdf`, `03_image_matching.pdf`
> (dettaglio pagine in [`../slides/Slides-Index.md`](../slides/Slides-Index.md)).

L'opposto filosofico dell'idea 1: **niente training, niente rete neurale**. Si sfrutta il
fatto che un cono ha una "firma" geometrica così forte e prevedibile da poterla descrivere
a mano con regole, usando gli strumenti del corso (gradiente, soglie, blob detection).

## In una frase
Costruire una **pipeline algoritmica** che, dato il DEM, evidenzia le proprietà
morfologiche del cono (pendenza, rugosità, depressione del cratere) e poi estrae i
candidati con soglie e analisi di forma — **senza imparare da esempi**.

---

## 1. Su che dati si lavora

Stessi dati dell'idea 1, ma con un ruolo **diverso** per gli shapefile:
- **DEM** (Etna 2m, El Hierro 5m, Jeju 10m) → l'input vero su cui gira la pipeline
- **Shapefile** → qui **NON** servono per addestrare (non c'è training). Servono **solo alla
  fine, per validare**: confronti i coni trovati con quelli veri e calcoli precision/recall.

Conseguenza pratica: questa idea **funziona anche su Jeju** (che non ha shapefile), perché
non ha bisogno di etichette per girare — al massimo Jeju resta senza valutazione numerica,
ma la pipeline produce comunque risultati.

---

## 2. La "firma" morfologica del cono (su cosa ci si basa)

Dal `cinder_cones.pdf`, un cono ha proprietà misurabili:
- **fianchi ripidi** → slope 25–35°
- **superficie liscia** → roughness (rugosità) bassa, più liscia del terreno circostante
- **aspect radiale** → la pendenza "punta" verso l'esterno tutt'attorno al centro
- **cratere = depressione locale** al vertice → un buco circolare

Si trasforma ognuna di queste proprietà in un **canale-immagine** e poi si combinano.

---

## 3. La pipeline (passo per passo)

```
DEM
 │
 ├─(a) smoothing gaussiano         → toglie il rumore        [01_image_processing]
 │
 ├─(b) slope = |gradiente|          → evidenzia i fianchi      [01_image_processing, p.~13]
 ├─(b) aspect = orientazione grad.  → pattern radiale
 ├─(b) roughness = varianza locale  → i coni sono lisci
 │
 ├─(c) soglia (Otsu / adattiva)     → maschera binaria         [09_segmentation, p.~15-22]
 │     "questi pixel hanno la pendenza giusta"
 │
 ├─(d) connected components         → raggruppa i pixel in blob [09_segmentation, p.~13]
 │     ogni blob = un candidato cono
 │
 ├─(e) filtro per forma             → tieni solo blob con area
 │     (area, circolarità, ...)       e circolarità plausibili
 │
 └─(f) blob detection LoG/DoG       → trova i crateri circolari [03_image_matching, p.~14-16]
       multi-scala                    a scale diverse
```

Punti chiave (e cosa è cambiato rispetto alla v1):
- **(b) slope/aspect/roughness** = magnitudo/orientazione del gradiente e varianza locale →
  tutto da `01_image_processing`.
- **(c)+(d) soglia + connected components** = modo "da manuale" per estrarre blob da una
  mappa → da `09_segmentation`.
- **(f) LoG/DoG (Laplacian/Difference of Gaussian)** = **blob detector scale-invariant** del
  deck matching: trova macchie circolari a diverse dimensioni → perfetto per crateri di
  raggio variabile.
- ⚠️ **Hough declassato**: nella v1 si puntava sulla *Hough circolare* per i crateri. La
  trasformata di Hough **non è nelle slide del corso**, quindi è stata tolta dal cuore della
  pipeline e lasciata come extra opzionale. Al suo posto si usa LoG/DoG (insegnato, stesso
  scopo: trovare cerchi/blob).

---

## 4. Quali "modelli"

Non ci sono modelli da addestrare — i "modelli" sono **algoritmi con parametri da tarare a
mano**:
- soglia di slope (es. 20°), dimensione del kernel gaussiano (σ), soglie di area/circolarità,
  scale del LoG/DoG.
- Strumenti: `numpy`, `scipy.ndimage` (gradiente, connected components, filtri),
  `scikit-image` (`blob_log`, `blob_dog`, `regionprops`), `rasterio`/`geopandas` per dati e
  validazione.
- Tutto su **CPU**, in pochi secondi/minuti — nessuna GPU.

---

## 5. Come si valuta

Identica all'idea 1 (deck `RCNN`): i coni trovati si confrontano con lo shapefile di
El Hierro via **IoU**, e si calcolano **precision / recall / F1**. Si può tracciare una
**curva** facendo variare la soglia di slope (più bassa → più recall ma più falsi positivi).

---

## 6. Cosa aspettarsi (realisticamente)

- **Parte subito**, senza dati di training e senza GPU → ottimo per risultati rapidi.
- Cattura facilmente i **coni "perfetti"** e isolati; fatica sui casi difficili del PDF:
  coni **parziali**, **sovrapposti**, **erosi** (slope irregolare).
- Tantissimo **tuning manuale** delle soglie → prezzo da pagare al posto del training. Una
  soglia buona su El Hierro (5m) può non funzionare su Etna (2m): proprio un risultato da
  discutere.
- Narrativa forte all'orale: "**metodi classici fanno l'80% del lavoro senza una sola
  etichetta di training**" → poi confronto con l'idea 1 per mostrare cosa si guadagna col
  supervisionato.

---

## Idea 1 vs Idea 2 in breve

| | Idea 1 (supervisionata) | Idea 2 (CV classica) |
|---|---|---|
| Training | sì, su El Hierro | no |
| Serve GPU | sì (consigliata) | no |
| Serve ground truth | per train **e** test | solo per test |
| Gira su Jeju (no GT) | solo predizione | sì, anche senza valutazione |
| Sforzo principale | preprocessing + training | tuning soglie + design pipeline |
| Robustezza coni difficili | migliore (se ha dati) | più fragile |
| Deck del corso | RCNN, semantic_seg, image_repr | image_processing, segmentation, matching |

Le due idee sono il **confronto perfetto** da mettere nello stesso progetto: pipeline
classica come baseline, deep learning come "vediamo se vale la pena". È la raccomandazione
finale di `Project-Ideas.md`.
