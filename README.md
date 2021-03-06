# Earth-Engine-from-R
## Basic tutorial for working in Earth Engine from R

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

In this tutorial, I will walk through how to use Google Earth Engine from R; how to import a raster dataset from Earth Engine into R; and highlight some common raster analyses using the landscapemetrics R package. I assume you have a basic working knowledge of R and have created an Earth Engine account. To get started with Earth Engine, please see the resources documented [here](https://earthengine.google.com).

Throughout this tutorial, I will be working with the global human modification dataset (gHM) which provides a cumulative measure of human modification of terrestrial lands globally at 1 square-kilometer resolution. For more information about this dataset, please see the original [paper](https://onlinelibrary.wiley.com/doi/abs/10.1111/gcb.14549) and dataset [description](https://developers.google.com/earth-engine/datasets/catalog/CSP_HM_GlobalHumanModification) within the Earth Engine [Data Catalog](https://developers.google.com/earth-engine/datasets/catalog). 

## First steps

If you haven't already done so, install [Anaconda](https://www.anaconda.com/distribution/) on your computer. 

Next, open your terminal and run the following:
```{r, engine = 'bash', eval = FALSE}
conda create --name gee-demo      # Create a conda environment from which to work from

conda activate gee-demo           # Activate the environment

conda install -c conda-forge earthengine-api # Install the Earth Engine Python API

earthengine authenticate          # Authenticate your access with the command line tool

conda install pandas
conda install numpy

```

Next, in R, load the following libraries and initalize earth engine. You will be prompted to log-in to your EarthEngine account. 

```{r message=FALSE, warning=FALSE}
# load libraries
library(reticulate) # interface with python
library(rgee) # interface with earth engine
library(raster)
library(sf)
library(dplyr)
library(landscapemetrics)
library(maps)

# Point to the Conda environment you just created in the first step
use_condaenv("gee-demo", conda = paste(yourpath, "gee-demo", sep = "/"), required = T)

ee = import("ee") # Import earth engine library
ee$Initialize() # Trigger authentication
np = import("numpy") # import numpy library
pd = import("pandas") # import pandas library
```
## Next steps

Now that we have the environments set, libraries loaded and have initalized EarthEngine, we are ready to import the gHM data from EarthEngine and start working with it in R. 

First, let's import the gHM layer in our workspace and plot it:

```{r}
# Import Human Modification collection
hm_col = ee$
  ImageCollection('CSP/HM/GlobalHumanModification')

# select the first image of the collection which corresponds to the cumulative HM  
hm = hm_col$first()

# plot map of HM
ee_map(eeobject = hm,
       bands = c('gHM'),
       vizparams = list(palette = list('blue','green','yellow','orange','red','magenta'), 
                        min = 0,
                        max = 1),
       center = c( -101.299591, 47.116386),
       zoom_start = 3) 
```

Now, let's clip the gHM layer by a polygon. I'll use Colorado as an example, but you can use any shapefile you can import into R. The last bit of code will apply the median reducer to calculate the median gHM value across all pixels within the polygon feature. 

```{r warning=FALSE, message=FALSE}
# Get states shapefile as sf object as example how to clip and summarise by polygon
states <- st_as_sf(map("state", plot = FALSE, fill = TRUE))
# Filter Colorado shapefile
states = states %>% dplyr::filter(ID == "colorado")
# convert to EarthEngine object
states = sf_as_ee(states)
# Clip HM layer to Colorado polygon
coloradoHM = hm$clip(states)$rename('COHM')
# Plot the clipped layer
ee_map(eeobject = coloradoHM,
       bands = c('gHM'),
       vizparams = list(palette = list('blue','green','yellow','orange','red','magenta'), 
                        min = 0,
                        max = 1),
       center = c(-104.991531, 39.742043),
       zoom_start = 5)
# Get median HM for Colorado
median_hm = hm$reduceRegion(reducer = ee$Reducer$median(),
                              geometry = states)$getInfo()[[1]]
print(paste("median HM for CO:", median_hm, sep = " "))
```
The above code can be modified to extract buffers around points, lines, and/or other polygons, and to calculate additional statistics using EarthEngine's native functions. However, you might also want to make use of R's various packages and tools for further processing and analysis. Below, I'll show how to convert the EarthEngine image into R's raster format and then calculate some basic landscape statistics on it. 

```{r warning=FALSE, message=FALSE}
coloradoHM = coloradoHM$unmask(0)
# Reduce colorado HM to a list
latlng = ee$Image$pixelLonLat()$addBands(coloradoHM)
latlng = latlng$reduceRegion(reducer=ee$Reducer$toList(),
                               geometry=states,
                               maxPixels=1e13,
                               #crs = 'EPSG:3857',
                               scale=1000) # data resolution is 1 km
# Create numpy array with latidude, longitude and gHM values of each pixel
  data = np$array((ee$Array(latlng$get("COHM"))$getInfo()))
  lats = np$array((ee$Array(latlng$get("latitude"))$getInfo()))
  lngs = np$array((ee$Array(latlng$get("longitude"))$getInfo()))
  df <- data.frame(x = lngs, y = lats, tree = data) # convert to data frame
  XYZ <- rasterFromXYZ(df) # convert to raster
# Plot using raster package in R
  plot(XYZ)
```

As a last example, let's reclassify the gHM into 5 categories corresponding to "very low", "low", "medium", "high", and "very high" using thresholds defined in [Kennedy et al. (2019)](https://onlinelibrary.wiley.com/doi/abs/10.1111/gcb.14549). Finally, we'll use the landscapemetrics R package to calculate the number of patches (NP) of very low HM throughout CO and get the largest patch index (LPI) for the "very low" HM category. For additional resources on how to calculate landscape metrics for local landscapes, please see Jakub Nowosad's [website](https://nowosad.github.io/post/lsm-bp2/).

```{r}
# Reclassify HM into 5 classes: "very low", "low", "medium", "high", "very high"
reclassmat = matrix(c(0.0,0.01, 1,
               0.01, 0.1, 2,
               0.1, 0.4, 3,
               0.4, 0.7, 4,
               0.7, 1.0, 5), ncol = 3, byrow = T)

HM_reclass <- reclassify(XYZ, reclassmat)

# Get number of patches for very low HM category using Queen's case
NP <- lsm_c_np(HM_reclass, directions = 8) %>%
  dplyr::filter(class == 1) %>%
  dplyr::select(one_of(c("class","value")))
names(NP)<- c("class", "NP") 

# Get largest patch index for very low HM category in Colorado
LPI <- lsm_c_lpi(HM_reclass, directions = 8) %>%
  dplyr::filter(class == 1) %>%
  dplyr::select(one_of(c("class","value")))
names(LPI) <- c("class", "LPI")

metrics = left_join(NP, LPI, by = "class")
print(metrics)

```
