Uzd05_Zuperka
================
Marks

Izvēlēšos tks93_50km kvadrātus balstoties uz tā vai tie šķērso Centra
mežniecības datus vai nē

``` r
if (!require("sfarrow")) install.packages("sfarrow")
```

    ## Loading required package: sfarrow

``` r
if (!require("sf")) install.packages("sf")
```

    ## Loading required package: sf

    ## Linking to GEOS 3.12.2, GDAL 3.9.3, PROJ 9.4.1; sf_use_s2() is TRUE

``` r
if (!require("ggplot2")) install.packages("ggplot2")
```

    ## Loading required package: ggplot2

``` r
#Ielasu
tks93_50km <- st_as_sf(arrow::read_parquet("C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd03\\HiQBioDiv_vector_reference_grids\\tks93_50km.parquet"))
Combined_centrs <- st_as_sf(arrow::read_parquet("C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd02\\Combined_centrs.parquet"))

tks_centrs_i <- st_filter(tks93_50km, Combined_centrs) #Iegūstu tks93_50km poligonus, kuri krustojas ar Centra mežniecības datiem.

#Apskatīšos kuri 4 poligoni ir blakus
ggplot(data = tks_centrs_i) +
  geom_sf() + 
  geom_sf_text(aes(label = NOSAUKUMS), size = 3, vjust = -0.5) +
  theme_minimal()
```

![](Uzd05_Zuperka_files/figure-gfm/ielasu%20datus%20un%20izvēlos%20teritoriju%20(sagatavošanās%20darbam)-1.png)<!-- -->

``` r
#Paņemšu Ķemeri, Tukums, Jaunpils, Dobele
saraksts <- c("Ķemeri", "Tukums", "Jaunpils", "Dobele")
tks_filtrs <- tks_centrs_i[tks_centrs_i$NOSAUKUMS %in% saraksts, ]

#Izdzēsīšu nevajadzīgo
rm(tks_centrs_i, tks93_50km)
```

<br> Pārskatīšu funkciju - veicu dažas izmaiņas. Galvenokārt pārtaisīju
no rasterize uz fasterize (nezinu kāpēc bet šoreiz rasterize gāja ļoti
ilgi, bet šis gāja kā vajag - iepriekšējā uzdevumā bija otrādi, lai gan
tāpat strādāju ar apgrieztu references rastru) un noņēmu saglabāšanu, ko
veikšu for ciklā katrā apakšuzdevumā atsevišķi.

``` r
library(terra)
```

    ## terra 1.8.15

``` r
library(fasterize)
```

    ## 
    ## Attaching package: 'fasterize'

    ## The following object is masked from 'package:graphics':
    ## 
    ##     plot

    ## The following object is masked from 'package:base':
    ## 
    ##     plot

``` r
library(dplyr)
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:terra':
    ## 
    ##     intersect, union

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
#Ielasīšu rastrus kā SpatRaster objektus
Latvia_raster_ref <- rast("C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd03/HiQBioDiv_raster_reference_grids/LV10m_10km.tif")
Latvia_raster_100m_ref <- rast("C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd03/HiQBioDiv_raster_reference_grids/LV100m_10km.tif")

Latvia_raster <- raster(terra::subst(Latvia_raster_ref, 1, 0)) #Šis jāpārveido uz parasto raster objektu, jo fasterize nepieņem SpatRaster kā ievades slāni
```

    ## |---------|---------|---------|---------|=========================================                                          

``` r
crs(Latvia_raster) <- crs(Latvia_raster_ref) #Biju nedaudz spēlējies citās programmās ar GDAL, tāpēc standarta implementācijā likt EPSG:3059 nestrādā, tāpēc vienkārši kopēju references rastra koordinātu sistēmu

mana_funkcija <- function(sf_file_st, Latvia_raster) {
    
    sf_file_filtered <- sf_file_st[sf_file_st$s10 == 1, , drop=FALSE] #Nomainīju filtrēšanas funkciju, drop=FALSE lai paliek citas vērtības 
    sf_file_filtered$s10 <- as.double(sf_file_filtered$s10) #Fasterize ievadei pieprasa double tipu - pārveidoju
    
    priedes_10x10 <- terra::rast(fasterize::fasterize(sf_file_filtered, Latvia_raster, field = "s10", background = 0)) #fasterize rezultātu uzreiz pārveidoju uz SpatRaster objektu (aizņem daudzreiz mazāku RAM apjomu un turpmākām darbībām)
  
    priedes_100m_prop <- aggregate(priedes_10x10, fact = 10, fun="sum", na.rm = TRUE) #Sum lai iegūtu cik šūnas ir ar vērtībām
    priedes_100m_prop <- priedes_100m_prop / 100 
    
    return(priedes_100m_prop)
}
```

