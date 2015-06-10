---
layout:  post
title: "Merging Spatial Data Sources"
author: Jacob Mortensen
date: 2015-06-09
categories:
  - spatial statistics
tags: [R, GIS, merging-data, spatial-statistics]
---

Currently my research focuses on how environmental heat affects health outcomes such as morbidity and mortality.
In order to be certain that the effects we are attributing to spatial location and temperature are not being 
confounded with anything else, my advisor and I decided to account for population and age in our model. 
This information is readily obtained from the 2010 census, but unfortunately the blocks used by the census do not line
up very neatly with the spatial structure we are using for our analysis. This post will demonstrate how
to use R to combine information from two differently shaped spatial sources.

### Data

For my research we have divided the city of Houston into 1428 grid cells, each 1 km\\(^2\\). Over that same area
there are 1536 irregularly shaped census blocks. Each block contains a variety of information, such as average household size, 
number of hispanics, blacks, whites and asians, and number of males and females. We will be focusing on two specific variables:
total population and percentage of the population over 65. Our challenge is to figure out how to merge this data
with our grid cells despite the fact that the census blocks do not line up well with the grid cells.

<figure>
  <img src="/assets/article_images/2015-06-09-merging_spatial_data/censusBlocks.png" style="width: 49%; display: inline-block">
  <img src="/assets/article_images/2015-06-09-merging_spatial_data/gridCells.png" style="width: 49%; display: inline-block">
  <figcaption>On the left is a map of the census blocks which contain the data we are trying to merge with the grid cells in
    the map on the right. Clearly they do not line up neatly.</figcaption>
</figure>



### The Setup

In order to merge this data we will be using R and we will need to load
the following packages:

```R
library(sp)
library(rgeos)
library(GISTools)
```

In our research so far we have not actually had to represent our grid cells as cells. Instead we have just been using
the latitude and longitude coordinates for the center of each cell. This means that rather than having a grid we simply 
have a collection of points. These points are contained within the object `pp.grid`. Therefore, our first order of business
is to convert these points into a grid so that we can see how each cell overlaps with the census blocks. This can 
be done with the following code:

```R
# pp.grid[, 2] is longitude, pp.grid[, 1] is latitude.
# We use pp.grid to create a SpatialPoints object, which 
# is easily converted to a grid.
sp.pts <- SpatialPoints(cbind(pp.grid[, 2], pp.grid[, 1])) 
sp.grd <- points2grid(sp.pts, tolerance = 0.000763359)
grd.poly <- as.SpatialPolygons.GridTopology(sp.grd)

# points2grid creates a rectangular grid over the entire box that
# bounds pp.grid so we we use gContains to eliminate those grid 
# cells that do not contain one of our desired grid points.
grd.bool <- ifelse(colSums(gContains(grd.poly, sp.pts, byid=T)) > 0, TRUE, FALSE)
grd.layer <- SpatialPolygonsDataFrame(grd.poly[grd.bool],
                data = data.frame(ID = c(1:length(sp.pts))),
                match.ID = FALSE)

# Row names are used when calculating the intersection, 
# and we deleted many cells in the previous step, so
# we update the row names in order to simplify things.
row.names(grd.layer) <- paste("g", 1:length(sp.pts), sep="")
```

Now that we have converted our points into a grid, we need to load the 2010 census data into R, and calculate
the intersection between the grid and the census blocks. 

```R
# Read in the 2010 census shapefile
census <- readShapePoly("census2010.shp")

# R treats all of the variables associated with the shapefile
# as factors so here we convert them to numerics
census$TotalPop <- as.numeric(as.character(census$TotalPop))
census$PCTover65 <- as.numeric(as.character(census$PCTover65))

# The census shapefile covers a much larger area than just
# Houston, so before we calculate the intersection we cut down the 
# area of the census we are intersecting with in order to speed up 
# computation. Note that gIntersects just returns a logical vector,
# it has nothing to do with actually calculating the intersection.
cen.bool <- ifelse(colSums(gIntersects(census, grd.layer, byid=T)) > 0, TRUE, FALSE)
census.layer <- census[cen.bool, ]

# Here is where the intersection is calculated. gIntersection 
# returns a SpatialPolygonDataFrame that has a polygon for every 
# intersected shape in the two input objects.
int.res <- gIntersection(grd.layer, census.layer, byid=T)
```

Understanding the output of `gIntersection` is easier with a picture. The following
graphic shows that each area of the census blocks that intersects a grid cell 
becomes its own polygon. 
<figure>
  <img src="/assets/article_images/2015-06-09-merging_spatial_data/intersection.png">
  <figcaption>Here we see that the intersecting spaces of the grid cells and the census
    blocks have each been made into individual polygons.</figcaption>
</figure>

Total population and percent over 65 represent two different kinds of data: count and a proportion.
As a result they have to be approached differently. First we will demonstrate how to do this
for the population variable.

