---
layout: post
title: "eBird Poles of Inaccessibility"
published: true
excerpt: >
  Using eBird data to create maps of birding effort and the least birded 
  locations globally.
category: ebird
tags: r gis ebird
leaflet: true
splash: "/img/inaccessibility/ebird-inaccessibility_distance-to-checklist.png"
editor_options: 
  chunk_output_type: console
---

I work with [eBird](https://ebird.org/) data every data, both for my job and for personal projects, and I think it's one of the coolest freely available datasets around! At almost half a billion records, it's certainly one of the largest, and the possibilities for cool visualization and analyses are endless. A friend recently proposed the idea of finding the birding "poles of inaccessibility". In general, [poles of inaccesibility](https://en.wikipedia.org/wiki/Pole_of_inaccessibility) are the locations on earth that are most challenging to get to. For example, a group at Oxford recently produced a [global map of travel time to the nearest city](https://roadlessforest.eu/map.html). In the context of birding, the idea is to find the locations on the planet that are the least visited by birders; this could serve both to identify interesting areas to go birding and gaps in the eBird data that could be filled.

## Setup

I'll start by loading libraries and defining a few functions that will come in handy later.


```r
library(raster)
library(fasterize)
library(tidyverse)
library(sf)
library(RPostgreSQL)
library(rnaturalearth)
library(here)
proj <- "+proj=wag4 +lon_0=0"

# generate a bounding box to visualize edges of map
make_bbox <- function(lng, lat, spacing, crs = 4326) {
  if (is.na(spacing[1])) {
    lng_seq <- lng
  } else {
    lng_seq <- seq(lng[1], lng[2], length.out = ceiling(diff(lng) / spacing[1]))
  }
  if (is.na(spacing[2])) {
    lat_seq <- lat
  } else {
    lat_seq <- seq(lat[1], lat[2], length.out = ceiling(diff(lat) / spacing[2]))
  }
  bb <- rbind(
    data.frame(lng = lng_seq, lat = lat[1]),
    data.frame(lng = lng[2], lat = lat_seq),
    data.frame(lng = rev(lng_seq), lat = lat[2]),
    data.frame(lng = lng[1], lat = rev(lat_seq))) %>% 
    as.matrix() %>% 
    list() %>% 
    sf::st_polygon() %>% 
    sf::st_sfc(crs = 4326) %>% 
    sf::st_transform(crs = crs)
  return(bb)
}

# genrate graticules for map
make_graticules <- function(lng_breaks, lat_breaks, spacing, crs = 4326) {
  if (is.na(spacing[1])) {
    lng_seq <- c(-180, 180)
  } else {
    lng_seq <- seq(-180, 180, length.out = ceiling(360 / spacing[1]))
  }
  if (is.na(spacing[2])) {
    lat_seq <- c(-90, 90)
  } else {
    lat_seq <- seq(-90, 90, length.out = ceiling(180 / spacing[2]))
  }
  meridians <- purrr::map(lng_breaks, ~ cbind(., lat_seq)) %>% 
    sf::st_multilinestring() %>% 
    sf::st_sfc(crs = 4326) %>% 
    sf::st_sf(type = "meridian")
  parallels <- purrr::map(lat_breaks, ~ cbind(lng_seq, .)) %>% 
    sf::st_multilinestring() %>% 
    sf::st_sfc(crs = 4326) %>% 
    sf::st_sf(type = "parallel")
  grat <- rbind(meridians, parallels) %>% 
    sf::st_transform(crs = crs)
  return(grat)
}

# replace zeros with NA in a raster
raster_to_na <- function(x, value = 0) {
  stopifnot(inherits(x, "Raster"))
  stopifnot(is.numeric(value), length(value) == 1)
  
  if (inherits(x, "RasterLayer")) {
    x[x[] == value] <- NA_real_
  } else {
    for (i in seq.int(raster::nlayers(x))) {
      x[[i]][x[[i]][] == value] <- NA_real_
    }
  }
  return(x)
}
```

## eBird to PostgreSQL

In general, I try to post complete code outlining the whole process of going from raw data to finished visualization. In this case, because the eBird data is quite large, and can be challenging to work with, I've skipped some steps. I've downloaded the whole dataset of [eBird checklists](https://ebird.org/data/download/ebd) (referred to as the "Sampling Event Data" file), imported this text file into a PostgreSQL database, and summarized it into an effort table giving the number of checklists at every location in eBird. Here I connect to PostgreSQL and grab this effort table.


```r
# postgis
pgis <- dbConnect(PostgreSQL(),
                  host = "localhost", port = 5433, dbname = "ebird",
                  user = "postgres", password = "postgres")

# effort from postgis
effort <- st_read_db(pgis, "effort") %>% 
  select(n_checklists) %>% 
  st_transform(crs = proj) %>% 
  as("Spatial")

dbDisconnect(pgis)
```

## eBird Effort

As a useful intermediate step, I'll create a map of eBird effort. First, I generate a raster of the number of eBird checklists per 10km grid cell.


```r
# template raster
r <- raster(xmn = -180, xmx = 180, ymn = -90, ymx = 90, vals = 1)
r <- projectRaster(r, crs = proj)
res(r) <- 10000

# world map
land <- ne_download(scale = 50, type = "land", category = "physical",
                    returnclass = "sf") %>% 
  st_transform(crs = proj)
r_land <- fasterize(land, r)
# country lines
borders <- ne_download(scale = 50, type = "admin_0_boundary_lines_land", 
                       category = "cultural", returnclass = "sf") %>% 
  st_transform(crs = proj)

# rasterize total effort in each cell
f_effort <- here("_source", "data", "inaccessibility", 
                 "ebird-effort_n-checklists.tif")
if (!file.exists(f_effort)) {
  r_effort <- rasterize(effort, r_land, field = "n_checklists", fun = "sum") %>% 
    raster_to_na() %>% 
    writeRaster(f_effort, overwrite = TRUE)
}
r_effort <- stack(f_effort)
```

Now I produce a map of these data. I'm using a log scale here and a palette similar to the [night lights satellite maps](https://www.nasa.gov/feature/goddard/2017/new-night-lights-maps-open-up-possible-real-time-applications) that NASA produces.


```r
# effort maps
# first make a bounding box and graticules
bb <- make_bbox(c(-180, 180), c(-90, 90), spacing = c(NA, 0.1), crs = proj)
grat <- make_graticules(seq(-150, 180, 30), seq(-90, 90, 30), 
                        spacing = c(10, 1), crs = proj)
pal <- c("#570d0e", "#6d1112", "#841116", "#9d1b1b", 
         "#b12d18", "#cd4e17", "#e97307", "#f1a01b", "#fcbd22", "#fdd94d", 
         "#faee80", "#ffffc3") %>%
  colorRampPalette()
# make effort map
trans <- log
itrans <- exp
r_fig <- r_effort %>% trans()
here("img", "inaccessibility", "ebird-effort_n-checklists.png") %>% 
  png(width = 4000, height = 2000, res = 200)
par(mar = c(0, 0, 0, 0), bg = "black")
plot(bb %>% st_geometry(), col = "#01021e", border = NA, axes = FALSE)
plot(grat %>% st_geometry(), col = "#0c1842", lwd = 0.5, add = TRUE)
plot(land %>% st_geometry(), col = "#0c1842", border = NA, add = TRUE)
plot(r_fig, col = pal(256), legend = FALSE, add = TRUE)
plot(borders %>% st_geometry(), col = "#6587ff", lwd = 0.5, add = TRUE)
plot(bb %>% st_geometry(), col = NA, border = "#0c1842", lwd = 3, add = TRUE)
# legend
rng <- cellStats(r_fig, range)
brks <- seq(rng[1], rng[2], length.out = 5)
lbls <- itrans(brks) %>% round() %>% scales::comma()
title <- "eBird Effort (# checklists)"
fields::image.plot(zlim = rng, legend.only = TRUE,
                   col = pal(256),
                   legend.width = 1, horizontal = TRUE,
                   smallplot = c(0.4, 0.6, 0.07, 0.07 + 0.02),
                   axis.args = list(at = brks, labels = lbls, 
                                    line = -1, fg = NA, col.axis = "white"),
                   legend.args = list(text = title, side = 3, col = "white", 
                                      cex = 1.5))
dev.off()
```

eBirders are an adventurous bunch, travelling to far flung destinations to find interesting birds, and every country on Earth has checklists in the eBird database as a result. North America and Western Europe are particularly well covered by eBird data. That said, large parts of the world, especially the North and much of Africa, have only been covered sparsely. This could be an issue of inaccessibility (e.g., the Amazon and the Arctic), safety (e.g., Algeria and Libya), or just that birders are somewhat predictable, mostly following in the footsteps of other birders and visiting known locations to get target species.

Click on the map to view it full size:
<br />
<a href="/img/inaccessibility/ebird-effort_n-checklists.png">
<img src="/img/inaccessibility/ebird-effort_n-checklists.png" alt="eBird Effort" style="display: block; margin: auto; " />
</a>

## Inaccessibility

Now that I've created a map of effort, I can use the `distance()` function from the `raster` package to calculate the distance to the nearest non-`NA` cell in the raster. Since I've set all cells with no checklists to `NA`, this will give the distance to the nearest checklist. Note that for such a large raster, `distance()` will take a long time to run; just under half an hour on my machine.


```r
# inaccessibility map
f_poles <- f_effort <- here("_source", "data", "inaccessibility", 
                            "ebird-inaccessibility_distance-to-checklist.tif")
if (!file.exists(f_poles)) {
  poles <- distance(r_effort, filename = f_poles, overwrite = TRUE)
}
poles <- raster(f_poles)

# map
trans <- function(x) x^(1/3)
itrans <- function(x) x^3
poles_plot <- mask(poles / 1000, r_land) %>% trans()
here("img", "inaccessibility", 
     "ebird-inaccessibility_distance-to-checklist.png") %>% 
  png(width = 4000, height = 2000, res = 200)
par(mar = c(0, 0, 0, 0))
plot(bb %>% st_geometry(), col = "light blue", border = NA, axes = FALSE)
plot(grat %>% st_geometry(), col = "grey20", lwd = 0.25, add = TRUE)
plot(poles_plot, col = pal(256) %>% rev(), legend = FALSE, add = TRUE)
plot(borders %>% st_geometry(), col = "black", lwd = 0.5, add = TRUE)
plot(bb %>% st_geometry(), col = NA, border = "grey20", lwd = 3, add = TRUE)
# legend
rng <- cellStats(poles_plot, range)
brks <- seq(rng[1], rng[2], length.out = 5)
lbls <- itrans(brks) %>% round() %>% scales::comma()
title <- paste("eBird Poles of Inaccessibility",
               "Distance to closest checklist (km)", sep = "\n")
fields::image.plot(zlim = rng, legend.only = TRUE,
                   col = pal(256) %>% rev(),
                   legend.width = 1, horizontal = TRUE,
                   smallplot = c(0.4, 0.6, 0.15, 0.15 + 0.02),
                   axis.args = list(at = brks, labels = lbls, 
                                    line = -1, fg = NA, col.axis = "black"),
                   legend.args = list(text = title, side = 3, col = "black",
                                      cex = 1.5))
dev.off()
```

Unsurprisingly, polar areas are the least birder, but there's also lots of other areas that are quite far from an eBird checklist, for example parts of North Africa and the Amazon. Click for the full size map:

<br />
<a href="/img/inaccessibility/ebird-inaccessibility_distance-to-checklist.png">
<img src="/img/inaccessibility/ebird-inaccessibility_distance-to-checklist.png" alt="eBird Effort" style="display: block; margin: auto; " />
</a>