<br> Principā funkcijā var iebarot nevis kombinēto sf_file_st, bet
savienotu ar join failu katrai nodaļai. To arī darīšu. <br> Pirmais
uzdevums - izmantošu parasto pieeju ar for ciklu.

``` r
#Nomainu uz 5. uzdevuma direktoriju
output_dir <- "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd05/Spatial join" 

if (!dir.exists(output_dir)) dir.create(output_dir, recursive = TRUE)

Sakums1 <- Sys.time()

for (i in seq_along(saraksts)) {

  nosaukums <- saraksts[i] #Izvelku i-tā faila nosaukumu 
  temp_file <- st_as_sf(tks_filtrs[tks_filtrs$NOSAUKUMS == nosaukums, , drop=FALSE]) #Filtrēju no karšu lapām i-to failu pēc nosaukuma

  sf_file_st <- st_join(Combined_centrs, temp_file, join = st_intersects, left=FALSE) #Kā join parametru norādu st_intersects, kas atgriezīs tikai tās x vērtības kuras krustojas ar y vērtībām

  #Apgriežu rastru pēc attiecīgā i faila bounding box
  bbox <- raster::extent(st_bbox(temp_file)) 

  crop_rastrs <- terra::crop(Latvia_raster, bbox)

  priedes_100m_prop <- mana_funkcija(sf_file_st, crop_rastrs) #Izpildu funkciju
  
  file_name <- paste0(output_dir, "/", tools::file_path_sans_ext(basename(nosaukums)), "_join.tif") #Iegūstu nosaukumu
  terra::writeRaster(priedes_100m_prop, file_name, datatype = "FLT4S", overwrite = TRUE)
}
#Ielasu failus ko izveidoju
Spatial_join_list <- list.files(output_dir, pattern = "\\.tif$", full.names = TRUE)

sp_raster_list <- list() #Izveidoju sarakstu kuram for cikla veidā pievienoju izveidotos failus

for (i in seq_along(Spatial_join_list)) {
  sp_raster_list[[i]] <- rast(Spatial_join_list[i])
  names(sp_raster_list)[i] <- tools::file_path_sans_ext(basename(Spatial_join_list[i]))
}

#Izveidoju SpatRastCollection kuru iebarot mosaic funkcijā
rsrc <- terra::sprc(sp_raster_list)

terra::mosaic(rsrc, filename = "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd05/Spatial join/Join_mosaic.tif", fun = "max")
```

    ## class       : SpatRaster 
    ## dimensions  : 500, 500, 1  (nrow, ncol, nlyr)
    ## resolution  : 100, 100  (x, y)
    ## extent      : 425000, 475000, 275000, 325000  (xmin, xmax, ymin, ymax)
    ## coord. ref. : LKS92 / Latvia TM 
    ## source      : Join_mosaic.tif 
    ## varname     : Dobele_join 
    ## name        : layer 
    ## min value   :     0 
    ## max value   :     1

``` r
#Mērogošanas laika beigas
Beigas1 <- Sys.time()

Starpiba_join <- difftime(Beigas1, Sakums1, units = "mins")

cat("Apstrādes laiks ar st_join:", round(Starpiba_join,1), "minūtes \n")  
```

    ## Apstrādes laiks ar st_join: 0.3 minūtes

<br> 1.1. - spatial join objektu skaits ir 10 000 - 20 000 katrā.
Iegūtie slāņi ietver arī ārpus lapas robežām esošos failus (ja neliela
daļa centra poligona iekrita iekšā karšu lapā, tad viss poligons
saglabājās) <br> 1.2. - katrā atsevišķi saglabātā failā ir tikai tās
šūnas, kuras ir iekšā kartes lapā. Gan jau jo biju apgriezis rastru kuru
izmantoju funkcijā. <br> 1.3. Apvienotajā failā robežas neredzu, jo tas
pats kas iepr. punktā. <br> Tālāk izmantošu clip lai iegūtu ievades
sf_file_st failu, tā pati pieeja

