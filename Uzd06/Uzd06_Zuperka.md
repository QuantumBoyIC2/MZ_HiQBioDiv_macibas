Uzd06_Zuperka
================
Marks

1)  Rasterizējiet Lauku atbalsta dienestā reģistrētos laukus, izveidojot
    klasi “100” (vai izmantojiet trešajā uzdevumā sagatavoto slāni, ja
    tas ir korekts); <br><br> Trešajā uzdevumā sagatavoju 10x10 slāni,
    kur ir tikai atzīmēts vai attiecīgajā šūnā ir lauku bloki vai nav
    (0 - nav, 100 - ir)

``` r
#Rasterizēšu lauka blokus un pārbaudīšu vai sakrīt ar references 10x10 rastru
LAD_sf <- st_as_sf(read_parquet("C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd03\\LAD_data\\LAD_data.parquet"))
Latvia_ref_raster_10x10 <- rast("C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd03\\HiQBioDiv_raster_reference_grids\\LV10m_10km.tif")

#Kā jau pierasts izmantošu apgrieztu references rastru
bbox <- ext(st_bbox(LAD_sf))
Latvia_ref_raster_10x10_crop <- crop(Latvia_ref_raster_10x10, bbox)

Latvia_ref_raster_10x10_crop[!is.na(Latvia_ref_raster_10x10_crop)] <- 0

#Izmēģināšu LAD_raster ar touches=FALSE un touches=TRUE (paskatīšos kāda ir šūnu skaita atšķirība)        
LAD_raster_10x10 <- terra::rasterize(vect(LAD_sf), Latvia_ref_raster_10x10_crop, update=TRUE, touches=FALSE) 
cat("Ar touches=FALSE šūnu skaits ar vērtību 1 ir:", global(LAD_raster_10x10 == 1, fun="sum", na.rm=TRUE)[,1])
```

    ## Ar touches=FALSE šūnu skaits ar vērtību 1 ir: 2880902

``` r
LAD_raster_10x10_true <- terra::rasterize(vect(LAD_sf), Latvia_ref_raster_10x10_crop, update=TRUE, touches=TRUE) 
cat("Ar touches=TRUE šūnu skaits ar vērtību 1 ir:", global(LAD_raster_10x10_true == 1, fun="sum", na.rm=TRUE)[,1])
```

    ## Ar touches=TRUE šūnu skaits ar vērtību 1 ir: 3115323

``` r
#Salīdzinu ar to ko darīju iepriekš - fasterize. Šis beigās lai dabūtu pareizu rastru ir vēl jāsavieto ar references rastru, jo funkcijā neļauj ielikt iekšā update. Izmantošu rasterize tālāk lai ietaupītu laiku uz merge'ošanu.   
LAD_raster_10x10_fast <- rast(fasterize(LAD_sf, raster(Latvia_ref_raster_10x10_crop), background=NA))
cat("Ar fasterize šūnu skaits ar vērtību 1 ir:", global(LAD_raster_10x10_fast == 1, fun="sum", na.rm=TRUE)[,1])
```

    ## |---------|---------|---------|---------|=========================================                                          Ar fasterize šūnu skaits ar vērtību 1 ir: 2880902

``` r
rm(LAD_raster_10x10_true, LAD_raster_10x10_fast)

LAD_raster_10x10 <- subst(LAD_raster_10x10, 1, 100)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
#Izvadei nav CRS, to pievienoju (kopēju references rastra CRS) 
terra::crs(LAD_raster_10x10) <- terra::crs(Latvia_ref_raster_10x10)

#Fasterize jau ievadīju references rastru, tāpēc tas sakrīt un neko citu darīt nevajag. Saglabāju diskā
writeRaster(LAD_raster_10x10, "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd06/LAD_raster.tif", datatype = "INT1U", overwrite = TRUE)
```

    ## |---------|---------|---------|---------|=========================================                                          

<br> Izskatās, ka `fasterize` darbojas kā `terra::rasterize` ar
touches=FALSE argumentu (kas rasterize ir default arguments).
Touches=TRUE piešķir 1 arī tai šūnai, kuras centroīds ir ārpus poligona,
pat ja tikai neliela daļa no potenciālās šūnas ir tajā iekšā. <br>
Domājot par trīs metožu atšķirību ekoloģiski es teiktu ka touhes=FALSE
būtu pareizā pieeja. Ja mēs izmantojam touches=TRUE, tad mēs uzskatam ka
neeksistē malas efekts un viens biotops bez robežām pāriet citā, jo par
poligonam atbilstošu biotopu uzskatam arī to, kas atrodas reāli ārpus
tā. Daudz mazāka bēda es domāju ir biotopu “nerasterizēt līdz galam”
(resp. izmantojot FALSE sanāks ka kaut kāda daļa biotopa netiks
uzskatīta par tādu), jo tā daļa kas nebūs rasterizēta tāpat būs poligona
malās, un to mēs varam norakstīt uz malas efektu. <br> Līdz ar to
turpmāk palikšu pie fasterize vai rasterize bez touches argumenta
izmainīšanas. <br> 2. Izveidojiet rastra slāņus ar skujkoku (klase
“204”), šaurlapju (klase “203”), platlapju (klase “202”) un jauktu koku
mežiem (klase “201”) no sevis ierosinātās klasifikācijas otrajā
uzdevumā. <br><br> Otrā uzdevuma klasifikācija nebija laba, tāpēc to
pārtaisu, bet izmantoju daļu no otrā uzdevuma komandrindām.

