# =====================================================================
# PROGETTO: Analisi Multitemporale della Muraglia Verde (2005 - 2022)
# OBIETTIVO: Quantificare il rinverdimento e la riduzione del suolo nudo
# =====================================================================
# =====================================================================
# INTRODUZIONE
# =====================================================================
# La "Muraglia Verde" (Great Green Wall) è un'iniziativa panafricana lanciata nel
# 2007 dall'Unione Africana per contrastare la desertificazione e il degrado del
# suolo lungo la fascia del Sahel. Il progetto prevede la realizzazione di un
# mosaico di alberi, pascoli e vegetazione lungo una fascia di circa 8.000 km che
# attraversa il continente dal Senegal, a ovest, a Gibuti, a est, coinvolgendo
# undici paesi saheliani (tra cui Senegal, Mauritania, Mali, Niger, Ciad, Sudan
# ed Etiopia). L'obiettivo dichiarato dell'iniziativa, da raggiungere entro il
# 2030, è il ripristino di 100 milioni di ettari di terreno degradato.
#
# Il degrado del suolo in quest'area è storicamente legato a siccità ricorrenti,
# sovrasfruttamento agro-pastorale e crescente pressione demografica: fattori che
# rendono il monitoraggio satellitare uno strumento importante per valutare
# l'efficacia reale degli interventi di ripristino sul campo.
#
# In questo progetto si confrontano due immagini satellitari Landsat acquisite a
# circa 17 anni di distanza (2005 e 2022) su un settore della Muraglia Verde, per
# valutare se e in che misura l'intervento abbia prodotto un effettivo
# rinverdimento dell'area e una riduzione delle superfici di suolo nudo.

# =====================================================================
# OBIETTIVO DEL PROGETTO
# =====================================================================
# L'obiettivo dell'elaborazione in R è monitorare i cambiamenti nella vegetazione
# e nel suolo nudo tra il 2005 (pre-intervento) e il 2022 (post-intervento),
# calcolando quattro indici spettrali a partire dalle bande Blu, Rosso, Vicino
# Infrarosso (NIR) e SWIR:
# - DVI  (Difference Vegetation Index)            -> quantità assoluta di vegetazione
# - NDVI (Normalized Difference Vegetation Index) -> salute/vigore della vegetazione
# - BSI  (Bare Soil Index)                        -> presenza di suolo nudo/esposto
# - SAVI (Soil Adjusted Vegetation Index)         -> vegetazione corretta per il rumore del suolo
# Attraverso la classificazione binaria (a soglia) di NDVI e SAVI e il confronto
# percentuale tra le due date, si vuole quantificare l'eventuale riduzione del
# suolo nudo e l'aumento della copertura vegetale nell'area di studio.

# =====================================================================
# METODOLOGIA: RACCOLTA DELLE IMMAGINI
# =====================================================================
# Le immagini satellitari sono state scaricate tramite Google Earth Engine
# (https://earthengine.google.com/), selezionando l'area di studio lungo la
# Muraglia Verde e i due periodi di acquisizione:
# - Pre-intervento: immagine Landsat 5 TM, anno 2005
# - Post-intervento: immagine Landsat 8 OLI, anno 2022
# Il codice JavaScript utilizzato per selezionare ed esportare le immagini da GEE
# è disponibile nel file Code.js.
#
# Data la diversa numerazione nativa delle bande tra i due sensori, in fase di
# esportazione le bande sono state riordinate secondo lo schema comune
# 1=BLUE, 2=GREEN, 3=RED, 4=NIR, 5=SWIR, così da rendere le due immagini
# direttamente confrontabili banda per banda nelle analisi successive.
# =====================================================================
# 1. CARICAMENTO DELLE LIBRERIE
# =====================================================================
# Richiamiamo i pacchetti necessari per l'analisi spaziale e la visualizzazione
library(terra)      # Pacchetto fondamentale per la gestione e l'analisi dei dati raster
library(imageRy)    # Pacchetto didattico per l'analisi di immagini satellitari
library(ggplot2)    # Libreria per la creazione di grafici avanzati
library(viridis)    # Palette di colori colorblind-friendly (es. viridis, inferno, magma)
library(reshape2)   # Utile per la manipolazione di dataframe complessi (da wide a long)

# =====================================================================
# 2. IMPORTAZIONE DELLE IMMAGINI
# =====================================================================
# Impostiamo la directory di lavoro dove sono salvati i file .tif
setwd("C:/Users/attilio/Desktop/immagini telerilevamento")