``` r
#Nomainu jaunu direktoriju
output_dir <- "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd05/Clipping" 

if (!dir.exists(output_dir)) dir.create(output_dir, recursive = TRUE)

Sakums2 <- Sys.time()

for (i in seq_along(saraksts)) {

  nosaukums <- saraksts[i] #Izvelku i-tā faila nosaukumu 
  temp_file <- st_as_sf(tks_filtrs[tks_filtrs$NOSAUKUMS == nosaukums, , drop=FALSE]) #Filtrēju no karšu lapām i-to failu pēc nosaukuma

  sf_file_st <- st_intersection(Combined_centrs, temp_file)

  #Apgriežu rastru pēc attiecīgā i faila bounding box
  bbox <- raster::extent(st_bbox(temp_file)) 

  crop_rastrs <- terra::crop(Latvia_raster, bbox)

   priedes_100m_prop <- mana_funkcija(sf_file_st, crop_rastrs) #Izpildu funkciju
  
  file_name <- paste0(output_dir, "/", tools::file_path_sans_ext(basename(nosaukums)), "_clip.tif") #Iegūstu nosaukumu
  terra::writeRaster(priedes_100m_prop, file_name, datatype = "FLT4S", overwrite = TRUE)
}

#Ielasu failus ko izveidoju
Spatial_join_list <- list.files(output_dir, pattern = "\\.tif$", full.names = TRUE)

sp_raster_list <- list() #Izveidoju sarakstu kuram for cikla veidā pievienoju izveidotos failus

for (i in seq_along(Spatial_join_list)) {
  sp_raster_list[[i]] <- rast(Spatial_join_list[i])
  names(sp_raster_list)[i] <- tools::file_path_sans_ext(basename(Spatial_join_list[i]))
}

#Izveidoju SpatRastCollection kuru iebarot mosaic funkcijā
rsrc <- terra::sprc(sp_raster_list)

terra::mosaic(rsrc, filename = "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd05/Clipping/Clip_mosaic.tif", fun = "max")
```

    ## class       : SpatRaster 
    ## dimensions  : 500, 500, 1  (nrow, ncol, nlyr)
    ## resolution  : 100, 100  (x, y)
    ## extent      : 425000, 475000, 275000, 325000  (xmin, xmax, ymin, ymax)
    ## coord. ref. : LKS92 / Latvia TM 
    ## source      : Clip_mosaic.tif 
    ## varname     : Dobele_clip 
    ## name        : layer 
    ## min value   :     0 
    ## max value   :     1

``` r
Beigas2 <- Sys.time()

Starpiba_clipping <- difftime(Beigas2, Sakums2, units = "mins")

cat("Apstrādes laiks ar st_intersection:", round(Starpiba_clipping,1), "minūtes \n")
```

    ## Apstrādes laiks ar st_intersection: 0.2 minūtes

<br> 2.1. Intersection jau izgriež pēc karšu lapas robežām un apgriež
malās esošos poligonus, un nav neviena poligona kas būtu ārpus karšu
lapas robežas. <br> 2.2. Sanāk tā ka st_intersection izveidotā lapa jau
ir apgriezta, tāpēc ari rezultāts automātiski būs apgriezts, rastra
apgriešana tikai samazina skaitļošanas laiku. <br> 2.3. Tas pats kas
iepr. punktā. <br> 3. uzdevums - izmantošu <u> spatial filtering </u>
funkciju st_filter lai iegūtu datus par karšu lapā esošiem mežiem

