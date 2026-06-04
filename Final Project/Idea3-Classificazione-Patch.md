# Idea 3 — Classificazione patch positive/negative

> Approfondimento dell'idea 3 di [`Project-Ideas.md`](./Project-Ideas.md).
> Deck di riferimento: `06_image_representations_intro.pdf`, `09_segmentation.pdf`, `RCNN.pdf`
> (dettaglio pagine in [`../slides/Slides-Index.md`](../slides/Slides-Index.md)).

L'idea **più semplice da far partire** e, insieme, la **più allineata in assoluto** sia
alla Consegna sia alle slide. La Consegna la suggerisce testualmente: *"preparing a dataset
of positive / negative examples using ground truth bounding boxes to extract the positive
examples"*.

## In una frase
Invece di cercare i coni in tutta la mappa (detection), si riduce il problema a una domanda
binaria su **ritagli (patch) di immagine**: *"questo quadratino contiene un cono — sì o
no?"*. Si costruisce un dataset di esempi positivi e negativi e si addestra un classificatore
a distinguerli.

---

## 1. Su che dati si lavora

Stessi dati di base (DEM + shapefile), trasformati in un **dataset di patch**:

**Patch POSITIVE** → ritagli centrati sui coni veri, usando i poligoni dello shapefile come
guida. Da ~180 coni di El Hierro ricavi ~180 patch positive (poi moltiplicate con
augmentation).

**Patch NEGATIVE** → ritagli da zone **senza coni** (terreno piatto, valli, pendii lineari
non vulcanici). Generabili a piacere campionando il resto del DEM → quante ne vuoi.

Ogni patch è un'immagine (mono- o multicanale: DEM + slope + shading, come nell'idea 1), con
etichetta `1=cono / 0=non-cono`. È esattamente la struttura `(x_i, y_i)` del deck
`06_image_representations` con l'esempio **face / not-face** (p.~4) e la **skin segmentation**
di `09_segmentation` (p.~25-27).

> Differenza chiave dall'idea 1: lì il modello deve dire *dove* sono i coni (localizzazione);
> qui deve solo dire *se* una patch già ritagliata contiene un cono (classificazione).
> Problema più ristretto = più facile.

---

## 2. Come si costruisce il dataset (la parte centrale)

È il cuore dell'idea e ciò che la Consegna premia:
1. allinea shapefile↔DEM (come idea 1)
2. per ogni cono, ritaglia una finestra attorno al centro → **positivo**
3. campiona finestre lontane dai coni → **negativo**
4. bilancia le classi (≈ stesso numero di positivi e negativi)
5. **data augmentation** sui positivi (rotazioni, flip) per compensare i pochi esempi
6. split train/validation/test (es. allena su El Hierro, testa su Etna → transfer
   cross-vulcano)

Decisioni di design da motivare nella relazione: **dimensione della patch** (deve contenere
un cono intero ma non troppo contorno), **canali** da usare, **rapporto positivi/negativi**.

---

## 3. Quali modelli — due livelli

Il deck `06_image_representations` mostra la progressione storica, presentabile come baseline
crescenti:

**Livello A — feature handcrafted + classificatore classico**
- rappresenta ogni patch con **istogrammi di gradiente / BoW (bag-of-keypoints)** o feature
  semplici (media/varianza di slope, simmetria radiale…) → deck `06`, p.~9-11
- classificatore: **SVM** (anche Random Forest)
- Pro: poco dato, niente GPU, molto interpretabile.

**Livello B — feature CNN**
- una **piccola CNN** che impara le feature da sola (deck `06`, p.~13-14), o transfer learning
  da una rete pre-addestrata usata come estrattore di feature + SVM sopra
- Pro: di solito più accurato; Contro: serve più dato/augmentation e preferibilmente GPU.

Per 1 CFU: **Livello A come baseline** e, se c'è tempo, Livello B come confronto. Il confronto
"feature a mano vs feature imparate" è già un buon racconto (messaggio del deck 06: *"from
features engineering to features learning"*).

---

## 4. Come si valuta

Classificazione binaria → metriche dirette (anche dal deck `RCNN`, p.~6-9):
- **accuracy**, **precision**, **recall**, **F1**
- **confusion matrix** (quanti coni scambiati per non-coni e viceversa)

Nota onesta: queste metriche valutano la **classificazione di patch**, non la detection
sull'intera mappa. Sono cose diverse (vedi sotto).

---

## 5. Cosa aspettarsi (realisticamente)

- **La più rapida da mettere in piedi** e con meno rischi: anche un SVM su feature semplici dà
  subito numeri sensati.
- Ottima come **componente** di un sistema più grande: un classificatore di patch può diventare
  il motore di un detector a **sliding window** (finestra che scorre sulla mappa, classifichi
  ogni posizione — idea base di detection in `RCNN`, p.~11). Così l'idea 3 si **trasforma**
  nell'idea 1, ma con strumenti più semplici e tutti del corso.
- Limite da dichiarare: la classificazione di patch da sola **non localizza** e non gestisce
  bene coni di scala/posizione variabile finché non la inserisci nello sliding window.
- Strumenti: `numpy`, `scikit-learn` (SVM, Random Forest, metriche), `scikit-image`/`opencv`
  per le feature, opzionale `PyTorch` per la CNN.

---

## Dove si colloca rispetto a 1 e 2

L'idea 3 è il **ponte** tra le altre due:
- condivide con l'**idea 1** la natura supervisionata (impara da esempi) ma è molto più
  leggera;
- condivide con l'**idea 2** la possibilità di restare "classica" (feature a mano + SVM, niente
  deep learning obbligatorio);
- messa dentro uno **sliding window**, diventa un detector completo → percorso più graduale per
  arrivare comunque a "trovare i coni sulla mappa".

Spesso è la scelta migliore per **iniziare**: costruisci il dataset positive/negative (lavoro
che serve comunque anche all'idea 1), ottieni subito un classificatore funzionante, e poi
decidi se evolverlo verso detection (idea 1) o confrontarlo con la pipeline classica (idea 2).