# Usiamo la funzione rast() di 'terra' per importare i dati satellitari
# Le immagini contengono le bande: 1=BLUE, 2=GREEN, 3=RED, 4=NIR, 5=SWIR
# NB: pre deriva da Landsat 5 (TM) e post da Landsat 8 (OLI). I due sensori hanno
# una numerazione nativa delle bande diversa, per cui in fase di pre-processing
# le bande sono state riordinate secondo lo schema comune sopra indicato, in modo
# da renderle direttamente confrontabili banda per banda tra le due date.
pre <- rast("Landsat5_Pre_Muraglia.tif")    # Immagine antecedente al progetto (2005)
post <- rast("Landsat8_Post_Muraglia.tif")  # Immagine successiva agli interventi (2022)

# =====================================================================
# 3. VISUALIZZAZIONE RAPIDA (Falsi Colori)
# =====================================================================
# Creiamo un'immagine in falsi colori (Infrarosso a colori) per far risaltare la biomassa.
# Montando la banda NIR (4) sul canale del Rosso (r), la vegetazione apparirà rossa intenso.
par(mfrow=c(1,2)) # Divide lo spazio grafico in 1 riga e 2 colonne
im.plotRGB(pre, r = 4, g = 3, b = 2, title = "Pre-Muraglia (2005)")
im.plotRGB(post, r = 4, g = 3, b = 2, title = "Post-Muraglia (2022)")
dev.off() # Pulisce e chiude la finestra grafica

# Commento: nel composito NIR-Rosso-Verde la vegetazione sana appare rossa intensa
# (alta riflettanza nel NIR), mentre suolo nudo e superfici urbanizzate restano su
# toni chiari. Confrontando le due immagini si individuano a colpo d'occhio le zone
# in cui la copertura vegetale è aumentata o diminuita tra il 2005 e il 2022.

# =====================================================================
# 4. ISTOGRAMMI DELLE BANDE (Riflettanza)
# =====================================================================
# Analizziamo la distribuzione dei valori di riflettanza pixel per pixel per ogni banda.

# Calcoliamo un intervallo comune per l'asse x (valori di riflettanza) e per l'asse y
# (densità), così da rendere gli 8 istogrammi seguenti direttamente confrontabili
# tra loro, sia tra bande diverse sia tra il 2005 e il 2022.
limite_x <- range(c(values(pre[[1:4]]), values(post[[1:4]])), na.rm = TRUE)

dens_max <- c(
  sapply(1:4, function(i) max(hist(values(pre[[i]]),  plot = FALSE)$density)),
  sapply(1:4, function(i) max(hist(values(post[[i]]), plot = FALSE)$density))
)
limite_y <- c(0, max(dens_max))

par(mfrow=c(2,4)) # Spazio grafico a 2 righe e 4 colonne