``` r
#Nomainu jaunu direktoriju
output_dir <- "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd05/Spatial filtering" 

if (!dir.exists(output_dir)) dir.create(output_dir, recursive = TRUE)

Sakums3 <- Sys.time()

for (i in seq_along(saraksts)) {

  nosaukums <- saraksts[i] #Izvelku i-tā faila nosaukumu 
  temp_file <- st_as_sf(tks_filtrs[tks_filtrs$NOSAUKUMS == nosaukums, , drop=FALSE]) #Filtrēju no karšu lapām i-to failu pēc nosaukuma

  sf_file_st <- st_filter(Combined_centrs, temp_file)

  #Apgriežu rastru pēc attiecīgā i faila bounding box
  bbox <- raster::extent(st_bbox(temp_file)) 

  crop_rastrs <- terra::crop(Latvia_raster, bbox)

  priedes_100m_prop <- mana_funkcija(sf_file_st, crop_rastrs) #Izpildu funkciju
  
  file_name <- paste0(output_dir, "/", tools::file_path_sans_ext(basename(nosaukums)), "_filter.tif") #Iegūstu nosaukumu
  terra::writeRaster(priedes_100m_prop, file_name, datatype = "FLT4S", overwrite = TRUE)
}

#Ielasu failus ko izveidoju
Filter_list <- list.files(output_dir, pattern = "\\.tif$", full.names = TRUE)

sp_raster_list <- list() #Izveidoju sarakstu kuram for cikla veidā pievienoju izveidotos failus

for (i in seq_along(Filter_list)) {
  sp_raster_list[[i]] <- rast(Filter_list[i])
  names(sp_raster_list)[i] <- tools::file_path_sans_ext(basename(Filter_list[i]))
}

#Izveidoju SpatRastCollection kuru iebarot mosaic funkcijā
rsrc <- terra::sprc(sp_raster_list)

terra::mosaic(rsrc, filename = "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd05/Spatial filtering/Filter_mosaic.tif", fun = "max")
```

    ## class       : SpatRaster 
    ## dimensions  : 500, 500, 1  (nrow, ncol, nlyr)
    ## resolution  : 100, 100  (x, y)
    ## extent      : 425000, 475000, 275000, 325000  (xmin, xmax, ymin, ymax)
    ## coord. ref. : LKS92 / Latvia TM 
    ## source      : Filter_mosaic.tif 
    ## varname     : Dobele_filter 
    ## name        : layer 
    ## min value   :     0 
    ## max value   :     1

``` r
Beigas3 <- Sys.time()

Starpiba_filter <- difftime(Beigas3, Sakums3, units = "mins")

cat("Apstrādes laiks ar st_filter:", round(Starpiba_filter,2), "minūtes \n")
```

    ## Apstrādes laiks ar st_filter: 0.26 minūtes

<br> 3.1. st_filter atgriež st_join līdzīgu rezultātu - poligoni ir
ārpus karšu lapām. Vienīgais ko pamanīju, ir ka st_filter izveidotam
slānim ir 71 lauks, bet ar st_join un st_intersection - 75. Izskatās ka
join un intersection nodublē daļu no kolonnām (piem. id un objectid_1 ir
identiski).  
<br> 3.2. Līdzīgi kā iepriekš izmantoju pēc karšu lapas apgrieztu rastru
tāpēc gala rastra šūnas ir tikai lapas iekšienē.  
<br> 3.3. Tas pats kas iepr. punktā. <br> 4. uzdevums - izmantošu <u>
clipping </u> pieeju datu izgriešanai, bet gala rastra ieguvei
neizmantošu for ciklu.

