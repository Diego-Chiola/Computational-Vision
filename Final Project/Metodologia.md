# Metodologia del progetto — pilastri obbligatori

> Documento trasversale a tutte le idee (`Idea1`…`Idea6`, `Project-Ideas.md`).
> Definisce **cosa va assolutamente messo nella relazione** e in che ordine, a prescindere
> dall'idea finale scelta. Ogni capitolo del progetto deve poter rispondere a queste quattro
> domande:
>
> 1. **Da quali assunzioni partiamo?**
> 2. **Qual è l'obiettivo preciso?** (cosa entra, cosa esce, come si misura il successo)
> 3. **Come è fatto il dato e quale preprocessing serve** — deciso *osservando* i dataset?
> 4. **Perché ogni scelta?** — con dimensionalità, grandezza delle matrici e *cosa si ottiene
>    dopo ogni passaggio*.
>
> ⚠️ I valori numerici contrassegnati con `‹da misurare›` vanno riempiti dopo aver scaricato
> il dataset e letto i metadati dei raster (vedi §6 — come misurarli).

---

## 1. Assunzioni iniziali

Le ipotesi che diamo per vere e su cui poggia tutta la pipeline. Vanno **dichiarate
esplicitamente** perché determinano i limiti di validità dei risultati.

### Sui dati
- Ogni DEM è una **immagine a singolo canale** in cui il valore del pixel è una **quota in
  metri** (range/depth image — cfr. `04_3Dvision_small`). Non è un'immagine RGB: niente colore,
  solo elevazione.
- La **georeferenziazione** (`.prj`, `.tfw`/`.tiff` tags) è corretta e permette di allineare
  raster ↔ shapefile nello stesso sistema di coordinate. **Senza questa assunzione, la
  ground truth non è sovrapponibile al DEM.**
- La **risoluzione spaziale** è nota e costante per dataset: Etna 2 m/px, El Hierro 5 m/px,
  Jeju 10 m/px. Un pixel = quell'area reale a terra. → tutte le soglie metriche (raggio
  cono, area cratere) vanno convertite px↔metri usando questo fattore.
- I valori di **no-data** (pixel fuori dominio / mare) esistono e vanno mascherati prima di
  qualsiasi statistica, altrimenti inquinano gradienti e soglie.

### Sul fenomeno (la "firma" del cono — da `cinder_cones.pdf`)
- Un cono di scoria ha morfologia **prevedibile**: fianchi ripidi (slope 25–35°), superficie
  relativamente liscia (roughness bassa), simmetria (aspect radiale), spesso un **cratere =
  depressione locale** al vertice.
- I coni hanno **scala variabile** (raggi diversi) → serve un approccio multi-scala, non una
  taglia fissa.
- La ground truth è **imperfetta**: Etna ha falsi positivi/negativi e segmentazioni sporche;
  El Hierro è migliore ma mescola basi, crateri e fissure nello stesso shapefile; Jeju non ha
  ground truth. → assunzione operativa: **El Hierro = riferimento per train/validazione**,
  Etna = stress test, Jeju = demo qualitativa/unsupervised.

### Sulle risorse
- Etna DEM (~2.86 GB) **non entra in RAM** → si lavora a **tile/finestre**, non sull'immagine
  intera. Questa assunzione vincola tutta l'architettura del codice (vedi §3 e §4).

---

## 2. Obiettivo

Cosa il progetto **vuole ottenere**, formulato in modo verificabile. Va scelto **uno**
obiettivo primario e dichiarato in termini di input → output → metrica.

### Obiettivo primario (proposto)
> **Dato un DEM, individuare automaticamente i coni di scoria** e validare la qualità del
> rilevamento contro la ground truth di El Hierro.

- **Input**: stack multicanale derivato dal DEM (§3).
- **Output**: insieme di rilevamenti — bounding box o maschera per ogni cono candidato.
- **Metrica di successo** (da `RCNN`): **IoU/Jaccard** tra predetto e ground truth, e da lì
  **precision / recall / F1**; opzionale **confusion matrix** e curva P/R al variare della
  soglia.

### Obiettivi secondari ammessi (a seconda dell'idea)
- **Classificazione** patch positive/negative (Idea 3): output = etichetta cono/non-cono per
  patch → metrica = accuracy/precision/recall sulla patch.
- **Clustering/unsupervised** su Jeju (Idea 5): output = segmentazione in regioni → metrica =
  Davies-Bouldin + valutazione qualitativa (niente ground truth).
- **Preprocessing/visualizzazione** (Idea 6): output = canali derivati → metrica =
  **ablation study** (quanto migliora la detection aggiungendo ciascun canale).

