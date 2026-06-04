# Idea 4 — Mining delle direzioni delle fissure (extra opzionale)

> Approfondimento dell'idea 4 di [`Project-Ideas.md`](./Project-Ideas.md).
> Deck di riferimento: pochi agganci diretti (vedi avvertenza) — `01_image_processing.pdf`
> (gradiente/orientazione), `09_segmentation.pdf` (centroidi da connected components).
> Dettaglio pagine in [`../slides/Slides-Index.md`](../slides/Slides-Index.md).

## ⚠️ Avvertenza
È un'**estensione opzionale**, non un progetto principale: gli strumenti chiave (PCA su
punti, Hough lineare) **non sono trattati nelle slide del corso**, quindi è meno difendibile
all'orale rispetto alle idee 1-3-5-6. Vale come "tocco geologico" distintivo se hai già un
sistema di detection funzionante.

## In una frase
I coni non sono sparsi a caso: spesso si **allineano lungo le fratture (fissure)** da cui
risale il magma. L'idea è scoprire automaticamente queste **direzioni preferenziali di
allineamento** a partire dalle posizioni dei coni.

---

## 1. Su che dati si lavora

- **El Hierro `Cones.shp`** è l'unico dataset adatto: contiene già **le linee di fissura**
  interpretate dai geologi (oltre a basi e crateri). Sono la ground truth di questa idea.
- Input dell'analisi: i **centroidi dei coni** (un punto per cono) — ottenibili dallo
  shapefile, oppure dall'output di un detector (idea 1/2/3) come passo successivo.
- Etna ha lo shapefile ma senza fissure affidabili; Jeju non ha shapefile → questa idea è
  essenzialmente **El Hierro-only**.

---

## 2. Cosa si fa

Due sotto-problemi possibili:

**a) Direzione globale/locale di allineamento.**
- raggruppa i coni vicini in cluster (es. con K-means o DBSCAN sui centroidi)
- per ogni cluster, stima la **direzione principale** con la **PCA** (l'autovettore maggiore
  della nuvola di punti = asse di allineamento)
- output: una direzione (angolo) per zona → mappa delle orientazioni delle fissure

**b) Linee di fissura come rette nello spazio.**
- costruisci una **mappa di densità** dei centroidi (immagine dove i coni "accendono" pixel)
- applica una **Hough lineare** per trovare le rette lungo cui si addensano i coni
- output: segmenti di retta = fissure candidate

Validazione: confronto angolare tra le direzioni stimate e le **linee di fissura** dello
shapefile di El Hierro (errore in gradi).

---

## 3. Quali strumenti

- `geopandas`/`shapely` per estrarre centroidi e fissure
- `numpy`/`scikit-learn` per **PCA** e clustering (K-means/DBSCAN)
- `scikit-image` (`hough_line`) per la variante con Hough
- nessuna GPU, gira su CPU in pochi secondi

---

## 4. Cosa aspettarsi

- Risultati interessanti e **molto visivi** (frecce/rette sovrapposte alla mappa), ottimi per
  una figura d'impatto nella relazione.
- Fragile dove i coni sono pochi o sparsi; dipende molto dal clustering iniziale.
- Da solo è **troppo poco** per un progetto: ha senso come **capitolo finale** sopra un
  detector già fatto ("ho trovato i coni → adesso ne studio l'organizzazione spaziale").

---

## Dove si colloca
È un **post-processing geologico**: prende l'output di detection (idee 1-2-3) e ci aggiunge
un livello di interpretazione (organizzazione spaziale). Consigliata solo come estensione,
non come cuore del progetto.