``` r
#Nomainu jaunu direktoriju
output_dir <- "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd05/Bez cikla" 

if (!dir.exists(output_dir)) dir.create(output_dir, recursive = TRUE)

Sakums4 <- Sys.time() #Mērogošanas sākums

#Sākumā ielasīšu atsevišķi katru karšu lapu darba vidē

lapas_saraksts <- split(tks_filtrs, seq(nrow(tks_filtrs))) #Sadalu tks_filtrs ko ieguvu iepriekš tā, lai katra karšu lapa ir savā rindiņā

#Ielasīšu katru karšu lapu kā atsevišķu sf objektu
Lapa_1 <- st_as_sf(tks_filtrs[1, , drop = FALSE])
Lapa_2 <- st_as_sf(tks_filtrs[2, , drop = FALSE])
Lapa_3 <- st_as_sf(tks_filtrs[3, , drop = FALSE])
Lapa_4 <- st_as_sf(tks_filtrs[4, , drop = FALSE])

#Dabūšu katras karšu lapas nosaukumu lai vēlāk var saglabāt
nosaukums1 <- tks_filtrs$NOSAUKUMS[1]
nosaukums2 <- tks_filtrs$NOSAUKUMS[2]
nosaukums3 <- tks_filtrs$NOSAUKUMS[3]
nosaukums4 <- tks_filtrs$NOSAUKUMS[4]

Lapa_1 <- st_intersection(Combined_centrs, Lapa_1)
Lapa_2 <- st_intersection(Combined_centrs, Lapa_2)
Lapa_3 <- st_intersection(Combined_centrs, Lapa_3)
Lapa_4 <- st_intersection(Combined_centrs, Lapa_4)

#Tālāk izpildīšu mana funkcija katrai karšu lapai atsevišķi bez for cikla  
sf_file_st <- Lapa_1 #Nodrošinu ka atbilst nosaukumam funkcijā

#Izgriezīšu rastru pēc attiecīgās funkcijas bbox
bbox <- raster::extent(st_bbox(sf_file_st)) 
crop_rastrs <- terra::crop(Latvia_raster, bbox)
  
priedes_100m_prop <- mana_funkcija(sf_file_st, crop_rastrs)

file_name <- paste0(output_dir, "/", nosaukums1, "_bez_cikla.tif")
terra::writeRaster(priedes_100m_prop, file_name, datatype = "FLT4S", overwrite = TRUE)
 
###2 
sf_file_st <- Lapa_2

bbox <- raster::extent(st_bbox(sf_file_st)) 
crop_rastrs <- terra::crop(Latvia_raster, bbox)
  
priedes_100m_prop <- mana_funkcija(sf_file_st, crop_rastrs)

file_name <- paste0(output_dir, "/", nosaukums2, "_bez_cikla.tif")
terra::writeRaster(priedes_100m_prop, file_name, datatype = "FLT4S", overwrite = TRUE)  
 
###3
sf_file_st <- Lapa_3

bbox <- raster::extent(st_bbox(sf_file_st)) 
crop_rastrs <- terra::crop(Latvia_raster, bbox)
  
priedes_100m_prop <- mana_funkcija(sf_file_st, crop_rastrs)  

file_name <- paste0(output_dir, "/", nosaukums3, "_bez_cikla.tif")
terra::writeRaster(priedes_100m_prop, file_name, datatype = "FLT4S", overwrite = TRUE) 

###4
sf_file_st <- Lapa_4

bbox <- raster::extent(st_bbox(sf_file_st)) 
crop_rastrs <- terra::crop(Latvia_raster, bbox)
  
priedes_100m_prop <- mana_funkcija(sf_file_st, crop_rastrs)  

file_name <- paste0(output_dir, "/", nosaukums4, "_bez_cikla.tif")
terra::writeRaster(priedes_100m_prop, file_name, datatype = "FLT4S", overwrite = TRUE) 

Beigas4 <- Sys.time() #Mērogošanas beigas

Starpiba_bez_for <- difftime(Beigas4, Sakums4, units = "mins")

cat("Apstrādes laiks bez for cikla:", round(Starpiba_bez_for, 2), "minūtes \n")
```

    ## Apstrādes laiks bez for cikla: 0.18 minūtes

<br> Neredzu nekādas atšķirības ar iepriekšējiem uzdevumiem - gan laika
ziņā, gan rezultātā. Visos variantos vērtības ir identiskas. Bez for
cikla vienīgi sanāk ļoti daudz copy-paste mainot tikai nosaukumus <br>
5. apakšuzdevums - pielietošu mana_funkcija visiem mežniecības datoriem
un izgriezīšu man interesējošās lapas

``` r
#Nomainu jaunu direktoriju
output_dir <- "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd05/Izgriešana no centra rezultāta" 
if (!dir.exists(output_dir)) dir.create(output_dir, recursive = TRUE)

Sakums5 <- Sys.time() #Mērogošanas sākums (tā kā iepriekš references rastra apgriešana bija iekšā for ciklā un laika mērogošanas sākums bija pirms apgriešanas sākuma, uzskatu ka korekti būs arī šeit to likt pirms references rastra griešanas)

#Apgriezīšu rastru pēc interesējošo 4 lapu bounding box
bbox <- raster::extent(st_bbox(tks_filtrs)) 
crop_rastrs <- terra::crop(Latvia_raster, bbox)

#Pielietošu mana_funkcija
sf_file_st <- Combined_centrs
priedes_100m_prop <- mana_funkcija(sf_file_st, crop_rastrs)

clip <- mask(priedes_100m_prop, tks_filtrs)

#Saglabāšu izvades rastru
file_name <- paste0(output_dir, "/", "Izgriezts_centrs.tif")
terra::writeRaster(clip, file_name, datatype = "FLT4S", overwrite = TRUE)

Beigas5 <- Sys.time() #Mērogošanas beigas

Starpiba_izgriezts_centrs <- difftime(Beigas5, Sakums5, units = "mins")

cat("Apstrādes laiks bez for cikla:", round(Starpiba_izgriezts_centrs, 2), "minūtes \n")
```

    ## Apstrādes laiks bez for cikla: 0.06 minūtes