```R 
# Get row names for grd.layer and census.layer
tmp <- strsplit(names(int.res), " ")
grid.id <- sapply(tmp, "[[", 1)
census.id <- sapply(tmp, "[[", 2)

# Calculate areas for census blocks, grid cells, and the
# intersecting areas.
census.areas <- gArea(census.layer, byid=T)
grid.areas <- gArea(grd.layer, byid=T)
int.areas <- gArea(int.res, byid=T)

# Make sure that census.areas is the same length as
# int.areas and that it is lined up properly.
cens.index <- match(census.id, row.names(census.layer))
census.areas <- census.areas[cens.index]

# Calculate the proportion between intersection areas and 
# the census blocks.
census.prop <- zapsmall(int.areas / census.areas, 3)

# Use the proportion to approximate the population
# within each intersection area.
pop <- zapsmall(census.layer$TotalPop[cens.index] * census.prop, 5)
```

Now that we have approximated the population in each intersection area, we can turn our 
attention to calculating the percentage over 65. With population we took the total
population in a census block and assigned a proportion of that count to the intersection
area according to the percentage of census block area covered by the intersection area. 
To try and divide up a percentage in this same way would be incorrect. Instead, we think
of the intersection areas in relation to the areas of each grid cell. 

If a grid cell is entirely contained within a census block, then the intersection area for that grid cell is
just the cell itself, and so the whole cell will be given the same percentage over 65 as 
the census block in which it is contained. If the grid cell is divided between two census
blocks, so that 40% of its total area is within one block and 60% is in another, then the
percent over 65 within the two census blocks will be multiplied by the appropriate percentage.
These two numbers will then be added together to get the approximate percentage over 65 for a given grid cell. 
The following code shows how this can be done in R.

```R
# Make sure that grid.areas is the same length as 
# int.areas and that it is lined up properly.
grid.areas <- grid.areas[grid.id]

# Calculate the proportion of intersection area with 
# respect to it's containing grid cell.
grid.prop <- zapsmall(int.areas / grid.areas, 3)

# Calculate the percentage over 65 within each intersection area.
pct.65 <- zapsmall(census.layer$PCTover65[cens.index] * grid.prop, 5)
```

Now that we have calculated the total population and percent over 65 for each intersection area
we need to combine them into the grid cell areas. We do this by creating a data.frame and then
combining the variables based on grid.id.

```R
df <- data.frame(grid.id, pct.65, pop)

int.pct.65 <- xtabs(df$pct.65 ~ df$grid.id)
int.pop <- xtabs(df$pop ~ df$grid.id)

# In order to make sure that we combine these variables
# with our grid in the proper order we need to calculate
# this index.
index <- as.numeric(gsub("g", "", names(int.pct.65))
grid.pct.65 <- numeric(length(grd.layer))
grid.pop <- numeric(length(grd.layer))
grid.pct.65[index] <- int.pct.65
grid.pop[index] <- int.pop

# Combine this data with the grd.layer. Note that we
# include the coordinates at this point so that we 
# can later match this data with the points in pp.grid.
grid.w.data <- SpatialPolygonsDataFrame(grd.layer,
                data = data.frame(data.frame(grd.layer),
                                  grid.pop,
                                  grid.pct.65,
                                  coordinates(grd.layer)),
                match.ID = FALSE)

```

Now we are able to plot this data and see how the census data 
compares to the plots we've created with the grid cells. 

<figure>
  <img src="/assets/article_images/2015-06-09-merging_spatial_data/TotalPopulation.png">
  <figcaption><bold>Total Population data for the census blocks and the grid cells.</bold></figcaption>
</figure>

When examining the plots of the population count data we might be concerned about some of the differences
between the two plots. First, we see that some of the areas in the upper right corner of the grid cell
plot do not appear to be as densely populated as those in the census blocks. However, note that the population
for these larger areas is being distributed equally among many smaller grid cells, accounting for this difference.
Another potential concern is the difference in the legends. However, these were simply generated automatically
using the auto.shading function in R and are accounting for the difference in the ranges of the two data sets.

<figure>
  <img src="/assets/article_images/2015-06-09-merging_spatial_data/PercentOver65.png">
  <figcaption><bold>Plots showing the percentage of the population that is over 65 within each 
  census block and each grid cell.</bold></figcaption>
</figure>

Here we see that the plots showing the percentage of the population over 65 appear to be consistent between the census
blocks and the grid cells. 

Having examined the plots and concluding that everything looks okay, we have just one more step before we are 
able to use this data in our model. We need to make sure that it lines up with `pp.grid`. We do this in the
following code.

```R
grid.df <- data.frame(grid.w.data)
names(grid.df) <- c("ID", 
                    "Population", 
                    "PercentOver65", 
                    "Longitude", 
                    "Latitude")

# By ordering this by longitude first and then by latitude,
# we match the ordering of the coordinates in pp.grid
grid.df <- grid.df[order(grid.df$Longitude, grid.df$Latitude), ]
pp.grid.w.data <- cbind(pp.grid, grid.df)
```

Now we have successfully taken the census data and merged it with our grid cells. The complete code can be found [here][1].
For more information about using R to perform GIS tasks like we've done here check out [*An Introduction to R for Spatial Analysis
& Mapping* by Brunsdon & Comber][2]. 

[1]: https://github.com/jwmortensen/research-code/blob/master/mergeInterceptData.R "github"
[2]: http://www.amazon.com/An-Introduction-Spatial-Analysis-Mapping/dp/1446272958/ref=tmm_pap_title_0 "Introduction to R for Spatial Analysis on Amazon"
