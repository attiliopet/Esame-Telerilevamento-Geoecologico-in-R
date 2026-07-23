# 🌳 Analisi Multitemporale della Muraglia Verde (2005 - 2022)
### Esame di Telerilevamento Geo-Ecologico in R - 2026
#### [Petreni Attilio] — matricola n. [0001055390]

# 📑 Introduzione

La "Muraglia Verde" (Great Green Wall) è un'iniziativa panafricana lanciata nel 2007 dall'Unione Africana per contrastare la desertificazione e il degrado del suolo lungo la fascia del Sahel. Il progetto prevede la realizzazione di un mosaico di alberi, pascoli e vegetazione lungo una fascia di circa 8.000 km che attraversa il continente dal Senegal, a ovest, a Gibuti, a est, coinvolgendo undici paesi saheliani (tra cui Senegal, Mauritania, Mali, Niger, Ciad, Sudan ed Etiopia). L'obiettivo dichiarato dell'iniziativa, da raggiungere entro il 2030, è il ripristino di 100 milioni di ettari di terreno degradato.

Il degrado del suolo in quest'area è storicamente legato a siccità ricorrenti, sovrasfruttamento agro-pastorale e crescente pressione demografica: fattori che rendono il monitoraggio satellitare uno strumento importante per valutare l'efficacia reale degli interventi di ripristino sul campo.

In questo progetto si confrontano due immagini satellitari Landsat acquisite a circa 17 anni di distanza (2005 e 2022) su un settore della Muraglia Verde, per valutare se e in che misura l'intervento abbia prodotto un effettivo rinverdimento dell'area e una riduzione delle superfici di suolo nudo.

<p align="center">
  <img src="img/00_area_di_studio.png" width="800">
</p>

> Area di studio (da sostituire con una mappa/immagine reale della zona analizzata)

# 🎯 Obiettivo del progetto

L'obiettivo dell'elaborazione in R è monitorare i cambiamenti nella vegetazione e nel suolo nudo tra il 2005 (pre-intervento) e il 2022 (post-intervento), calcolando quattro indici spettrali a partire dalle bande Blu, Rosso, Vicino Infrarosso (NIR) e SWIR:

- **DVI** (Difference Vegetation Index) → quantità assoluta di vegetazione
- **NDVI** (Normalized Difference Vegetation Index) → salute/vigore della vegetazione
- **BSI** (Bare Soil Index) → presenza di suolo nudo/esposto
- **SAVI** (Soil Adjusted Vegetation Index) → vegetazione corretta per il rumore del suolo

Attraverso la classificazione binaria (a soglia) di NDVI e SAVI e il confronto percentuale tra le due date, si vuole quantificare l'eventuale riduzione del suolo nudo e l'aumento della copertura vegetale nell'area di studio.

# 🛰️ Metodologia: raccolta delle immagini