# Istogrammi 2005 (Impostiamo freq=FALSE per visualizzare la densità di probabilità)
# --- Istogrammi 2005 (Pre-Muraglia) ---
hist(values(pre[[1]]), freq = FALSE, main="Istogramma Blue 2005", col="blue", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
hist(values(pre[[2]]), freq = FALSE, main="Istogramma Green 2005", col="green", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
hist(values(pre[[3]]), freq = FALSE, main="Istogramma Red 2005", col="red", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y) 
hist(values(pre[[4]]), freq = FALSE, main="Istogramma NIR 2005", col="purple", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y) 

# --- Istogrammi 2022 (Post-Muraglia) ---
hist(values(post[[1]]), freq = FALSE, main="Istogramma Blue 2022", col="blue", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y) 
hist(values(post[[2]]), freq = FALSE, main="Istogramma Green 2022", col="green", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y) 
hist(values(post[[3]]), freq = FALSE, main="Istogramma Red 2022", col="red", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y) 
hist(values(post[[4]]), freq = FALSE, main="Istogramma NIR 2022", col="purple", xlab="Riflettanza", ylab="Densità", xlim=limite_x, ylim=limite_y)
dev.off()

# =====================================================================
# 5. PLOT DELLE BANDE SEPARATE
# =====================================================================
# Visualizziamo spazialmente la riflettanza di ogni singola banda usando la palette 'magma'.
par(mfrow=c(2,4))

# Bande 2005
plot(pre[[1]], col = magma(100), main = "Pre - Blue")
plot(pre[[2]], col = magma(100), main = "Pre - Green") 
plot(pre[[3]], col = magma(100), main = "Pre - Red") 
plot(pre[[4]], col = magma(100), main = "Pre - NIR") 

# Bande 2022
plot(post[[1]], col = magma(100), main = "Post - Blue")
plot(post[[2]], col = magma(100), main = "Post - Green") 
plot(post[[3]], col = magma(100), main = "Post - Red") 
plot(post[[4]], col = magma(100), main = "Post - NIR") 
dev.off()

# Commento: la banda NIR è la più informativa sullo stato di salute della vegetazione:
# una riflettanza più alta (toni chiari nella palette magma) corrisponde a vegetazione
# più vigorosa. Un suo calo tra il 2005 e il 2022 in una data zona segnala una perdita
# di biomassa; un suo aumento, al contrario, un rinverdimento.

# =====================================================================
# 6. INDICE DVI E DIFFERENZA
# =====================================================================
# Il Difference Vegetation Index misura la differenza assoluta tra NIR e RED.
# Formula: DVI = NIR - RED
dvi_pre <- pre[[4]] - pre[[3]]
dvi_post <- post[[4]] - post[[3]]
diff_dvi <- dvi_post - dvi_pre # Calcoliamo il delta per vedere dove la biomassa è aumentata

# Visualizzazione
par(mfrow=c(1,3))
plot(dvi_pre, main = "DVI 2005", col = viridis::viridis(100))
plot(dvi_post, main = "DVI 2022", col = viridis::viridis(100))
plot(diff_dvi, main = "Differenza DVI (ΔDVI)", col = viridis::inferno(100))
dev.off()

# Commento: essendo un indice non normalizzato, il DVI è utile soprattutto nel
# confronto diretto (differenza) tra le due date: valori positivi in diff_dvi
# indicano un aumento della biomassa fogliare (rinverdimento) tra il 2005 e il 2022,
# valori negativi segnalano invece una perdita di vegetazione.

# =====================================================================
# 7. INDICE NDVI, DIFFERENZA E CLASSIFICAZIONE
# =====================================================================
# Il Normalized Difference Vegetation Index normalizza i valori tra -1 e +1.
# Formula: NDVI = (NIR - RED) / (NIR + RED)
ndvi_pre <- (pre[[4]] - pre[[3]]) / (pre[[4]] + pre[[3]])
ndvi_post <- (post[[4]] - post[[3]]) / (post[[4]] + post[[3]])
diff_ndvi <- (ndvi_post - ndvi_pre)

# Visualizzazione (Fissiamo il range a c(0,1) per l'NDVI e c(-0.5, 0.5) per la differenza)
par(mfrow=c(1,3))
plot(ndvi_pre, col=viridis(100), range=c(0,1), main = "NDVI 2005")
plot(ndvi_post, col=viridis(100), range=c(0,1), main = "NDVI 2022")
plot(diff_ndvi, col=viridis(100), range=c(-0.5,0.5), main = "Differenza NDVI")
dev.off()

# Commento: valori di NDVI vicini a 1 indicano vegetazione densa e sana, valori
# prossimi a 0 (o negativi) indicano suolo nudo, roccia o acqua. Nella mappa di
# differenza, i valori positivi indicano un aumento dell'NDVI (miglioramento della
# vegetazione) tra il 2005 e il 2022, i valori negativi un peggioramento.
# Nota: se alcuni pixel dovessero uscire dal range impostato in plot(), verranno
# visualizzati con il colore limite della palette; ampliare il range se necessario.

# --- CLASSIFICAZIONE BINARIA NDVI ---
# Stabiliamo una soglia ecologica di 0.3 per separare il suolo nudo dalla vegetazione
soglia = 0.3 

# Riclassifichiamo il raster in base alla soglia tramite una matrice a 3 colonne
# (da, a, diventa): NDVI < soglia -> classe 0 (suolo nudo); NDVI >= soglia -> classe 1 (vegetazione)
classi_pre = classify(ndvi_pre, rcl=matrix(c(-Inf, soglia, 0,  soglia, Inf, 1), ncol=3, byrow=TRUE))
classi_post = classify(ndvi_post, rcl=matrix(c(-Inf, soglia, 0,  soglia, Inf, 1), ncol=3, byrow=TRUE))

# Calcoliamo le frequenze assolute (numero di pixel per classe)
freq_pre = freq(classi_pre)
freq_post = freq(classi_post)

# Convertiamo i pixel in percentuali relative all'intera area di studio
# ncell() calcola il numero totale di celle nel raster
perc_pre = freq_pre$count * 100 / ncell(classi_pre)
perc_post = freq_post$count * 100 / ncell(classi_post)

# Nota: qui si assume che entrambe le classi (0 = suolo nudo, 1 = vegetazione) siano
# presenti in entrambe le immagini e nello stesso ordine restituito da freq(). È
# buona norma, in caso di dubbio, controllare freq_pre$value e freq_post$value
# prima di costruire la tabella sottostante.

# Creazione della tabella finale riassuntiva
tabella_ndvi = data.frame(
  Classe = c("Suolo nudo", "Vegetazione"),
  Pre_Muraglia = round(perc_pre, 2),
  Post_Muraglia = round(perc_post, 2)
)
print("--- STATISTICHE PERCENTUALI NDVI ---")
print(tabella_ndvi) 

# =====================================================================
# 8. INDICE BSI (Bare Soil Index)
# =====================================================================
# Il BSI evidenzia le aree di suolo nudo combinando le bande SWIR e RED contro NIR e BLUE.
# Formula: BSI = ((SWIR + RED) - (NIR + BLUE)) / ((SWIR + RED) + (NIR + BLUE))
bsi_pre  <- ((pre[[5]] + pre[[3]]) - (pre[[4]] + pre[[1]])) / ((pre[[5]] + pre[[3]]) + (pre[[4]] + pre[[1]]))
bsi_post <- ((post[[5]] + post[[3]]) - (post[[4]] + post[[1]])) / ((post[[5]] + post[[3]]) + (post[[4]] + post[[1]]))
diff_bsi <- bsi_pre - bsi_post # I valori positivi in diff_bsi indicheranno una riduzione del suolo nudo

# Visualizzazione (Fissiamo il range della differenza molto stretto per evidenziare le variazioni)
im.multiframe(1, 3)
plot(bsi_pre,  main = "BSI 2005", col = viridis::viridis(100), range = c(-1, 1))
plot(bsi_post, main = "BSI 2022", col = viridis::viridis(100), range = c(-1, 1))
plot(diff_bsi, main = "Differenza BSI", col = viridis::inferno(100), range = c(-0.2, 0.2))
dev.off()

# Commento: valori più alti di BSI indicano maggiore presenza di suolo nudo/esposto.
# Qui la differenza è calcolata come (pre - post): un valore positivo significa che
# il BSI era più alto nel 2005 rispetto al 2022, cioè che il suolo nudo si è ridotto,
# in linea con l'obiettivo di rinverdimento della Muraglia Verde.

# =====================================================================
# 9. INDICE SAVI (Soil Adjusted Vegetation Index) E CLASSIFICAZIONE
# =====================================================================
# Il SAVI corregge l'influenza della riflettanza del suolo chiaro, ideale per zone aride.
# Introduciamo il fattore L = 0.5 (correzione standard)
L <- 0.5
savi_pre <- ((pre[[4]] - pre[[3]]) / (pre[[4]] + pre[[3]] + L)) * (1 + L)
savi_post <- ((post[[4]] - post[[3]]) / (post[[4]] + post[[3]] + L)) * (1 + L)
diff_savi <- savi_post - savi_pre

# Visualizzazione
par(mfrow=c(1,3))
plot(savi_pre, main = "SAVI 2005", col = viridis::viridis(100), range = c(0, 1))
plot(savi_post, main = "SAVI 2022", col = viridis::viridis(100), range = c(0, 1))
plot(diff_savi, main = "Differenza SAVI", col = viridis::inferno(100), range = c(-0.3, 0.3))
dev.off()

# --- CLASSIFICAZIONE BINARIA SAVI ---
# Essendo un indice depurato dal rumore del suolo, abbassiamo la soglia a 0.2
soglia_savi <- 0.2
classi_savi_pre <- classify(savi_pre, rcl=matrix(c(-Inf, soglia_savi, 0, soglia_savi, Inf, 1), ncol=3, byrow=TRUE))
classi_savi_post <- classify(savi_post, rcl=matrix(c(-Inf, soglia_savi, 0, soglia_savi, Inf, 1), ncol=3, byrow=TRUE))

# Mappa di cambiamento (post - pre): con classi 0/1 i valori possibili sono
# -1 = vegetazione persa (era 1, ora 0), 0 = nessun cambiamento di classe,
# +1 = vegetazione guadagnata (era 0, ora 1)
diff_classi_savi <- classi_savi_post - classi_savi_pre

# Visualizzazione delle tre mappe di classificazione SAVI
im.multiframe(1, 3)
plot(classi_savi_pre,  main = "Classi SAVI 2005", col = c("orange", "darkgreen"))
plot(classi_savi_post, main = "Classi SAVI 2022", col = c("orange", "darkgreen"))
plot(diff_classi_savi, main = "Cambiamento classi SAVI (2005 → 2022)", col = c("firebrick3", "grey85", "darkgreen"))
dev.off()

# Commento: nelle prime due mappe l'arancione indica suolo/aree degradate (SAVI < soglia_savi)
# e il verde scuro la vegetazione. Nella mappa di cambiamento, il verde scuro (+1) individua le
# zone di nuovo rinverdimento (da suolo a vegetazione), il rosso (-1) le zone di vegetazione persa
# (da vegetazione a suolo), il grigio (0) le aree rimaste nella stessa classe (sia essa suolo o
# vegetazione) tra il 2005 e il 2022.

freq_savi_pre <- freq(classi_savi_pre)
freq_savi_post <- freq(classi_savi_post)

perc_savi_pre <- freq_savi_pre$count * 100 / ncell(classi_savi_pre)
perc_savi_post <- freq_savi_post$count * 100 / ncell(classi_savi_post)

tabella_savi <- data.frame(
  Classe = c("Suolo/Aree degradate", "Vegetazione SAVI"),
  Pre_Muraglia = round(perc_savi_pre, 2),
  Post_Muraglia = round(perc_savi_post, 2)
)
print("--- STATISTICHE PERCENTUALI SAVI --- 📝")
print(tabella_savi)

# =====================================================================
# 10. CONFRONTO GRAFICO DELLE PERCENTUALI (NDVI E SAVI)
# =====================================================================
# Il pacchetto reshape2, caricato all'inizio, serve proprio qui: lo usiamo per
# trasformare le tabelle riassuntive da formato "wide" (una colonna per periodo)
# a formato "long" (una riga per ogni combinazione classe-periodo), che è il
# formato richiesto da ggplot2 per raggruppare le barre in base al periodo.
tabella_ndvi_long <- melt(tabella_ndvi, id.vars = "Classe",
                          variable.name = "Periodo", value.name = "Percentuale")

tabella_savi_long <- melt(tabella_savi, id.vars = "Classe",
                          variable.name = "Periodo", value.name = "Percentuale")

# Grafico a barre NDVI: confronto Pre/Post per ciascuna classe
ggplot(tabella_ndvi_long, aes(x = Classe, y = Percentuale, fill = Periodo)) +
  geom_bar(stat = "identity", position = "dodge") +
  geom_text(aes(label = round(Percentuale, 1)),
            position = position_dodge(width = 0.9), vjust = -0.25, size = 3) +
  scale_fill_viridis_d() +
  ylim(0, 100) +
  labs(title = "Classificazione NDVI: confronto Pre/Post Muraglia Verde",
       y = "% area", x = NULL, fill = "Periodo") +
  theme_minimal()

# Grafico a barre SAVI: confronto Pre/Post per ciascuna classe
ggplot(tabella_savi_long, aes(x = Classe, y = Percentuale, fill = Periodo)) +
  geom_bar(stat = "identity", position = "dodge") +
  geom_text(aes(label = round(Percentuale, 1)),
            position = position_dodge(width = 0.9), vjust = -0.25, size = 3) +
  scale_fill_viridis_d() +
  ylim(0, 100) +
  labs(title = "Classificazione SAVI: confronto Pre/Post Muraglia Verde",
       y = "% area", x = NULL, fill = "Periodo") +
  theme_minimal()

# Commento conclusivo: confrontando le percentuali di "Suolo nudo/degradato" contro
# "Vegetazione" tra il 2005 (Pre_Muraglia) e il 2022 (Post_Muraglia) in entrambe le
# tabelle, una diminuzione della quota di suolo nudo accompagnata da un aumento della
# quota di vegetazione supporterebbe l'ipotesi di un rinverdimento dell'area, in
# coerenza con quanto osservabile anche nelle mappe di differenza DVI, NDVI e BSI.