``` r
rm(list = ls())
gc() #Nākamais uzdevums ir apjomīgs tāpēc izdzēšu nevajadzīgo info
```

    ##           used  (Mb) gc trigger  (Mb)  max used  (Mb)
    ## Ncells 3256552 174.0   10530720 562.5  13163400 703.1
    ## Vcells 5035989  38.5   95432276 728.1 116472722 888.7

<br> Šī pieeja ir visātrākā - sanāk 3 reizes ātrāk nekā pārējās metodes.
Sākumā biju tks_filtrs vietā pie bbox licis centra failu - bija tikpat
ilgi cik pārējos uzdevumos, bet rezultējošais rastrs nesakrita ar
references rastru un `terra::project()` nestrādāja. Karšu lapas jau
sakrīt ar references rastriem, tādēļ šīs problēmas nav. <br> Citādi
rezultāts ir tāds pats. <br> 6. uzdevums - saistīšu Centra mežniecības
datus ar projekta references vektoru 100x100 metri un aprēķināšu priežu
īpatsvaru katrā no šūnām.

``` r
Sakums6 <- Sys.time() #Mērogošanas sākums

#Iepriekš izdzēsu visu tāpēc ielādēju pa jaunam tikai to ko vajag un uztaisu vajadzīgo ievades slāni
Combined_centrs <- st_as_sf(arrow::read_parquet("C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd02\\Combined_centrs.parquet"))
tks93_50km <- st_as_sf(arrow::read_parquet("C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd03\\HiQBioDiv_vector_reference_grids\\tks93_50km.parquet"))
tks_centrs_i <- st_filter(tks93_50km, Combined_centrs)
saraksts <- c("Ķemeri", "Tukums", "Jaunpils", "Dobele")
tks_filtrs <- tks_centrs_i[tks_centrs_i$NOSAUKUMS %in% saraksts, ]

Combined_centrs <- st_intersection(Combined_centrs, tks_filtrs) 
#Ielasīšu vektoru
tikls100_vector <- st_as_sf(arrow::read_parquet("C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd03\\HiQBioDiv_vector_reference_grids\\tikls100_sauzeme.parquet"))

#Apgriezīšu viņu uz tikai interesējošo teritoriju, izmantošu par st_intersection optimālāku un ātrāku pieeju - no Centra datiem izvilkšu bounding box, izveidošu sf objektu no tā un apgriezīšu izmantojot to
bbox <- st_bbox(Combined_centrs)

#Uztaisīšu data frame kurā attiecīgi saglabāšu poligona virsotņu koordinātas 
dataframe <- data.frame(
  X_coord = c(bbox["xmin"], bbox["xmax"], bbox["xmax"], bbox["xmin"]), 
  Y_coord = c(bbox["ymax"], bbox["ymax"], bbox["ymin"], bbox["ymin"])
)

if (!require("sfheaders")) install.packages("sfheaders") #Izmantošu sfheaders pakotni, kas ļauj izveidot poligonu uzreiz no data frame
```

    ## Loading required package: sfheaders

``` r
polygon <- sfheaders::sf_polygon(
  obj = dataframe,
  x = "X_coord",
  y = "Y_coord"
)

#Nodrošinu ka sakrīt CRS
sf::st_crs(polygon) <- st_crs(tikls100_vector)

#Apgriežu tīkla vektordatus
tikls100_v_clip <- st_intersection(tikls100_vector, polygon)

#Savienoju centra datus un pielieku tiem datus kurai references šūnai tie atbilst.
Centrs_100_join <- st_intersection(Combined_centrs, tikls100_v_clip)
```