Le immagini satellitari sono state scaricate tramite [**Google Earth Engine**](https://earthengine.google.com/), selezionando l'area di studio lungo la Muraglia Verde e i due periodi di acquisizione:
- Pre-intervento: immagine **Landsat 5 TM**, anno 2005
- Post-intervento: immagine **Landsat 8 OLI**, anno 2022

Data la diversa numerazione nativa delle bande tra i due sensori, in fase di esportazione le bande sono state riordinate secondo lo schema comune **1=BLUE, 2=GREEN, 3=RED, 4=NIR, 5=SWIR**, così da rendere le due immagini direttamente confrontabili banda per banda nelle analisi successive.

## Script Google Earth Engine (JavaScript)

```javascript
// =====================================================================
// GOOGLE EARTH ENGINE SCRIPT: Esportazione Immagini per Muraglia Verde
// Anni: 2005 (Landsat 5 - Pre) e 2022 (Landsat 8 - Post)
// =====================================================================

// 1. Definizione dell'Area di Studio (ROI - Region of Interest)
// Area nel Sahel / Senegal (settore Louga / Ferlo della Muraglia Verde)
var roi = /* color: #d63031 */ ee.Geometry.Polygon(
    [[[-15.45, 15.35],
      [-15.45, 15.25],
      [-15.30, 15.25],
      [-15.30, 15.35]]]);
Map.centerObject(roi, 11);

// 2. Funzione per la rimozione delle nuvole (Cloud Masking Landsat SR)
function maskL578sr(image) {
  var qa = image.select('QA_PIXEL');
  var cloudShadowMask = (1 << 4);
  var cloudsBitMask = (1 << 3);
  var mask = qa.bitwiseAnd(cloudShadowMask).eq(0)
    .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
  return image.updateMask(mask);
}

// =====================================================================
// 3. ELABORAZIONE IMMAGINE 2005 (Landsat 5 - Pre-Muraglia)
// =====================================================================
var l5 = ee.ImageCollection('LANDSAT/LT05/C02/T1_L2')
  .filterBounds(roi)
  .filterDate('2005-01-01', '2005-12-31')
  .filter(ee.Filter.lt('CLOUD_COVER', 10)) // Filtraggio per bassa nuvolosità
  .map(maskL578sr)
  .median() // Composizione mediana per rimuovere eventuali disturbi residui
  .clip(roi);

// Landsat 5: SR_B1=Blue, SR_B2=Green, SR_B3=Red, SR_B4=NIR, SR_B5=SWIR1
var imgPre = l5.select(
  ['SR_B1', 'SR_B2', 'SR_B3', 'SR_B4', 'SR_B5'],
  ['BLUE', 'GREEN', 'RED', 'NIR', 'SWIR']
).multiply(0.0000275).add(-0.2); // Fattore di scaling ufficiale Collection 2

Map.addLayer(imgPre, {bands: ['NIR', 'RED', 'BLUE'], min: 0, max: 0.4}, 'Pre-Muraglia 2005');

// =====================================================================
// 4. ELABORAZIONE IMMAGINE 2022 (Landsat 8 - Post-Muraglia)
// =====================================================================
var l8 = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
  .filterBounds(roi)
  .filterDate('2022-01-01', '2022-12-31')
  .filter(ee.Filter.lt('CLOUD_COVER', 10))
  .map(maskL578sr)
  .median()
  .clip(roi);

// Landsat 8: SR_B2=Blue, SR_B3=Green, SR_B4=Red, SR_B5=NIR, SR_B6=SWIR1
var imgPost = l8.select(
  ['SR_B2', 'SR_B3', 'SR_B4', 'SR_B5', 'SR_B6'],
  ['BLUE', 'GREEN', 'RED', 'NIR', 'SWIR']
).multiply(0.0000275).add(-0.2);

Map.addLayer(imgPost, {bands: ['NIR', 'RED', 'BLUE'], min: 0, max: 0.4}, 'Post-Muraglia 2022');

// =====================================================================
// 5. ESPORTAZIONE SU GOOGLE DRIVE (GeoTIFF)
// =====================================================================
Export.image.toDrive({
  image: imgPre,
  description: 'Landsat5_Pre_Muraglia',
  folder: 'immagini telerilevamento',
  region: roi,
  scale: 30, // Risoluzione spaziale Landsat (30 metri)
  maxPixels: 1e9
});

Export.image.toDrive({
  image: imgPost,
  description: 'Landsat8_Post_Muraglia',
  folder: 'immagini telerilevamento',
  region: roi,
  scale: 30,
  maxPixels: 1e9
});
```

> [!NOTE]
> Lo script filtra le collezioni Landsat Collection 2 Level 2 (riflettanza di superficie) per copertura nuvolosa < 10%, applica una maschera nuvole/ombre tramite la banda `QA_PIXEL`, calcola il composito mediano annuale e applica il fattore di scala ufficiale USGS (moltiplicatore 0.0000275, offset -0.2) per ottenere valori di riflettanza fisicamente interpretabili tra 0 e 1.

---

# 💻 Analisi in R

## 1. Caricamento delle librerie

```r
library(terra)      # Pacchetto fondamentale per la gestione e l'analisi dei dati raster
library(imageRy)    # Pacchetto didattico per l'analisi di immagini satellitari
library(ggplot2)    # Libreria per la creazione di grafici avanzati
library(viridis)    # Palette di colori colorblind-friendly (es. viridis, inferno, magma)
library(reshape2)   # Utile per la manipolazione di dataframe complessi (da wide a long)
```

## 2. Importazione delle immagini

```r
setwd("C:/Users/attilio/Desktop/immagini telerilevamento")

pre <- rast("Landsat5_Pre_Muraglia.tif")    # Immagine antecedente al progetto (2005)
post <- rast("Landsat8_Post_Muraglia.tif")  # Immagine successiva agli interventi (2022)
```

> [!NOTE]
> Le immagini contengono le bande 1=BLUE, 2=GREEN, 3=RED, 4=NIR, 5=SWIR (vedi Metodologia sopra).

## 3. Visualizzazione rapida (falsi colori) 🌈

```r
im.multiframe(1, 2)
im.plotRGB(pre, r = 4, g = 3, b = 2, title = "Pre-Muraglia (2005)")
im.plotRGB(post, r = 4, g = 3, b = 2, title = "Post-Muraglia (2022)")
dev.off()
```

<p align="center">
  <img src="img/03_falsi_colori.png" width="800">
</p>

> **Commento**
>
> Nel composito NIR-Rosso-Verde la vegetazione sana appare rossa intensa (alta riflettanza nel NIR), mentre suolo nudo e superfici urbanizzate restano su toni chiari. Confrontando le due immagini si individuano a colpo d'occhio le zone in cui la copertura vegetale è aumentata o diminuita tra il 2005 e il 2022.

## 4. Istogrammi delle bande 📊

```r
limite_x <- range(c(values(pre[[1:4]]), values(post[[1:4]])), na.rm = TRUE)

dens_max <- c(
  sapply(1:4, function(i) max(hist(values(pre[[i]]),  plot = FALSE)$density)),
  sapply(1:4, function(i) max(hist(values(post[[i]]), plot = FALSE)$density))
)
limite_y <- c(0, max(dens_max))

im.multiframe(2, 4)
hist(values(pre[[1]]), freq = FALSE, main="Istogramma Blue 2005", col="blue", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
hist(values(pre[[2]]), freq = FALSE, main="Istogramma Green 2005", col="green", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
hist(values(pre[[3]]), freq = FALSE, main="Istogramma Red 2005", col="red", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
hist(values(pre[[4]]), freq = FALSE, main="Istogramma NIR 2005", col="purple", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)

hist(values(post[[1]]), freq = FALSE, main="Istogramma Blue 2022", col="blue", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
hist(values(post[[2]]), freq = FALSE, main="Istogramma Green 2022", col="green", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
hist(values(post[[3]]), freq = FALSE, main="Istogramma Red 2022", col="red", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
hist(values(post[[4]]), freq = FALSE, main="Istogramma NIR 2022", col="purple", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
dev.off()
```

<p align="center">
  <img src="img/04_istogrammi.png" width="900">
</p>

> [!TIP]
> `limite_x` e `limite_y` fissano lo stesso range su tutti gli 8 istogrammi, così bande diverse e anni diversi restano direttamente confrontabili tra loro.

## 5. Plot delle bande separate 🎨

```r
im.multiframe(2, 4)
plot(pre[[1]], col = magma(100), main = "Pre - Blue")
plot(pre[[2]], col = magma(100), main = "Pre - Green")
plot(pre[[3]], col = magma(100), main = "Pre - Red")
plot(pre[[4]], col = magma(100), main = "Pre - NIR")

plot(post[[1]], col = magma(100), main = "Post - Blue")
plot(post[[2]], col = magma(100), main = "Post - Green")
plot(post[[3]], col = magma(100), main = "Post - Red")
plot(post[[4]], col = magma(100), main = "Post - NIR")
dev.off()
```

<p align="center">
  <img src="img/05_bande_separate.png" width="900">
</p>

> **Commento**
>
> La banda NIR è la più informativa sullo stato di salute della vegetazione: una riflettanza più alta (toni chiari nella palette magma) corrisponde a vegetazione più vigorosa. Un suo calo tra il 2005 e il 2022 in una data zona segnala una perdita di biomassa; un suo aumento, al contrario, un rinverdimento.

## 6. Indice DVI e differenza 🌿

$$ DVI = NIR - RED $$

```r
dvi_pre <- pre[[4]] - pre[[3]]
dvi_post <- post[[4]] - post[[3]]
diff_dvi <- dvi_post - dvi_pre

im.multiframe(1, 3)
plot(dvi_pre, main = "DVI 2005", col = viridis::viridis(100))
plot(dvi_post, main = "DVI 2022", col = viridis::viridis(100))
plot(diff_dvi, main = "Differenza DVI (ΔDVI)", col = viridis::inferno(100))
dev.off()
```

<p align="center">
  <img src="img/06_dvi.png" width="900">
</p>

> **Commento**
>
> Essendo un indice non normalizzato, il DVI è utile soprattutto nel confronto diretto (differenza) tra le due date: valori positivi in `diff_dvi` indicano un aumento della biomassa fogliare (rinverdimento) tra il 2005 e il 2022, valori negativi segnalano invece una perdita di vegetazione.

## 7. Indice NDVI, differenza e classificazione 🍃

$$ NDVI = \frac{NIR - RED}{NIR + RED} $$

```r
ndvi_pre <- (pre[[4]] - pre[[3]]) / (pre[[4]] + pre[[3]])
ndvi_post <- (post[[4]] - post[[3]]) / (post[[4]] + post[[3]])
diff_ndvi <- (ndvi_post - ndvi_pre)

im.multiframe(1, 3)
plot(ndvi_pre, col=viridis(100), range=c(0,1), main = "NDVI 2005")
plot(ndvi_post, col=viridis(100), range=c(0,1), main = "NDVI 2022")
plot(diff_ndvi, col=viridis(100), range=c(-0.5,0.5), main = "Differenza NDVI")
dev.off()
```

<p align="center">
  <img src="img/07_ndvi.png" width="900">
</p>

> **Commento**
>
> Valori di NDVI vicini a 1 indicano vegetazione densa e sana, valori prossimi a 0 (o negativi) indicano suolo nudo, roccia o acqua. Nella mappa di differenza, i valori positivi indicano un aumento dell'NDVI (miglioramento della vegetazione) tra il 2005 e il 2022, i valori negativi un peggioramento.

### Classificazione binaria NDVI

```r
soglia = 0.3
classi_pre = classify(ndvi_pre, rcl=matrix(c(-Inf, soglia, 0,  soglia, Inf, 1), ncol=3, byrow=TRUE))
classi_post = classify(ndvi_post, rcl=matrix(c(-Inf, soglia, 0,  soglia, Inf, 1), ncol=3, byrow=TRUE))

freq_pre = freq(classi_pre)
freq_post = freq(classi_post)

perc_pre = freq_pre$count * 100 / ncell(classi_pre)
perc_post = freq_post$count * 100 / ncell(classi_post)

tabella_ndvi = data.frame(
  Classe = c("Suolo nudo", "Vegetazione"),
  Pre_Muraglia = round(perc_pre, 2),
  Post_Muraglia = round(perc_post, 2)
)
print(tabella_ndvi)
```

> [!IMPORTANT]
> Si è scelta la soglia NDVI = 0.3 per separare suolo nudo e vegetazione. La tabella assume che entrambe le classi (0 e 1) siano presenti in entrambe le immagini nello stesso ordine restituito da `freq()`; in caso di dubbio conviene controllare `freq_pre$value` e `freq_post$value`.

| Classe | Pre-Muraglia (2005) | Post-Muraglia (2022) |
|---|---|---|
| Suolo nudo | _da compilare_ % | _da compilare_ % |
| Vegetazione | _da compilare_ % | _da compilare_ % |

## 8. Indice BSI (Bare Soil Index) 🏜️

$$ BSI = \frac{(SWIR + RED) - (NIR + BLUE)}{(SWIR + RED) + (NIR + BLUE)} $$

```r
bsi_pre  <- ((pre[[5]] + pre[[3]]) - (pre[[4]] + pre[[1]])) / ((pre[[5]] + pre[[3]]) + (pre[[4]] + pre[[1]]))
bsi_post <- ((post[[5]] + post[[3]]) - (post[[4]] + post[[1]])) / ((post[[5]] + post[[3]]) + (post[[4]] + post[[1]]))
diff_bsi <- bsi_pre - bsi_post

im.multiframe(1, 3)
plot(bsi_pre,  main = "BSI 2005", col = viridis::viridis(100), range = c(-1, 1))
plot(bsi_post, main = "BSI 2022", col = viridis::viridis(100), range = c(-1, 1))
plot(diff_bsi, main = "Differenza BSI", col = viridis::inferno(100), range = c(-0.2, 0.2))
dev.off()
```

<p align="center">
  <img src="img/08_bsi.png" width="900">
</p>

> **Commento**
>
> Valori più alti di BSI indicano maggiore presenza di suolo nudo/esposto. Qui la differenza è calcolata come (pre - post): un valore positivo significa che il BSI era più alto nel 2005 rispetto al 2022, cioè che il suolo nudo si è ridotto, in linea con l'obiettivo di rinverdimento della Muraglia Verde.

## 9. Indice SAVI e classificazione 🌱

$$ SAVI = \frac{NIR - RED}{NIR + RED + L} \times (1 + L) $$

```r
L <- 0.5
savi_pre <- ((pre[[4]] - pre[[3]]) / (pre[[4]] + pre[[3]] + L)) * (1 + L)
savi_post <- ((post[[4]] - post[[3]]) / (post[[4]] + post[[3]] + L)) * (1 + L)
diff_savi <- savi_post - savi_pre

im.multiframe(1, 3)
plot(savi_pre, main = "SAVI 2005", col = viridis::viridis(100), range = c(0, 1))
plot(savi_post, main = "SAVI 2022", col = viridis::viridis(100), range = c(0, 1))
plot(diff_savi, main = "Differenza SAVI", col = viridis::inferno(100), range = c(-0.3, 0.3))
dev.off()
```

<p align="center">
  <img src="img/09_savi.png" width="900">
</p>

### Classificazione binaria SAVI

```r
soglia_savi <- 0.2
classi_savi_pre <- classify(savi_pre, rcl=matrix(c(-Inf, soglia_savi, 0, soglia_savi, Inf, 1), ncol=3, byrow=TRUE))
classi_savi_post <- classify(savi_post, rcl=matrix(c(-Inf, soglia_savi, 0, soglia_savi, Inf, 1), ncol=3, byrow=TRUE))

diff_classi_savi <- classi_savi_post - classi_savi_pre

im.multiframe(1, 3)
plot(classi_savi_pre,  main = "Classi SAVI 2005", col = c("orange", "darkgreen"))
plot(classi_savi_post, main = "Classi SAVI 2022", col = c("orange", "darkgreen"))
plot(diff_classi_savi, main = "Cambiamento classi SAVI (2005 → 2022)", col = c("firebrick3", "grey85", "darkgreen"))
dev.off()

freq_savi_pre <- freq(classi_savi_pre)
freq_savi_post <- freq(classi_savi_post)

perc_savi_pre <- freq_savi_pre$count * 100 / ncell(classi_savi_pre)
perc_savi_post <- freq_savi_post$count * 100 / ncell(classi_savi_post)

tabella_savi <- data.frame(
  Classe = c("Suolo/Aree degradate", "Vegetazione SAVI"),
  Pre_Muraglia = round(perc_savi_pre, 2),
  Post_Muraglia = round(perc_savi_post, 2)
)
print(tabella_savi)
```

<p align="center">
  <img src="img/09b_classi_savi.png" width="900">
</p>

> **Commento**
>
> Nelle prime due mappe l'arancione indica suolo/aree degradate (SAVI < soglia_savi) e il verde scuro la vegetazione. Nella mappa di cambiamento, il verde scuro (+1) individua le zone di nuovo rinverdimento (da suolo a vegetazione), il rosso (-1) le zone di vegetazione persa (da vegetazione a suolo), il grigio (0) le aree rimaste nella stessa classe tra il 2005 e il 2022.

| Classe | Pre-Muraglia (2005) | Post-Muraglia (2022) |
|---|---|---|
| Suolo/Aree degradate | _da compilare_ % | _da compilare_ % |
| Vegetazione SAVI | _da compilare_ % | _da compilare_ % |

## 10. Confronto grafico delle percentuali (NDVI e SAVI) 📉

```r
tabella_ndvi_long <- melt(tabella_ndvi, id.vars = "Classe", variable.name = "Periodo", value.name = "Percentuale")
tabella_savi_long <- melt(tabella_savi, id.vars = "Classe", variable.name = "Periodo", value.name = "Percentuale")

ggplot(tabella_ndvi_long, aes(x = Classe, y = Percentuale, fill = Periodo)) +
  geom_bar(stat = "identity", position = "dodge") +
  geom_text(aes(label = round(Percentuale, 1)), position = position_dodge(width = 0.9), vjust = -0.25, size = 3) +
  scale_fill_viridis_d() +
  ylim(0, 100) +
  labs(title = "Classificazione NDVI: confronto Pre/Post Muraglia Verde", y = "% area", x = NULL, fill = "Periodo") +
  theme_minimal()

ggplot(tabella_savi_long, aes(x = Classe, y = Percentuale, fill = Periodo)) +
  geom_bar(stat = "identity", position = "dodge") +
  geom_text(aes(label = round(Percentuale, 1)), position = position_dodge(width = 0.9), vjust = -0.25, size = 3) +
  scale_fill_viridis_d() +
  ylim(0, 100) +
  labs(title = "Classificazione SAVI: confronto Pre/Post Muraglia Verde", y = "% area", x = NULL, fill = "Periodo") +
  theme_minimal()
```

<p align="center">
  <img src="img/10_grafico_percentuali.png" width="900">
</p>

---

# 📌 Conclusioni

Il confronto tra il 2005 e il 2022 su questo settore della Muraglia Verde in Senegal permette di quantificare, tramite quattro indici spettrali indipendenti (DVI, NDVI, BSI, SAVI), l'entità del cambiamento nella copertura vegetale e nel suolo nudo nell'arco di circa diciassette anni. Confrontando le percentuali di "Suolo nudo/degradato" contro "Vegetazione" tra le due date in entrambe le tabelle (NDVI e SAVI), una diminuzione della quota di suolo nudo accompagnata da un aumento della quota di vegetazione supporterebbe l'ipotesi di un rinverdimento dell'area, in coerenza con quanto osservabile anche nelle mappe di differenza DVI, NDVI e BSI.

È opportuno interpretare questi risultati con cautela, perché la letteratura scientifica sulla Muraglia Verde in Senegal non è unanime. Alcuni studi basati su serie storiche NDVI (Gore et al., 2023) riportano un miglioramento della relazione tra vegetazione e parametri climatici successivo agli interventi di rimboschimento, mentre lavori più recenti basati su telerilevamento satellitare (Zhu et al., 2025) segnalano che, su un campione di aree rimboschite in Senegal, solo una minoranza mostra un rinverdimento significativamente superiore a quello atteso naturalmente, imputando parte del divario tra finanziamenti globali e risultati locali a fattori come la bassa sopravvivenza delle piantine e la discontinuità nella manutenzione. Il confronto pre/post di un singolo settore, come quello condotto in questo progetto, va quindi considerato un caso di studio locale e non necessariamente rappresentativo dell'andamento dell'intera iniziativa.

Restano inoltre alcuni limiti metodologici da tenere presenti:
- il confronto si basa su due sole annate (2005 e 2022): variazioni interannuali di piovosità possono influenzare NDVI/DVI/SAVI indipendentemente dagli interventi di ripristino;
- le soglie di classificazione (0.3 per NDVI, 0.2 per SAVI) sono scelte comuni in letteratura ma restano in parte arbitrarie.

Nel complesso, il telerilevamento multitemporale si conferma comunque uno strumento fondamentale ed economico per monitorare oggettivamente gli effetti di interventi di ripristino ambientale su larga scala, specialmente in aree remote dove il monitoraggio a terra è logisticamente difficile.

# 📚 Riferimenti bibliografici

- Gore, B.T.O., Aman, A., Kouadio, Y., Duclos, O.-M. (2023). Recent Vegetation Cover Dynamics and Climatic Parameters Evolution Study in the Great Green Wall of Senegal. *Journal of Environmental Protection*, 14, 254-284.
- Schucknecht, A., Meroni, M., Rembold, F. (2016). Monitoring Project Impact on Biomass Increase in the Context of the Great Green Wall for the Sahara and Sahel Initiative in Senegal. European Commission Joint Research Centre, Ispra.
- Zhu, A.L., Ndiaye, A., Dahm, R., Mauclaire, M., Boas, I. (2025). Africa's Great Green Mirage? Assessing the disconnect between global finance and local implementation in Africa's Great Green Wall. *Land Use Policy*, 157, 107670.

# Grazie per l'attenzione! 🌳