### Criterio di scelta dell'obiettivo
Dichiarare *perché* quell'obiettivo: copertura del programma del corso, disponibilità di
ground truth, fattibilità senza GPU. (La raccomandazione di `Project-Ideas.md` combina
preprocessing → detection classica → baseline DL → valutazione.)

---

## 3. Preprocessing — deciso osservando i dataset

**Regola metodologica:** ogni passo di preprocessing nasce da un'**osservazione** sul dato,
non da un'abitudine. Per ogni passo va scritto: *cosa ho osservato → quale problema crea →
quale trasformazione lo risolve*.

### 3.0 Osservazione preliminare (da fare per prima, sul dato scaricato)
Prima di scrivere codice di trasformazione, **ispezionare** ogni DEM:
- dimensione `H×W`, dtype, risoluzione, presenza di no-data, istogramma delle quote (min/max,
  outlier), quanto i coni sono *visibili* nel DEM grezzo.
- **Atteso** (da `cinder_cones.pdf`, Fig. 5): il DEM grezzo è quasi piatto e i coni si vedono
  male → **giustifica** la necessità dei canali derivati (§3.2). Questa è l'osservazione che
  motiva tutto il preprocessing.

### 3.1 Pulizia e organizzazione
| Passo | Osservazione che lo motiva | Trasformazione |
|---|---|---|
| Mascheratura no-data | Pixel mare/fuori-dominio falsano statistiche | sostituzione con `NaN`/maschera booleana |
| Normalizzazione quote | Le quote assolute variano tra dataset (Etna vs El Hierro) | z-score o min-max **per-tile**, non globale |
| Tiling con overlap | Etna ~2.86 GB non entra in RAM | finestre `512×512` (o simili) con bordo di overlap per non tagliare i coni |
| Allineamento shapefile↔raster | Servono per validare/etichettare | reproiezione `geopandas` nel CRS del raster, poi rasterizzazione poligoni → maschera |

### 3.2 Generazione canali derivati (cuore — Idea 6, deck `01_image_processing`)
Dal DEM single-channel si producono i canali che rendono i coni visibili:

| Canale | Come si calcola | Cosa evidenzia |
|---|---|---|
| **slope** | magnitudo gradiente `√(g_x²+g_y²)` | fianchi ripidi del cono |
| **aspect** | orientazione gradiente `atan2(g_y,g_x)` | simmetria radiale |
| **roughness** | dev. standard locale (finestra k×k) | superficie liscia vs terreno |
| **hillshade(θ)** | filtri **steerable** `G₁^θ = cosθ·G₁⁰ + sinθ·G₁⁹⁰` | shading da qualsiasi luce |
| **scale-space** | smoothing gaussiano a σ crescenti | coni a scale diverse |

→ output: uno **stack multicanale** che diventa l'input di tutte le idee a valle.

### 3.3 Perché *osservando* e non a priori
La scelta dei parametri (σ del gaussiano, finestra della roughness, soglia di slope) va
**tarata sull'istogramma reale** del dataset osservato in §3.0, e può **differire tra Etna
(2 m) ed El Hierro (5 m)**: stessa soglia in gradi ma diversa in pixel. Questo va documentato
come risultato, non nascosto.

---

## 4. Perché ogni scelta — dimensionalità e cosa si ottiene dopo ogni passaggio

**Regola metodologica:** per ogni trasformazione la relazione deve riportare **(a)** la forma
della matrice in ingresso, **(b)** la forma in uscita, **(c)** il significato del contenuto,
**(d)** perché quella scelta. Di seguito lo schema-tipo (forme simboliche; i valori
`‹da misurare›` si riempiono dai metadati reali — §6).

```
Notazione:  H, W = altezza/larghezza in pixel del DEM   |  R = risoluzione (m/px)
            C = numero canali   |  N = numero tile   |  P = lato patch   |  K = n. blob
```