<br> Tālāk izveidošu 100x100 m rastru, kur būs parādīts priežu mežaudžu
īpatsvars šūnā. Nevar izmantot mana_funkcija pieeju, jo tā balstās uz
mazāka 10x10 rastra šūnu saskaitīšanu, tāpēc jauna pieeja.

``` r
#Iepriekš biju izmantojis st_join, bet tad ir ļoti daudz dublējošās vērtības (piem. ja meža poligons A vienlaicīgi atrodas 3 kvadrātos B, tad tiks izveidoti 3 ģeometriski vienādi poligoni, kur vienīgais kas atšķirsies būs attiecīgā kvadrāta B dati), tāpēc izmantoju st_intersection, kas izveidos unikālos poligonus. 

#Tālāk vajag tikai 3 kolonnas - ģeometriju, s10 un "id.2" (kurai šūnai attiecas konkrētais poligons)
Centrs_vajadzigie <- Centrs_100_join[c("geometry", "s10", "id.2")]

#Filtrēju tikai tos poligonus, kuri ir priedes
Centrs_filtered <- Centrs_vajadzigie[Centrs_vajadzigie$s10 == 1, , drop=FALSE]

#Tālāk izmantošu ArcGIS "dissolve" alternatīvu R un apvienošu poligonus pēc "id.2" lauka (vieglāka rēķinašana turpmāk). Praktiski šis izveidos slāni, kurā katrā rindiņā būs 1 unikāls vektora šūnas identifikators, bet būs vairākas ģeometrijas kas tai atbilst 
Centrs_dissolve <- Centrs_filtered %>%
  group_by(id.2) %>%
  summarize(geometry = st_union(geometry))
  
#Aprēķināšu katrā karšu lapā esošo priežu poligonu platību m^2
Centrs_dissolve$laukums_m2 <- round(st_area(Centrs_dissolve), 2)

#Pārbaudu vai ir tā, ka id.2 ir unikāla (nedublējas šūnu identifikatori) un max laukuma vērtība ir zem 10 000 (ja ir 10 000, tas nozīmē, ka attiecīgo šūnu pilnībā klāj priežu mežs, un vairāk nedrīkst būt). 

ifelse(length(unique(Centrs_dissolve$id.2)) == length(Centrs_dissolve$id.2), "IR OK - KATRĀ RINDIŅĀ IR SAVA UNIKĀLA ŠŪNA", "NAV OK")
```

    ## [1] "IR OK - KATRĀ RINDIŅĀ IR SAVA UNIKĀLA ŠŪNA"

``` r
ifelse((max(Centrs_dissolve$laukums_m2) == 10000), "IR OK - MAX ŠŪNAS IZMĒRS IR 10 000", "NAV OK")
```

    ## [1] "IR OK - MAX ŠŪNAS IZMĒRS IR 10 000"

``` r
#Viss ok, tad varu rēķināt proporciju cik % attiecīgajā šūnā ir priežu meži (zinot, ka šūnas izmērs ir 10 000 m^2). 
Centrs_dissolve$laukums_m2 <- round((Centrs_dissolve$laukums_m2 / 10000), 2) 

#Esmu ieguvis vektorslāni, kur katrai šūnai ir norādīts cik % no tās veido priežu mežs. Tālāk varētu izmantot rasterize funkciju ar fun="sum", bet iešu citu ceļu. 
#Tālāk ģeometrija nav nepieciešama - to noņemu ātrākai skaitļošanai.
Centrs_dissolve <- st_drop_geometry(Centrs_dissolve)

#Pievienošu tikls_100_v_clip (apgrieztam references vektorslānim) datus par to cik priežu meži tajā ir.
#No sākuma uztaisīšu tā, lai sakrīt kolonnas nosaukumi, pēc kuriem tiks veikta savietošana.
Centrs_dissolve$id <- Centrs_dissolve$id.2

#Savietoju, izmantojot dplyr left_join 
priedes_100m_vekt <- dplyr::left_join(tikls100_v_clip, Centrs_dissolve, by = "id")
```

<br> Priedes_100m_vekt slānī katrai šūnai ir informācija no 0 - 1 par to
kāda ir priežu mežu aizņemtā platība. Atliek tikai rasterizēt.