``` r
Combined_centrs <- st_as_sf(arrow::read_parquet("C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd02\\Combined_centrs.parquet"))

#Sadalīšu datus 2 daļās (vienā ģeometrija un unikālais lauks, otrā - dati). Strādājot tikai ar datiem vajadzētu būt ātrāk
Combined_centrs_geom <- Combined_centrs[c("id","geometry")]
Combined_centrs_dati <- st_drop_geometry(Combined_centrs)

rm(Combined_centrs)

#Uztaisīšu tabulu, kur saglabāšu koku un tipu kuram tas pieder
kodu_tabula <- data.frame(
  KODS = c(1, 3, 4, 6, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 32, 35, 61, 62, 63, 64, 65, 66, 67, 68, 69),
  NOSAUKUMS = c("Skujkoku", "Skujkoku", "Šaurlapju", "Šaurlapju", "Šaurlapju", "Šaurlapju", 
                "Platlapju", "Platlapju", "Platlapju", "Skujkoku", "Skujkoku", "Skujkoku", 
                "Platlapju", "Platlapju", "Platlapju", "Šaurlapju", "Šaurlapju", "Šaurlapju", 
                "Skujkoku", "Skujkoku", "Platlapju", "Šaurlapju", "Šaurlapju", "Šaurlapju", 
                "Skujkoku", "Skujkoku", "Šaurlapju", "Šaurlapju", "Platlapju", "Platlapju", 
                "Platlapju", "Platlapju", "Platlapju", "Šaurlapju", "Šaurlapju", "Šaurlapju", 
                "Šaurlapju"))

#For cikla veidā:
for (k in 10:14) {
  jaunas_kolonnas_nosaukums <- paste0("k_s", k) #Izveidošu jaunu k-tās kolonnas nosaukumu

  k_s_kolonna <- Combined_centrs_dati[[paste0("s", k)]] #Izveidoju k-to kolonnu kā lielu skaitļu vektoru no attiecīgās sugu kolonnas, kuru pēc tam iebaroju sapply funkcijā (tam vajag vektoru)
  
  Combined_centrs_dati[[jaunas_kolonnas_nosaukums]] <- sapply(k_s_kolonna, function(kodesana) #Izmantojot sapply izpildu funkciju katrai sugu kolonnas vektora vērtībai 
    {
    if (kodesana %in% kodu_tabula$KODS) #Pārbaudu vai k_s vektora vērtība atbilst kādai kodu tabulas vērtībai (resp. vai nav 0 - nav datu)
      {
      return(kodu_tabula$NOSAUKUMS[kodu_tabula$KODS == kodesana])} #Ja atbilst, tad pievienoju nosaukumu
    } #Ja ir 0 un nav datu par koku sugu, tad atgriezīs NULL
  )
} #Rezultātā k_s10, k_s11, k_s12, k_s13 un k_s14 ir informācija par to kādam koku tipam atbilst pirmā stāva k-tā suga.

#Tālāk veidoju jaunas kolonnas, kurā būs summa kāds ir attiecīgā koku tipa aizņemtais šķērslaukums
Combined_centrs_dati$Shaur_proc <- apply(Combined_centrs_dati, 1, function(summesana) #Tagad izmantošu apply ar argumentu 1, kas pielietos funkciju "summesana" katrai tabulas rindiņai (sapply vajag ciparu virkni un kad vajad summēt vairākas vērtības būtu grūti strādāt ar ciparu virkni)
  {
  g_summa <- 0 #Izveidoju sākotnējo vērtību summēšanai
  for (k in 10:14) {
    if (summesana[paste0("k_s", k)] == "Šaurlapju") #Nosacījums, ka g kolonnu summa būs atgriezta tikai, ja attiecīgā vērtība k_s ir Šaurlapju 
      {
      g_summa <- g_summa + as.numeric(summesana[paste0("g", k)]) #Nu jā, otrajā uzdevumā visur g vietā bija a - ir jāpārskata LLM ražojumus...
    }
  }
  return(g_summa)
})

#Tas pats ar platlapju kokiem
Combined_centrs_dati$Plat_proc <- apply(Combined_centrs_dati, 1, function(summesana) {
  g_summa <- 0
  for (k in 10:14) {
    if (summesana[paste0("k_s", k)] == "Platlapju") {
      g_summa <- g_summa + as.numeric(summesana[paste0("g", k)])
    }
  }
  return(g_summa)
})


#Tas pats ar skujkokiem
Combined_centrs_dati$Skuj_proc <- apply(Combined_centrs_dati, 1, function(summesana) {
  g_summa <- 0
  for (k in 10:14) {
    if (summesana[paste0("k_s", k)] == "Skujkoku") {
      g_summa <- g_summa + as.numeric(summesana[paste0("g", k)])
    }
  }
  return(g_summa)
})
```

