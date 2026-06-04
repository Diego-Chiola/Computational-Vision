# Indice del contenuto delle slide (`slides/`)

Struttura e argomenti di ciascun deck del corso di Computational Vision (UniGe — MaLGa,
Francesca Odone & Matteo Moro). I numeri di pagina sono **approssimativi** e servono come
riferimento rapido per il Final Project (vedi `Final Project/Project-Ideas.md`).

Legenda rilevanza per il progetto coni: ★★★ alta · ★★ concettuale · ✗ non applicabile.

---

## `00_cv_introduction.pdf` — Introduzione (contesto)
- Modalità del corso, esame (50% progetto + 50% orale), libro di riferimento (Szeliski, cap. 1)
- Cos'è la computer vision: visione come task di AI, visione umana, definizione (geometria/dinamica/semantica)
- Storia (Marr, summer vision project), progetti classici (stitching, image-based rendering), ML/DL e ImageNet

## `01_image_processing.pdf` — Image processing ★★★
Basato su materiale DSIP. Concetti fondamentali per slope/aspect/shading.
- p.1-9: tipi di pixel, dynamic range, immagini a colori, istogrammi/descrittori globali, quantizzazione
- p.~12: **filtri lineari / convoluzione** `g = f * k`
- p.~13: **gradiente immagine** — magnitudo `M=√(gx²+gy²)` e orientazione `θ=arctan(gy/gx)` → **base di slope e aspect**
- p.~14-17: effetto del rumore, **smoothing gaussiano**, derivata di gaussiana
- p.~22: **filtri derivativi orientabili (steerable)** `G₁^θ = cos(θ)G₁⁰ + sin(θ)G₁⁹⁰` → sintesi shading da direzione arbitraria
- p.~23-26: **edge detection (Canny)**: smoothing → gradiente → non-maxima suppression + soglia
- p.~29-32: **corner (Harris/Shi-Tomasi)**: matrice di autocorrelazione, autovalori λ₁,λ₂
- p.~37-40: **scale-space e piramidi gaussiane**, corner e scala, aliasing, Nyquist

## `03_image_matching.pdf` — Image matching ★★★
- p.~3: matching globale (descrittore per immagine) vs locale (corrispondenze)
- p.~5-7: pipeline locale — **feature detection → description → matching**
- p.~8-13: difficoltà (traslazione/illuminazione/viewpoint), patch come descrittori, esempio SIFT su immagini NASA Mars Rover
- p.~14-16: **detector scale-invariant**, **selezione automatica di scala** (LoG/DoG → blob detection) → utile per crateri circolari a più scale
- (seguito) descrittori SIFT, matching, RANSAC/omografia

## `04_3Dvision_small.pdf` — Principi di 3D computer vision ★★
- p.3: **sensori 3D attivi** (LIDAR, IR/RGBD, RADAR) → range image = **il DEM è di questo tipo**
- p.4: geometria della formazione dell'immagine (camera, focale)
- p.5-6: scale ambiguity, proiezione prospettica non invertibile
- p.7-8: **stereopsi** (seconda vista per recuperare la profondità)

## `04_motion_analysis.pdf` — Motion analysis ✗
- Sequenze video/frame, importanza del moto, brightness constancy
- Problemi: corrispondenza (**optical flow**) e ricostruzione 3D
- (seguito) background subtraction, tracking, classificazione azioni
- → richiede dati temporali: **non applicabile a un DEM statico**

## `06_image_representations_intro.pdf` — Capire la semantica dell'immagine ★★★
- p.2-5: image classification (binaria/multiclasse), **esempio face/not-face = positive/negative**, recognition vs instance
- p.6-7: cosa sono le rappresentazioni, robustezza a variazioni intra-classe
- p.~8: rappresentazioni globali (pixel, istogrammi grigi/colore/gradiente, bag-of-keypoints, feature CNN)
- p.~9: **Bag of words/keypoints** (ispirazione dal testo)
- p.~10: pipeline **coding-pooling** (descrittori SIFT/HOG → coding → pooling → SVM)
- p.~11-14: feature learning, **CNN tipica**, ImageNet, visualizzazione di ciò che la CNN impara

## `09_segmentation.pdf` — Image segmentation ★★★
- p.2-7: segmentazione vs superpixel, supervised/unsupervised, edge-based vs region-based, definizione formale
- p.~8: metodi region-based (thresholding, region growing, watershed, clustering, graph cut)
- p.~13: **thresholding + connected components** → estrazione blob (caso classico binario)
- p.~15-18: **metodo di Otsu** (soglia globale ottima, between-class variance)
- p.~19-22: smoothing per migliorare il thresholding, **soglia variabile/adattiva**, moving average
- p.~23-27: color spaces (HSV), **color thresholding con esempi positive/negative (skin segmentation)**
- p.~28-31: **K-means clustering** per segmentazione (spazio RGB/RGBXY), scelta di K con **Davies-Bouldin**, pro/contro

## `SLIC_Superpixels.pdf` — SLIC Superpixels (paper Achanta et al., EPFL) ★★★
- p.1-4: cosa sono i superpixel, confronto metodi (graph-based vs gradient-ascent), mean-shift/quick-shift/watershed
- p.5: **distance measure 5D** `[l,a,b,x,y]`, `Ds = d_lab + (m/S)·d_xy` (funziona anche su grayscale)
- p.6: **Algoritmo SLIC** (k-means localizzato, complessità O(N))
- p.7-10: confronti, under-segmentation error, boundary recall, efficienza

## `semantic_segmentation.pdf` — Semantic segmentation ★★★
- p.2-3: definizione di segmentazione (senza significato semantico)
- p.5: idea generale — **etichettare ogni pixel** con una categoria (non distingue istanze)
- p.6: sliding window con CNN (inefficiente)
- p.7: **Fully Convolutional Network (FCN)**
- p.8: **encoder-decoder (downsample + upsample)** → struttura tipo **U-Net**
- p.9-12: pooling (max/average), **unpooling / max-unpooling**, downsampling con convoluzione (transposed)

## `RCNN.pdf` — Object detection & segmentation: approcci DL ★★★
- p.2-4: **valutazione dei detector** — **IoU / Jaccard metric**
- p.4: caso binario — TP/FP/FN/TN (soglia IoU ~0.5)
- p.5: esempi TP/FP/FN
- p.6: **Precision / Recall / F1**
- p.7-8: **confusion matrix** (valutazione multiclasse)
- p.9: R-CNN / **Faster R-CNN** (region proposal networks)
- p.10: nuance — classification / localization / detection / instance segmentation
- p.11: sliding window (detection = classificazione a ogni posizione)
- p.12-14: **classification + localization** (regressione del box, multitask loss)

## `Human_in_the_image_action.pdf` — Human action ✗
- Action/activity recognition su persone → non applicabile a DEM geologici

## `human_pose.pdf` — "Human in the image": Human motion / pose ✗
- p.1-5: motion capture, CGI (marker-based), applicazioni mediche (analisi del cammino), pose estimation in ambienti non vincolati
- → specifico sull'uomo: **non applicabile a DEM geologici**

## `Szeliski_CVAABook_2ndEd.pdf` — Libro di testo
- R. Szeliski, *Computer Vision: Algorithms and Applications*, 2nd ed., 2022 — riferimento (cap. 1 lettura iniziale; sez. 3.2 per i filtri)