``` r
#Atliek tikai rasterizēt - izmantošu fasterize. Atmiņai ietilpīgs process tāpēc izdzēsīšu visu kas nav vajadzīgs
rm(Centrs_100_join, Centrs_filtered, Centrs_vajadzigie, Centrs_dissolve, Combined_centrs, tikls100_v_clip, tikls_100_vector, polygon, dataframe, tks_centrs_i, saraksts, tks_filtrs, tks93_50km)
gc()
```

    ##             used   (Mb) gc trigger   (Mb)  max used   (Mb)
    ## Ncells  57271668 3058.7  112632535 6015.3 112632535 6015.3
    ## Vcells 217792974 1661.7  428024052 3265.6 356620034 2720.8

``` r
#Ielasu references rastru
Latvia_raster_100m_ref <- rast("C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd03/HiQBioDiv_raster_reference_grids/LV100m_10km.tif")

#Uzreiz apgriezīšu rastru uz vajadzīgo teritoriju (laika un atmiņas taupīšana)
ext <- terra::ext(bbox)

Latvia_raster_100m_ref_crop <- terra::crop(Latvia_raster_100m_ref, ext)

Latvia_raster_100m_ref_subs <- terra::subst(Latvia_raster_100m_ref_crop, 1, 0) #Aizvietoju vērtības un ielasu kā raster objektu (fasterize pieprasa raster objektu)

rm(Latvia_raster_100m_ref_crop, Latvia_raster_100m_ref)

#Rasterizēju
priedes_100m_prop <- terra::rast(fasterize::fasterize(priedes_100m_vekt, raster(Latvia_raster_100m_ref_subs), field = "laukums_m2", background = 0))

#Rasterizēšana veikta jau izmantojot references slāni, tāpēc citas darbības nav nepieciešamas. Saglabāju.
output_dir <- "C:/Users/mark7/Documents/MZ_HiQBioDiv_macibas/Uzd05/Izgriešana no centra rezultāta"
file_name <- paste0(output_dir, "/", "Priedes_6ais_uzdevums.tif")
 
if (!dir.exists(output_dir)) dir.create(output_dir, recursive = TRUE)

terra::writeRaster(priedes_100m_prop, file_name, datatype = "FLT4S", overwrite=TRUE)

Beigas6 <- Sys.time() #Mērogošanas beigas
Starpiba_spatial <- difftime(Beigas6, Sakums6, units = "mins")
cat("Apstrādes laiks bez for cikla:", round(Starpiba_spatial, 2), "minūtes \n")
```

    ## Apstrādes laiks bez for cikla: 2.38 minūtes

``` r
#Paskatīšos rezultātu - izmantošu raster minus un ar clip metodi iegūto rastru
minus <- rast(raster(priedes_100m_prop) - raster(rast("C:\\Users\\mark7\\Documents\\MZ_HiQBioDiv_macibas\\Uzd05\\Clipping\\Clip_mosaic.tif")))

terra::plot(minus, col = terrain.colors(100))
```

![](Uzd05_Zuperka_files/figure-gfm/sestais%20uzdevums%203%20-%20rasterizēju%20references%20vektoru%20ar%20priežu%20datiem-1.png)<!-- -->
<br> Atšķirība principā nav liela, līdz 5%, turklāt sanāk 6. variantā
izveidotais rāda nedaudz lielāku proporciju. Skaitļošanas laiks pirmajām
metodēm ir daudz ātrāks. <br> Laika ziņā efektīvākā būs piektā
apakšuzdevuma pieeja - strādāt ar vienu lielāku failu un beigās izgriezt
no tā nepieciešamo informāciju. Es griezu ārā references rastru, tāpēc
beigās sanāca ka visos gadījumos ārpus četrām karšu lapām nebija nekas,
bet ja būtu izmantojis pilnu rastru, tad teiktu ka st_intersection
otrajā uzdevumā būs labākā - nebūs poligonu kas ir ārpus robežām,
attiecīgi nebūs pēc tam problēmu tās savietot ar citām karšu lapu
aprēķiniem.  
<br>Precizitātes ziņā teiktu ka sestajā apakšuzdevumā būs precīzākā, jo
strādājam pa taisno ar vektora ģeometrijām, nevis rastru, kas praktiski
vienkāršo ģeometrijas.