<br> Pirms veicu klasifikāciju, skaidri definēju savas robežvērtības
(izmantošu otrā uzdevumā pirms tam minēto pieejo priedēm): <br>\*
Platlapju mežs - mežs, kurā platlapju šķērslaukums veido vismaz 75%
(ieskaitot) no kopējā koku šķērslaukuma. <br><br>\* Skujkoku mežs -
mežs, kurā skujkoku šķērslaukums veido vismaz 75% (ieskaitot) no kopējā
koku šķērslaukuma.  
<br><br>\* Šaurlapju mežs - mežs, kurā šaurlapju šķērslaukums veido
vismaz 75% (ieskaitot) no kopējā koku šķērslaukuma. <br><br>\* Jaukts
mežs - pārējie meži (jebkuras koku sugas šķērslaukums ir zem 75%
(neieskaitot).  
\<br<br>nav datu - ja nav noteikts meža tips un nav aizpildītas koku
seguma vērtības. <br> Pirmais solis - apzīmēšu tos mežus, par kuriem
nepietiek datu s un g kolonnās lai tos klasificētu.

``` r
#Izveidoju kolonnu kurā būs koku šķērslaukumu summa
Combined_centrs_dati$Koki_summa <- rowSums(Combined_centrs_dati[, c("Skuj_proc", "Shaur_proc", "Plat_proc")])

Shaur_proc <- Combined_centrs_dati$Shaur_proc
Plat_proc <- Combined_centrs_dati$Plat_proc 
Skuj_proc <- Combined_centrs_dati$Skuj_proc

Combined_centrs_dati$Meza_klase <- "turpinam klasificēt" #Sākotnējā kolonnas vērtība, kuru tālāk mainīšu klasifikācijā

#Ir diezgan daudz mežu, kur nav vērtību nedz par to kāds ir koku šķērslaukums, nedz kādi koki tur ir - šos izdalīšu jau sākumā
Combined_centrs_dati$Meza_klase <- apply(Combined_centrs_dati, 1, function(nezinu_mezs) {
  
  Skuj_proc <- as.numeric(nezinu_mezs["Skuj_proc"])
  Plat_proc <- as.numeric(nezinu_mezs["Plat_proc"])
  Shaur_proc <- as.numeric(nezinu_mezs["Shaur_proc"])
  
  if ((Skuj_proc == 0 && Plat_proc == 0 && Shaur_proc == 0) ||
      nezinu_mezs["mt"] == 0) {
    return("klasificēt citādi")
  } else {
    return(nezinu_mezs[["Meza_klase"]])
  }
})
```

<br> Tālāk klasificēšu mežus pēc 75% kritērija.

``` r
#Izveidošu filtru, lai funkcija nepārklasificē esošos datus
filtrs <- Combined_centrs_dati$Meza_klase == "turpinam klasificēt"

Combined_centrs_dati$Meza_klase[filtrs] <- apply(Combined_centrs_dati[filtrs, ], 1, function(def1) 
  {
  Skuj_proc <- as.numeric(def1["Skuj_proc"])
  Plat_proc <- as.numeric(def1["Plat_proc"])
  Shaur_proc <- as.numeric(def1["Shaur_proc"])
  
  summa <- Skuj_proc + Plat_proc + Shaur_proc
  
  if (Skuj_proc >= summa * 0.75) {
    return("Skujkoku mežs")
  } else if (Plat_proc >= summa * 0.75) {
    return("Platlapju mežs")
  } else if (Shaur_proc >= summa * 0.75) {
    return("Šaurlapju mežs")
  } else {
    return("Jauktu koku mežs")
  }
})

cat("Jaukto mežu poligonu skaits", sum(Combined_centrs_dati$Meza_klase == "Jauktu koku mežs"))
```

    ## Jaukto mežu poligonu skaits 62847

``` r
cat("Platlapju mežu poligonu skaits", sum(Combined_centrs_dati$Meza_klase == "Platlapju mežs"))
```

    ## Platlapju mežu poligonu skaits 4082

``` r
cat("Skujkoku mežu poligonu skaits", sum(Combined_centrs_dati$Meza_klase == "Skujkoku mežs"))
```

    ## Skujkoku mežu poligonu skaits 148046

``` r
cat("Šaurlapju mežu poligonu skaits", sum(Combined_centrs_dati$Meza_klase == "Šaurlapju mežs"))
```

    ## Šaurlapju mežu poligonu skaits 141630

<br> Tālāk izveidošu rastra slāņus ar klasēm. Iepriekšējā uzdevumā
noteicu, ka ātrākais variants bija strādāt ar visu failu un tad beigās
izgriezt rezultātu, bet šoreiz tas nestrādās, jo bbox visiem 4 rastriem
būs aptuveni vienāds. Nākamais ātrākais variants bija bez for
izmantošanas, kopējot komandas un tikai izmainot argumentus, to arī
daru.

``` r
#Izgriežu ref rastru jau pēc Centra mežniecības datu ģeometrijas
Latvia_ref_raster_10x10_crop <- terra::crop(Latvia_ref_raster_10x10, ext(st_bbox(Combined_centrs_geom)))
Latvia_ref_raster_10x10_crop[!is.na(Latvia_ref_raster_10x10_crop)] <- 0 
```

    ## |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          

``` r
#Pievienošu ģeometriju atpakaļ
Combined_centrs <- st_as_sf(dplyr::left_join(Combined_centrs_dati, Combined_centrs_geom, by = "id"))

rm(Combined_centrs_dati, Combined_centrs_geom)

##204
filtrs <- Combined_centrs[Combined_centrs$Meza_klase == "Skujkoku mežs", ]
r204 <- terra::rasterize(vect(filtrs), Latvia_ref_raster_10x10_crop, update=TRUE)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
r204 <- subst(r204, 1, 204, raw=TRUE)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
writeRaster(r204, "C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd06\\Centrs_204_vecums.tif", datatype = "INT1U", overwrite=TRUE)


##203
filtrs <- Combined_centrs[Combined_centrs$Meza_klase == "Šaurlapju mežs", ]
r203 <- terra::rasterize(filtrs, Latvia_ref_raster_10x10_crop, update=TRUE)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
r203 <- subst(r203, 1, 203, raw=TRUE)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
writeRaster(r203, "C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd06\\Centrs_203_vecums.tif", datatype = "INT1U", overwrite=TRUE)


##202
filtrs <- Combined_centrs[Combined_centrs$Meza_klase == "Platlapju mežs", ]
r202 <- terra::rasterize(filtrs, Latvia_ref_raster_10x10_crop, update=TRUE)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
r202 <- subst(r202, 1, 202, raw=TRUE)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
writeRaster(r202, "C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd06\\Centrs_202_vecums.tif", datatype = "INT1U", overwrite=TRUE)


##201
filtrs <- Combined_centrs[Combined_centrs$Meza_klase == "Jauktu koku mežs", ]
r201 <- terra::rasterize(filtrs, Latvia_ref_raster_10x10_crop, update=TRUE)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
r201 <- subst(r201, 1, 201, raw=TRUE)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
writeRaster(r201, "C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd06\\Centrs_201_vecums.tif", datatype = "INT1U", overwrite=TRUE)

gc()
```

    ##            used  (Mb) gc trigger   (Mb)  max used   (Mb)
    ## Ncells  8419016 449.7   52007226 2777.5  65009032 3471.9
    ## Vcells 78060270 595.6  237483087 1811.9 724740862 5529.4

<br> Tālāk izmainīšu rastra vērtību atbilstoši mežaudzes vecumam. <br>
Mežaudžu datos nav vienotas vērtības par to kāds ir kopējais mežaudzes
vecums. DBF specifikācijā eksporta failā ir aile “VGR”, kurā ir
mežaudzes iedalījums 5 klasēs un kurš sakrīt ar statistikas portāla
klasifikāciju (vismaz nosaukumos). Izmantošu [Oficiālā statistikas
portāla vecuma klašu definējumu atkarībā no
koka](https://stat.gov.lv/lv/statistikas-temas/noz/mezsaimnieciba/tabulas/mem050-mezaudzu-vecuma-struktura-gada-sakuma-tukst-ha).

``` r
kodu_tabula <- kodu_tabula %>%
  mutate(suga = case_when(
    KODS == "1" ~ "Priede",
    KODS == "3" ~ "Egle",
    KODS == "4" ~ "Bērzs",
    KODS == "6" ~ "Melnalksnis",
    KODS == "9" ~ "Baltalksnis",
    KODS == "8" ~ "Apse",
    KODS == "10" ~ "Ozols",
    KODS == "11" ~ "Osis",
    TRUE ~ "Citas sugas"
  ))

#Tālāk veidošu jaunas kolonnas suga_s10 - suga_s14, kurā būs rakstīts kāda ir suga katrā stāvā (ērtāk pārbaudīt rezultātus vēlāk kad viņi ir atsevišķās kolonnas, un neaiztikt izejas datus)
#For cikla veidā:
for (k in 10:14) {
  jaunas_kolonnas_nosaukums <- paste0("suga_s", k)
  suga_s_kolonna <- Combined_centrs[[paste0("s", k)]]
  
  Combined_centrs[[jaunas_kolonnas_nosaukums]] <- kodu_tabula$suga[match(suga_s_kolonna, kodu_tabula$KODS)]
  #Izmantoju match - sanāk ātrāk nekā izmantot if() funkciju
}

#Izveidošu jaunas kolonnas, kurās būs šo koku vecums(pārskatāmāk nekā kad a kolonna ir samētāta citos nevajadzīgos datos) un uztaisīšu atsevišķu nelielu tabulu (tas pats princips, kas kad dalīju atsevišķi ģeometriju un datus)

Combined_centrs_dati <- st_drop_geometry(Combined_centrs[c("id", "suga_s10", "suga_s11", "suga_s12", "suga_s13", "suga_s14")])

Combined_centrs_dati$vecums_s10 <- Combined_centrs$a10
Combined_centrs_dati$vecums_s11 <- Combined_centrs$a11
Combined_centrs_dati$vecums_s12 <- Combined_centrs$a12
Combined_centrs_dati$vecums_s13 <- Combined_centrs$a13
Combined_centrs_dati$vecums_s14 <- Combined_centrs$a14

Combined_centrs_dati$laukums_s10 <- Combined_centrs$g10
Combined_centrs_dati$laukums_s11 <- Combined_centrs$g11
Combined_centrs_dati$laukums_s12 <- Combined_centrs$g12
Combined_centrs_dati$laukums_s13 <- Combined_centrs$g13
Combined_centrs_dati$laukums_s14 <- Combined_centrs$g14
```

<br> Tālāk definēšu kāds vecums atbilst kurai vecuma grupai katram
kokam - to pievienošu savai kodu_tabula (tabulā būs max. vecuma vērtība
attiecīgai sugai)

<br> Jaunaudzes: *priedei, eglei, ozolam, osim: 1-40 g.; *bērzam,
melnalksnim, apsei un citām sugām: 1-20 g; \*baltalksnim - 1-10 g.

<br><br>Vidēja vecuma audzes: *priedei, ozolam: 41 - 80 g.; *eglei,
osim: 41 - 60 g.; *bērzam, melnalksnim un citām sugām: 21-60 g.; *Apsei:
20-30 g.; \*Baltalksnim: 10-20 g.

<br><br>Briestaudzes: *Priedei, ozolam: 80 - 100 g.; *Eglei, Osim un
citām cugām: 60 - 80 g.; *Bērzam, melnalksnim: 60 - 70. g; *Apsei: 30-40
g. \*Baltalksnim: 20 - 30 g.

<br><br> Meža reģistra datos pāraugušās audzes ir klasificētas
atsevišķi, bet statistikā tās ir kopā ar pieaugšām. Domāju, ka
ekoloģiski korekti būtu izdalīt pāraugušās audzes atsevišķi. Tās
izdalīšu pēc principa, ka lēni un vidēji ātri augošām koku sugām
(priede, egle, ozols, osis, citas sugas) pieaugšās audzes stadija ilgs
20 gadus, bet ātri augošām sugām (bērzs, melnalksnis, baltalksnis,
apse) - 10 gadus.

<br><br>Pieaugušās audzes: *Priede, ozols: 100 - 120 g.; *Egle, Osis,
citas sugas: 80 - 100 g.; *Bērzs, melnalksnis: 70 - 90 g.; *Baltalksnis:
30-40 g.; \*Apse: 40 - 50 g.

<br><br> Pāraugušās audzes - definēšu dinamiski pēc analizējamās meža
datu kopas vecuma kolonnu maksimālās vērtības (nodrošināšu to, ka
rezultāts ir replicējams jebkurai reģistra datu kopai)

<br> Tālāk pievienošu jaunas kolonnas, kurās noteikšu kādai vecuma
grupai atbilst s_tās kolonnas koks

``` r
#Uztaisīšu klasificēšanas funkciju koku grupām
priede_ozols <- function(Combined_centrs_dati, sugas_kolonna, vecuma_kolonna, jauna_kolonna) {
  
  Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(jauna_kolonna = if_else((sugas_kolonna == "Priede" | sugas_kolonna == "Ozols") & (vecuma_kolonna > 0 & vecuma_kolonna <= 40), "Jaunaudze", 
  if_else((sugas_kolonna == "Priede" | sugas_kolonna == "Ozols") & (vecuma_kolonna > 40 & vecuma_kolonna <= 80), "Vidēja vecuma audze",
  if_else((sugas_kolonna == "Priede" | sugas_kolonna == "Ozols") & (vecuma_kolonna > 80 & vecuma_kolonna <= 100), "Briestaudze",
  if_else((sugas_kolonna == "Priede" | sugas_kolonna == "Ozols") & (vecuma_kolonna > 100 & vecuma_kolonna <= 120), "Pieaugusi audze",
  if_else((sugas_kolonna == "Priede" | sugas_kolonna == "Ozols") & (vecuma_kolonna > 120), "Pāraugusi audze",
          "NA"
))))))}

egle_osis <- function(Combined_centrs_dati, sugas_kolonna, vecuma_kolonna, jauna_kolonna) {
  
  Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(jauna_kolonna = case_when(
    jauna_kolonna == "NA" & (sugas_kolonna == "Egle" | sugas_kolonna == "Osis") & (vecuma_kolonna > 0 & vecuma_kolonna <= 40) ~ "Jaunaudze", 
    jauna_kolonna == "NA" & (sugas_kolonna == "Egle" | sugas_kolonna == "Osis") & (vecuma_kolonna > 40 & vecuma_kolonna <= 60) ~ "Vidēja vecuma audze",
    jauna_kolonna == "NA" & (sugas_kolonna == "Egle" | sugas_kolonna == "Osis") & (vecuma_kolonna > 60 & vecuma_kolonna <= 80) ~ "Briestaudze",
    jauna_kolonna == "NA" & (sugas_kolonna == "Egle" | sugas_kolonna == "Osis") & (vecuma_kolonna > 80 & vecuma_kolonna <= 100) ~ "Pieaugusi audze",
    jauna_kolonna == "NA" & (sugas_kolonna == "Egle" | sugas_kolonna == "Osis") & (vecuma_kolonna > 100) ~ "Pāraugusi audze",
    TRUE ~ jauna_kolonna
  ))}

citas_sugas <- function(Combined_centrs_dati, sugas_kolonna, vecuma_kolonna, jauna_kolonna) {
  
  Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(jauna_kolonna = case_when(
    jauna_kolonna == "NA" & sugas_kolonna == "Citas sugas" & (vecuma_kolonna > 0 & vecuma_kolonna <= 20) ~ "Jaunaudze", 
    jauna_kolonna == "NA" & sugas_kolonna == "Citas sugas" & (vecuma_kolonna > 20 & vecuma_kolonna <= 60) ~ "Vidēja vecuma audze",
    jauna_kolonna == "NA" & sugas_kolonna == "Citas sugas" & (vecuma_kolonna > 60 & vecuma_kolonna <= 80) ~ "Briestaudze",
    jauna_kolonna == "NA" & sugas_kolonna == "Citas sugas" & (vecuma_kolonna > 80 & vecuma_kolonna <= 100) ~ "Pieaugusi audze",
    jauna_kolonna == "NA" & sugas_kolonna == "Citas sugas"  & (vecuma_kolonna > 100) ~ "Pāraugusi audze",
    TRUE ~ jauna_kolonna
  ))}


berzs_melnalksnis <- function(Combined_centrs_dati, sugas_kolonna, vecuma_kolonna, jauna_kolonna) {

  Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(jauna_kolonna = case_when(
    jauna_kolonna == "NA" & (sugas_kolonna == "Bērzs" | sugas_kolonna == "Melnalksnis") & (vecuma_kolonna > 0 & vecuma_kolonna <= 20) ~ "Jaunaudze", 
    jauna_kolonna == "NA" & (sugas_kolonna == "Bērzs" | sugas_kolonna == "Melnalksnis") & (vecuma_kolonna > 20 & vecuma_kolonna <= 60) ~ "Vidēja vecuma audze",
    jauna_kolonna == "NA" & (sugas_kolonna == "Bērzs" | sugas_kolonna == "Melnalksnis") & (vecuma_kolonna > 60 & vecuma_kolonna <= 70) ~ "Briestaudze",
    jauna_kolonna == "NA" & (sugas_kolonna == "Bērzs" | sugas_kolonna == "Melnalksnis") & (vecuma_kolonna > 70 & vecuma_kolonna <= 80) ~ "Pieaugusi audze",
    jauna_kolonna == "NA" & (sugas_kolonna == "Bērzs" | sugas_kolonna == "Melnalksnis") & (vecuma_kolonna > 80) ~ "Pāraugusi audze",
    TRUE ~ jauna_kolonna
  ))}


baltalksnis <- function(Combined_centrs_dati, sugas_kolonna, vecuma_kolonna, jauna_kolonna) {
  
  Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(jauna_kolonna = case_when(
    jauna_kolonna == "NA" & sugas_kolonna == "Baltalksnis" & (vecuma_kolonna > 0 & vecuma_kolonna <= 10) ~ "Jaunaudze", 
    jauna_kolonna == "NA" & sugas_kolonna == "Baltalksnis" & (vecuma_kolonna > 10 & vecuma_kolonna <= 20) ~ "Vidēja vecuma audze",
    jauna_kolonna == "NA" & sugas_kolonna == "Baltalksnis" & (vecuma_kolonna > 20 & vecuma_kolonna <= 30) ~ "Briestaudze",
    jauna_kolonna == "NA" & sugas_kolonna == "Baltalksnis" & (vecuma_kolonna > 40 & vecuma_kolonna <= 40) ~ "Pieaugusi audze",
    jauna_kolonna == "NA" & sugas_kolonna == "Baltalksnis"  & (vecuma_kolonna > 40) ~ "Pāraugusi audze",
    TRUE ~ jauna_kolonna
  ))}


apse <- function(Combined_centrs_dati, sugas_kolonna, vecuma_kolonna, jauna_kolonna) {

  Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(jauna_kolonna = case_when(
    jauna_kolonna == "NA" & sugas_kolonna == "Apse" & (vecuma_kolonna > 0 & vecuma_kolonna <= 20) ~ "Jaunaudze", 
    jauna_kolonna == "NA" & sugas_kolonna == "Apse" & (vecuma_kolonna > 20 & vecuma_kolonna <= 30) ~ "Vidēja vecuma audze",
    jauna_kolonna == "NA" & sugas_kolonna == "Apse" & (vecuma_kolonna > 30 & vecuma_kolonna <= 40) ~ "Briestaudze",
    jauna_kolonna == "NA" & sugas_kolonna == "Apse" & (vecuma_kolonna > 40 & vecuma_kolonna <= 50) ~ "Pieaugusi audze",
    jauna_kolonna == "NA" & sugas_kolonna == "Apse"  & (vecuma_kolonna > 50) ~ "Pāraugusi audze",
    TRUE ~ jauna_kolonna
  ))}
 
#Tālāk izpildīšu funkciju katrai kolonnai no s10 līdz s14 (sugu nosaukumiem). Definēšu funkciju sarakstu un tad izpildīšu visas funkcijas vienai kolonnai un izdarīšu to pašu ar pārējām. 

funkciju_saraksts <- list(priede_ozols, egle_osis, citas_sugas, berzs_melnalksnis, baltalksnis, apse) 

for (i in 10:14) {
  sugas_kolonna <- paste0("suga_s", i)
  vecuma_kolonna <- paste0("vecums_s", i)
  jauna_kolonna <- paste0("v_gr_s", i)
  
  for (funkcija in funkciju_saraksts) {
    Combined_centrs_dati <- funkcija(Combined_centrs_dati, 
                                     Combined_centrs_dati[[sugas_kolonna]], 
                                     Combined_centrs_dati[[vecuma_kolonna]])
  }

  Combined_centrs_dati <- Combined_centrs_dati %>% rename(!!jauna_kolonna := jauna_kolonna)
}
```

<br> Kā redzu atšķiras klasificētais vecums. Lai piešķirtu vienu vecumu
visam poligonam, izmantošu šādu pieeju: izsvarošu katras kolonnas
nozīmīgumu vecuma klases noteikšanā pēc koka aizņemtā šķērslaukuma un
gala nosaukumu piešķiršu pēc izsvarotās vērtības. Vecuma grupas
piešķiršanā izmantošu tās vecuma grupas vērtību, kuras kopējais segums
ir lielākais. Svarošana nodrošinās ar to, ka piem. mežs, kurā apse
(šķērslaukums 40) ir brieduma stadijā netiks nosaukts par nobriedušu, ja
audzē esošās egles (20) un priedes (30) ir jaunaudzes stadijā (tas tiks
klasificēts kā jaunaudze, jo jaunu koku šķērslaukums ir lielāks nekā
nobriedušu).

``` r
#Sasummēšu rindiņas  
Combined_centrs_dati$Laukums_sum <- rowSums(Combined_centrs_dati[, c("laukums_s10", "laukums_s11", "laukums_s12", "laukums_s13", "laukums_s14")])

#For cikla veidā dinamiski izveidošu jaunas kollonas, kurās atgriezīšu cik % no kopējā šķērslaukuma veido attiecīgais koks 
for (i in 10:14) {
  sugas_kolonna <- paste0("suga_s", i)
  laukuma_kolonna <- paste0("laukums_s", i)
  svara_kolonna <- paste0("svars_s", i)

  Combined_centrs_dati[[paste0("svars_s", i)]] <- round((Combined_centrs_dati[[laukuma_kolonna]] / Combined_centrs_dati$Laukums_sum * 100), 2)
  
  Combined_centrs_dati[[svara_kolonna]][is.nan(Combined_centrs_dati[[svara_kolonna]])] <- 0
}

#Tālāk veicu gala vecuma grupas klasifikāciju - to veikšu pēc tā paša principa, ko izmantoju kad skaitļoju mežu klasifikācijā cik ir skujkoku, platlapju un šaurlapju procenti un atgriezu atsevišķās kolonnās.

#Tikai pārrakstu to uz dplyr loģiku, jo tas strādā daudz ātrāk nekā apply un if(). To izdaru 5 reizes katram vecuma tipam.
#Jaunaudze
Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(Jaunaudze_svars = rowSums(
    sapply(10:14, function(kol) {
      sugas_kolonna <- paste0("v_gr_s", kol)
      svara_kolonna <- paste0("svars_s", kol)
      
      if_else(
        .[[sugas_kolonna]] == "Jaunaudze", 
        .[[svara_kolonna]], 
        0)
      })))

#Vidēja vecuma audze
Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(Vid_vec_svars = rowSums(
    sapply(10:14, function(kol) {
      sugas_kolonna <- paste0("v_gr_s", kol)
      svara_kolonna <- paste0("svars_s", kol)
      
      if_else(
        .[[sugas_kolonna]] == "Vidēja vecuma audze", 
        .[[svara_kolonna]], 
        0)
      })))

#Briestaudze
Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(Briest_svars = rowSums(
    sapply(10:14, function(kol) {
      sugas_kolonna <- paste0("v_gr_s", kol)
      svara_kolonna <- paste0("svars_s", kol)
      
      if_else(
        .[[sugas_kolonna]] == "Briestaudze", 
        .[[svara_kolonna]], 
        0)
      })))

#Pieaugusi audze
Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(Pieaug_svars = rowSums(
    sapply(10:14, function(kol) {
      sugas_kolonna <- paste0("v_gr_s", kol)
      svara_kolonna <- paste0("svars_s", kol)
      
      if_else(
        .[[sugas_kolonna]] == "Pieaugusi audze", 
        .[[svara_kolonna]], 
        0)
      })))

#Pāraugusi audze
Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(Paraug_svars = rowSums(
    sapply(10:14, function(kol) {
      sugas_kolonna <- paste0("v_gr_s", kol)
      svara_kolonna <- paste0("svars_s", kol)
      
      if_else(
        .[[sugas_kolonna]] == "Pāraugusi audze", 
        .[[svara_kolonna]], 
        0)
      })))
```

<br> Esmu ieguvis datus par to kāda ir koku vecuma proporcija katrā meža
poligonā. Meža vecuma grupas nosaukumu piešķiršu vienkārši pēc max.
vērtības. Uzreiz nokodēšu ciparos atsevišķā kolonnā rasterizēšanai pēc
tam (sakrīt ar DBF reģistra klasifikatoru)

``` r
Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(F_age = case_when(
    Jaunaudze_svars > pmax(Vid_vec_svars, Briest_svars, Pieaug_svars, Paraug_svars) ~ "Jaunaudze",
    Vid_vec_svars > pmax(Jaunaudze_svars, Briest_svars, Pieaug_svars, Paraug_svars) ~ "Vidēja vecuma audze",
    Briest_svars > pmax(Jaunaudze_svars, Vid_vec_svars, Pieaug_svars, Paraug_svars) ~ "Briestaudze",
    Pieaug_svars > pmax(Jaunaudze_svars, Vid_vec_svars, Briest_svars, Paraug_svars) ~ "Pieaugusi audze",
    Paraug_svars > pmax(Jaunaudze_svars, Vid_vec_svars, Briest_svars, Pieaug_svars) ~ "Pāraugusi audze",
    TRUE ~ "n.d."
  ))

Combined_centrs_dati <- Combined_centrs_dati %>%
  mutate(VGR = case_when(
    F_age == "Jaunaudze"~ 1,
    F_age == "Vidēja vecuma audze"~ 2,
    F_age == "Briestaudze"~ 3,
    F_age == "Pieaugusi audze"~ 4,
    F_age == "Pāraugusi audze"~ 5,
    TRUE ~ 0
  ))
```

<br> Var ķerties klāt rastru papildināšanai. No sākuma rasterizēšu sf
failu.

``` r
Combined_centrs <- left_join(Combined_centrs, Combined_centrs_dati, by = "id")
rm(Combined_centrs_dati)

#Rasterizēšu (paralēli nodrošināšu to, ka meža vecuma rastrs sakrīt ar mežu klasifikācijas rastriem)
VGR_rastrs <- rast(fasterize(Combined_centrs, raster(Latvia_ref_raster_10x10_crop), field = "VGR", background = NA))
sprc <- sprc(list(VGR_rastrs, Latvia_ref_raster_10x10_crop))
VGR_rastrs <- terra::merge(sprc)
```

    ## |---------|---------|---------|---------|=========================================                                          

<br> R vidē ir pieejama `Torch` pakotne, kas ļauj izmantot GPU
skaitļošanā. Un tā kā manam datoram ir uzstādīta RTX videokarte,
izmantošu to - pamēģināšu iegūt vajadzīgo rezultātu šādā veidā <br>
Ideja - izveidot 3 tensorus (kas glabājas GPU dRAM): vienā būs tikai
virknes pirmie cipari, otrajā - tikai trešais cipars, bet trešajā - VGR
vērtību rastrs. Tad ar ifelse analogu `torch_where`, ja VGR vektora
vērtība ir virs 0, ar vienkāršu matemātiku izveidot skaitli: pirmais
cipars \* 100 + otrais cipars \* 10 + trešais cipars. Ja nosacījumam
neatbilst, atgriezt references vērtību. Tad izfiltrēt tās vērtības,
kuras ir zem 211 (tās veidojas, jo VGR rastrs nav dalīts pēc mežu
klasēm). Pēc tam rezultāta tensoru konvertēt atpakaļ uz skaitļu virkni,
un izmantojot to pašu `values()`, izmainīt vērtības esošajā rastrā.
<br><br> Intereses pēc, šim uzdevumam veicu laika mērogošanu -
salīdzināšu cik ātri ir šo saskaitļot izmantojot GPU salīdzinājumā ja
izmantotu to pašu pieeju rēķinot CPU (`terra::ifelse`).

``` r
rastru_kolekcija <- list(r201 = r201, r202 = r202, r203 = r203, r204 = r204)

VGR_tensor <- torch_tensor(values(VGR_rastrs), dtype = torch_int32(), device = "cuda")

for (rastrs in names(rastru_kolekcija)) {
  r <- rastru_kolekcija[[rastrs]]
  
  nosaukums <- paste0(names(rastru_kolekcija[rastrs]), "_GPU")
  
  GPU1 <- Sys.time()
  
  tensor_first <- torch_tensor(values(r), dtype = torch_int32(), device = "cuda") %/% 100            
  tensor_third <- torch_tensor(values(r), dtype = torch_int32(), device = "cuda") %% 10       
  
  tensor_result <- torch_where(VGR_tensor > 0 & tensor_first == 2, 
                               tensor_first * 100 + VGR_tensor * 10 + tensor_third, 
                               0)
  
  tensor_result <- torch_where(tensor_result < 200, 0, tensor_result)
  
  tensor_result <- as.numeric(tensor_result)
  
  values(r) <- tensor_result
  
  GPU2 <- Sys.time() #Pēdējā faila mērogošanas laiku atgriežu tiklīdz iegūstu rastru ar vērtībām (noņemu laiku, ko aizņem terra::subst un merge, lai noformētu gala rastru)

  Starpiba_GPU <- difftime(GPU2, GPU1, units = "mins")

  r <- subst(r, 0, NA)
  r <- merge(r, Latvia_ref_raster_10x10_crop)
  crs(r) <- crs(Latvia_ref_raster_10x10)
  
  cuda_empty_cache()
  
  assign(nosaukums, r, envir = .GlobalEnv)
}
```

    ## |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          

``` r
cat("Izmantojot GPU pēdējo rastru izskaitļoju:", round(Starpiba_GPU, 1), "minūtēs \n")
```

    ## Izmantojot GPU pēdējo rastru izskaitļoju: 0.2 minūtēs

<br> Izmēģināšu to pašu loģiku, tikai skaitļojot ar `terra:ifel` (laikam
pieeja, kuru bija domāts izmantot šajā uzdevumā)

``` r
for (r in rastru_kolekcija) {
  r <- rastru_kolekcija[[rastrs]]
  
  nosaukums <- paste0(names(rastru_kolekcija[rastrs]), "_CPU")
 
  CPU1 <- Sys.time()
  
  ifelse_first <- r %/% 100
  ifelse_third <- r %% 10
  
  ifelse_result <- rep(0, length(ifelse_first))
  
  ifelse_result <- terra::ifel(VGR_rastrs > 0 & ifelse_first == 2, 
                        ifelse_first * 100 + VGR_rastrs * 10 + ifelse_third, 
                        ifelse_result)
  
  CPU2 <- Sys.time()

  Starpiba_CPU <- difftime(CPU2, CPU1, units = "mins")

  r <- subst(ifelse_result, 0, NA)
  r <- merge(r, Latvia_ref_raster_10x10_crop)
  crs(r) <- crs(Latvia_ref_raster_10x10)
  
  #Noņemu saglabāšanu, jo nevajag dublējošos failus
  #assign(nosaukums, r, envir = .GlobalEnv)
}
```

    ## |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          |---------|---------|---------|---------|=========================================                                          

``` r
cat("Izmantojot CPU pēdējo rastru izskaitļoju:", round(Starpiba_CPU, 1), "minūtēs \n")
```

    ## Izmantojot CPU pēdējo rastru izskaitļoju: 1.3 minūtēs

<br> 4. Savienojiet pirmajā un trešajā punktos izveidotos slāņus, tā,
lai mazāka skaitliskā vērtība nozīmē augstāku prioritāti/svaru/dominanci
rastra šūnas vērtības piešķiršanai. <br> Ar Andra atļauju uzdevumu
ietvaros to darīt - izmantoju GPU skaitļošanu arī nākamajā punktā.
Diezgan bieži iztīru atmiņu, jo 16 GB atmiņas, kas man ir pieejama torch
(8GB VRAM + 8GB shared), aizpildās ātri uztverot rastru kā matricu. <br>
Process: izveidošu no 5 rastriem vienu lielu tabulu, kur attiecīgi būs 4
kolonnas ar ievades rastriem un 250 milj. rindiņas ar katras rastra
šūnas vērtību attiecīgajā rastrā. Tad nomainīšu nulles ar kādu vērtību,
kas ir lielāka par max. meža poligona kodu, izmantošu 300. Tad ar
`torch_min` iziešu cauri katrai rindiņai un atgriezīšu tās minimālo
vērtību. Tā kā nulles ir aizvietotas, tagad, ja piem. lauku bloks sakrīt
ar meža poligonu, tiks atgriezta lauka bloka vērtība. Tāpat arī ar
mežiem - ja 201 rastra vērtība ir 211, bet 204 - 214, tiks ņemta 211, jo
tā ir mazākā. Tad savienošu ar references rastru, lai nodrošinātu ka
ārpus LV sauszemes teritorijas ir NA un nezināmās teritorijas ir 0.

``` r
#Iztīrīšu GPU atmiņu un pārsaukšu vajadzīgos failus
rm(tensor_first, tensor_third, VGR_tensor)

r201 <- r201_GPU
r202 <- r202_GPU
r203 <- r203_GPU
r204 <- r204_GPU

rm(r201_GPU, r202_GPU, r203_GPU, r204_GPU)
cuda_empty_cache()
gc()
```

    ##             used   (Mb) gc trigger   (Mb)  max used   (Mb)
    ## Ncells   8478754  452.9   26627700 1422.1  65009032 3471.9
    ## Vcells 342259545 2611.3  999303262 7624.1 831675659 6345.2

``` r
LAD_raster_10x10 <- crop(LAD_raster_10x10, Latvia_ref_raster_10x10_crop) #Jānodrošina, ka visiem ievades tensoriem ir vienāds šūnu skaits. 

#Izveidoju jaunus tensorus
r201_tensor <- torch_tensor(values(r201), dtype = torch_int32(), device = "cuda")
r202_tensor <- torch_tensor(values(r202), dtype = torch_int32(), device = "cuda")
r203_tensor <- torch_tensor(values(r203), dtype = torch_int32(), device = "cuda")
r204_tensor <- torch_tensor(values(r204), dtype = torch_int32(), device = "cuda")
LAD_tensor <- torch_tensor(values(LAD_raster_10x10), dtype = torch_int32(), device = "cuda")

torch_stack <- torch_stack(list(r201_tensor, r202_tensor, r203_tensor, r204_tensor, LAD_tensor)) #Savienoju tensorus, izveidojot vienu milzīgu tabulu (250 milj. kolonnu katrai rastra šūnai, 5 rindas katram failam)
torch_stack <- torch_stack$transpose(1, 2) #Domāju vai ir efektīvāk uzturēt 250 milj. kolonnas un 5 rindiņas vai otrādi. Sapratu, ka būtiskas atšķirības laikam nav? Bet lai paliek. 

rm(r201_tensor, r202_tensor, r203_tensor, r204_tensor, LAD_tensor)
gc()
```

    ##             used   (Mb) gc trigger   (Mb)  max used   (Mb)
    ## Ncells   8461090  451.9   26627700 1422.1  65009032 3471.9
    ## Vcells 342253158 2611.2  999303262 7624.1 831675659 6345.2

``` r
cuda_empty_cache()

tensor_result <- torch_where(torch_stack <= 0, 300, torch_stack) #Aizvietoju 0 ar 300

tensor_result <- torch_min(tensor_result, dim=2, keepdim=TRUE)
tensor_result <- tensor_result[[1]]
tensor_result <- tensor_result$transpose(1, 2)

tensor_result <- torch_where(tensor_result == 300, 0, tensor_result)

tensor_result <- as.numeric(tensor_result) 
  
apvienotais_rastrs <- Latvia_ref_raster_10x10_crop #Nodublēšu references rastru kuram iekšā likšu vērtības ko atgriež tensor_result
values(apvienotais_rastrs) <- tensor_result
rm(tensor_result, torch_stack) #Šie briesmoņi kopā aizņēma 14 GB RAM. 
gc()
```

    ##            used  (Mb) gc trigger   (Mb)  max used   (Mb)
    ## Ncells  8463535 452.1   26627700 1422.1  65009032 3471.9
    ## Vcells 97878910 746.8  799451069 6099.4 831675659 6345.2

``` r
cuda_empty_cache()

apvienotais_rastrs <- subst(apvienotais_rastrs, 0, NA)
apvienotais_rastrs <- merge(apvienotais_rastrs, Latvia_ref_raster_10x10_crop)
crs(apvienotais_rastrs) <- crs(Latvia_ref_raster_10x10)

#Saglabāšu.
writeRaster(apvienotais_rastrs, "C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd06\\MVR_LAD_centrs.tif", datatype = "INT2S", overwrite=TRUE)
```

<br> 5. Cik šūnās ir gan mežaudžu, gan lauku informācija? <br>6. Cik
šūnas atrodas Latvijas sauszemes teritorijā, bet nav raksturotas šī
uzdevuma iepriekšējos punktos? <br> Pārbaudīšu. Rezultāta rastrā esmu
nodrošinājies pret to, ka vienā šūnā ir 2 vērtības, atlasot no 5 ievades
rastriem mazāko vērtību, bet paskatīšos ievades rastros vai sakrīt.
Kamēr ļauj, praktizējos GPU skaitļošanā.

``` r
rastru_kolekcija[["rLAD"]] <- LAD_raster_10x10

Savienots_rastrs <- rast(rastru_kolekcija) #Alternatīvs risinājums terra::stack()

tensor_sav_r <- torch_tensor(as.matrix(Savienots_rastrs), dtype = torch_int32(), device = "cuda")

logical_mask <- tensor_sav_r > 0 #Uztaisu loģisko masku, noņemot 0. Rezultātā būs tā pati tabula, tikai TRUE ja šūnā nav 0, un false, ja ir. 
logical_mask <- logical_mask$sum(dim = 2) #No matricas izveidoju skaitļu virkni, kurā sassumēts attiecīgi katrā kolonnā (katrā šūnā) cik ir TRUE vērtības (cik ir lielākas par 0)

logical_mask <- torch_where(logical_mask == 1 | logical_mask == 5, 0, logical_mask) #Ja logical mask ir 1, tas nozīmē ka attiecīgajā šūnā ir tikai 1 vērtība no viena rastra slāņa. 5 bija visur ārpus LV robežas (pieļauju ka torch problēmas interpretēt NA)

logical_mask <- as.numeric(logical_mask)

sakritosas_shunas <- Latvia_ref_raster_10x10_crop 
values(sakritosas_shunas) <- logical_mask

sakritosas_shunas <- subst(sakritosas_shunas, 0, NA)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
sakritosas_shunas <- merge(sakritosas_shunas, Latvia_ref_raster_10x10_crop)
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
crs(sakritosas_shunas) <- crs(Latvia_ref_raster_10x10) 

shunas <- sakritosas_shunas > 0
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
cat("Šūnu skaits, kurām ir vairākas vērtības ievades rastra slāņos:", sum(values(shunas, na.rm = TRUE)))
```

    ## Šūnu skaits, kurām ir vairākas vērtības ievades rastra slāņos: 7813

``` r
#Saglabāšu, lai paskatītos kādā interaktīvā ĢIS programmā
writeRaster(sakritosas_shunas, "C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd06\\Sakritosas_shunas.tif", datatype = "INT1U", overwrite=TRUE)

cat("Samērā daudz šūnas, kurās dublējas vērtības. Apskatoties QGIS'ā saprotu, ka lielākā daļa no tām ir nevis tur, kur sakrīt lauki un meži, bet kur sakrīt dažādu klašu mežu informācija. Resp. pašā sākumā kad ielasīju Combined_centrs, nebiju apskatījies dublējošās ģeometrijas. Tās ir", sum(duplicated(st_geometry(Combined_centrs))),".")
```

    ## Samērā daudz šūnas, kurās dublējas vērtības. Apskatoties QGIS'ā saprotu, ka lielākā daļa no tām ir nevis tur, kur sakrīt lauki un meži, bet kur sakrīt dažādu klašu mežu informācija. Resp. pašā sākumā kad ielasīju Combined_centrs, nebiju apskatījies dublējošās ģeometrijas. Tās ir 129 .

<br><br> Paskatīšos cik šūnas nav aptvertas gala apvienotajā rastrā.

``` r
cat("Uzdevumā nav aptvertas:", sum(values(apvienotais_rastrs) == 0, na.rm = TRUE), "šūnas, kas ir", sum(values(apvienotais_rastrs) == 0, na.rm = TRUE) / ncell(apvienotais_rastrs) * 100, "% no visas teritorijas.")
```

    ## Uzdevumā nav aptvertas: 151972527 šūnas, kas ir 62.18712 % no visas teritorijas.