| # | Passaggio | Ingresso (forma) | Uscita (forma) | Cosa si ottiene / perché |
|---|---|---|---|---|
| 1 | DEM grezzo | — | `(H, W)` float32 | matrice di quote; 1 canale. `H,W` = ‹da misurare› |
| 2 | Tiling (overlap) | `(H, W)` | `N × (h, w)`, es. `h=w=512` | blocchi gestibili in RAM; `N ≈ (H·W)/(h·w)` |
| 3 | Normalizzazione | `(h, w)` | `(h, w)` float | stesso shape, range portato a ~[0,1] o media 0: confrontabilità tra aree |
| 4 | slope/aspect/rough. | `(h, w)` | 3 × `(h, w)` | da 1 a 3 mappe: ogni pixel ora descritto da più proprietà morfologiche |
| 5 | hillshade multi-dir | `(h, w)` | `D × (h, w)` | `D` direzioni di luce dai 2 filtri base (steerable) |
| 6 | **stack canali** | varie `(h,w)` | `(C, h, w)`, es. `C=6` | tensore multicanale = input dei modelli. `C` = DEM+slope+aspect+NE+NW+texture |
| 7a | soglia (Otsu) → maschera | `(h, w)` slope | `(h, w)` bool | pixel "candidati cono"; riduce a binario |
| 7b | connected components | `(h, w)` bool | label map `(h, w)` int + `K` blob | da pixel a **oggetti**: `K` candidati distinti |
| 7c | regionprops + filtro forma | `K` blob | `K' ≤ K` blob (area, circolarità…) | scarta blob implausibili; ogni blob → riga di feature |
| 7d | blob LoG/DoG (crateri) | `(h, w)` | lista `(y, x, σ)` × `M` | crateri come cerchi a scala `σ` variabile |
| 8 | **(patch)** estrazione | `(C, H, W)` + GT | `(N_p, C, P, P)` | dataset di patch positive/negative per classificazione (Idea 3). `P` es. 64 |
| 9 | **(DL)** modello | `(C, h, w)` | box/maschera + score | detection/segmentazione (Idea 1) |
| 10 | valutazione | predetti + GT | scalari: IoU, P, R, F1 | numeri di qualità; matrice di confusione |

### Come leggere la tabella nella relazione
- **Ogni riga è un "prima → dopo"**: si mostra la forma e, idealmente, una **figura** del
  contenuto (es. la mappa di slope, la maschera, i blob colorati) per far vedere *cosa è
  comparso* dopo il passaggio.
- I **cambi di dimensionalità chiave** da commentare:
  - 1→2: spaziale `H×W → N` tile (vincolo di memoria, §1).
  - 4→6: **canali 1 → C**: arricchimento della descrizione per-pixel (il punto dell'Idea 6).
  - 7a→7b: **pixel → oggetti** (`K` blob): è qui che "nasce" il rilevamento.
  - 7c/8: **immagine → tabella di feature** `(K', F)` o `(N_p, C, P, P)`: passaggio da dominio
    immagine a dominio "esempi" per classificatore.

---

## 4-bis. Ambiente di esecuzione (CPU locale vs Colab GPU)
- **Preprocessing + CV classica** (Idee 2, 5, 6): girano su **CPU in locale**, in
  secondi/minuti, senza GPU → notebook/script locali.
- **Baseline deep learning** (Idea 1: Faster R-CNN / U-Net, e feature CNN dell'Idea 3):
  richiedono **GPU** → si scrive un **notebook Colab (`.ipynb`)** dedicato, con il dataset
  caricato da Drive (o ri-scaricato), in modo che la parte pesante non dipenda dalla macchina
  locale. Il notebook va tenuto **separato** dalla parte classica e documentato come tale
  (runtime GPU, versioni librerie, tempi di training).

---

## 5. Checklist da spuntare per ogni esperimento
- [ ] Assunzioni dichiarate (dati, fenomeno, risorse).
- [ ] Obiettivo formulato come input → output → metrica.
- [ ] Osservazione del dato scritta **prima** del preprocessing (§3.0).
- [ ] Ogni passo di preprocessing motivato da un'osservazione.
- [ ] Per ogni passaggio: forma ingresso, forma uscita, significato, perché.
- [ ] Figura "prima/dopo" dove ha senso.
- [ ] Differenze di parametri tra Etna/El Hierro/Jeju documentate.
- [ ] Metrica calcolata su El Hierro; Etna come stress test; Jeju come qualitativo.

---

## 6. Come misurare i valori `‹da misurare›` (una volta scaricato il dataset)
Da fare appena `Final Project/CinderConesDataPack/` è presente:
- **forma/dtype/risoluzione/no-data** di ogni DEM → leggere i metadati con `rasterio`
  (`dataset.shape`, `dataset.dtypes`, `dataset.res`, `dataset.nodata`, `dataset.crs`).
- **peso in RAM** di una matrice = `H · W · bytes_per_pixel` (float32 = 4 byte) → giustifica
  numericamente la scelta del tiling.
- **numero di feature** negli shapefile → `geopandas.read_file(...).shape` e conteggio per
  campo `tipo` (separare basi/crateri/fissure su El Hierro).
- istogramma quote e slope → fissare le soglie su dati reali (§3.3).

> Quando questi numeri sono noti, sostituirli nelle tabelle §4 e nel §1: a quel punto la
> metodologia smette di essere un template e diventa la descrizione concreta del progetto.
