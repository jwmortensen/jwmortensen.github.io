---
layout: post
title: "Mapping spatial data with R and Mapbox"
date: 2015-04-30
categories: statistics
tags: [featured, mapbox, qgis, mapping, R]
---
Although R has fairly advanced graphics capabilities, when it comes to mapping
spatial data there are some alternatives that can produce much more detailed
and visually appealing graphics. One such alternative is [Mapbox][1], a company
committed to helping others easily create beautiful, detailed maps.
In this post I will show how to convert point data to a shapefile in R, and then 
demonstrate how to style and map it using Mapbox.

### Preliminaries
Before we dive into how to do this make sure that you have an account with Mapbox,
which can be done [here][2]. They offer a starter tier that is free for anyone, 
and if you're a student you can upgrade to their basic account without any additional
cost. Mapbox has a lot of really impressive tools, but for this tutorial we are going 
to keep it simple.

Once you've created your account you will need to download 
[TileMill][3]<a href="#tile-mill-footnote"><sup>1</sup></a>. This is the tool that 
we will use to style our map and generate raster tiles to upload to Mapbox.

If you'd like to follow along on your own machine, the code and data for this tutorial
can be found [here][4].

Now let's fire up R and make sure you have the following packages installed
and loaded:

```R
# Install packages if necessary
install.packages(c("sp", "rgeos", "FNN", "GISTools", "RColorBrewer")) 

library(sp) # SpatialPoints and SpatialPolygons objects
library(rgeos) # gContains function
library(FNN) # nearest neighbors function
library(GISTools) # choropleth maps
library(RColorBrewer) # good color palettes for mapping
```

### Data
The data that we will be using was generated as part of the research I am doing
for my master's thesis. It consists of the probability that a heat-related 911
call comes from one of 1,428 1 km\\(^2\\) grid cells in Houston, TX. If you 
```load("./mappingData.RData")```
you will have access to a data frame called ```map.df``` that consists of 
three columns: longitude and latitude coordinates for the center of each grid cell
and the probability that a 911 call comes from within that grid cell. 


### Converting grid points into grid cells
In the data frame we have the longitude and latitude coordinates, which are 
just points. So the first thing that we want to do is to figure out how to 
convert our points into a spatial grid. We'll do this using the ```sp``` and 
```rgeos``` packages.

```R
# Create SpatialPoints object from the coordinates in map.df.
sp.pts <- SpatialPoints(cbind(map.df$lon, map.df$lat))

# Convert the points into a grid.
# Note that we have to set the tolerance because the
# grid points are not perfectly equidistant.
sp.grd <- points2grid(sp.pts, tolerance=0.000763359) 
grd.layer <- as.SpatialPolygons.GridTopology(sp.grd)
```
At this point we have a grid that looks like this:

<img src="/assets/article_images/mapbox_and_R/full_grid.png"/>

but Houston looks like this:

<img src="/assets/article_images/mapbox_and_R/houston.png"/>

so we want to eliminate any cell in the grid that doesn't
actually contain one of our Houston grid points. We
can do that using the ```gContains``` function.

```R 
# Eliminate unnecessary grid cells
grd.cont <- ifelse(colSums(gContains(grd.layer, sp.pts, byid=T)) > 0, TRUE, FALSE)
grd.layer <- grd.layer[grd.cont]
```

Now ```grd.layer``` should look like this:

<img src="/assets/article_images/mapbox_and_R/houston_grid.png"/>

which is what we want. 

### Connecting data with the grid
So far, ```grd.layer``` is just a grid, it doesn't actually contain any
of the information that we're interested in. We need to combine the data
about the probability of a 911 call with the grid that we've created in 
order to map it. We can do this with the following code:

```R
# SpatialPolygonsDataFrame uses row names as ID's for matching,
# so we assign row names to grd.layer to ensure we match them 
# properly with the information in map.df
row.names(grd.layer) <- as.character(1:nrow(map.df))
# Use get.knnx to use the latitude and longitude coordinates
# to match up probabilities with the appropriate grid cell
k.nn <- get.knnx(coordinates(grd.layer), cbind(map.df$lon, map.df$lat), 1)
# Change row.names of map.df so they match grd.layer
row.names(map.df) <- k.nn$nn.index
grd.layer.df <- SpatialPolygonsDataFrame(grd.layer, map.df, match.ID = T)
```
Now grd.layer.df should be a grid that also contains the information
that we're interested. We can use the ```choropleth``` function from 
the ```GISTools``` package to plot this and make sure that it looks okay.

```R
shades <- auto.shading(grd.layer.df$prob.911, n = 9, cols = brewer.pal(9, "Greens"))
choropleth(grd.layer, grd.layer.df$prob.911, shades)
```
This code produces this plot

<img style="align: center" src="/assets/article_images/mapbox_and_R/choro_map.png"/>

which looks as we expect it to, and so we are now ready to export
this data as a shapefile and begin styling it in mapbox. Export it
by running:

```R
writePolyShape(grd.layer.df, "911probabilities")
```


### Importing shapefile into TileMill
Fire up TileMill and click on "New Project". This will open
the following window:

<img src="/assets/article_images/mapbox_and_R/tilemill_ss1.png"/>

Most of the information you enter doesn't really matter,
but make sure that you uncheck the "Default data" box. For our
purposes we do not want any pre-existing styles. 

Now when you open the project, all you will see is a blank blue 
background. Right now there is no data in our project, so we 
need to import the shapefile that we generated from R. In the 
bottom left you will see a button that looks like three sheets of paper
stacked on top of each other. Select this, and then click on 
"Add layer."

<img src="/assets/article_images/mapbox_and_R/tilemill_ss2.png"/>

In the window that this opens, click on the "Browse" button next
to the Datasource field and find the shapefile that you exported from R.
Now, in the line that is labeled "SRS" click the dropdown and select 
"WGS84." 

<img src="/assets/article_images/mapbox_and_R/tilemill_ss3.png"/>

Because the earth is round, but we typically depict data
on a flat plane, we have to tell the system how to project the 
information we feed it. Oftentimes a shapefile will include this information,
but the shapefile we exported from R does not, so we have to specify it manually. 
That's what we're doing here.

After changing the projection to "WGS84", click "Save & Style."

### Using TileMill to style data
In order to change the appearance of the map, TileMill uses a styling
language called CartoCSS. If you are familiar with CSS then CartoCSS will 
be very intuitive to you, but if you aren't, then basically CartoCSS is a 
language that allows you to select certain features of your data and apply
different styles to change their appearance. If you would like a more in-depth
explanation of CartoCSS and what styles you can use, 
Mapbox has provided a good reference [here][5], but for the purposes of this
tutorial you can skip it for now.

First, go to the bottom left and click the "layers" button (the icon that looks like
stacked sheets of paper) and click the icon that looks like a magnifying glass.

<img src="/assets/article_images/mapbox_and_R/tilemill_ss4.png"/>

This should bring the shapefile into view in the window so we can see how
the appearance changes as the different styles we apply take effect. 

Now on the right side of the main TileMill window there should be a
tab that looks something like this:

<img src="/assets/article_images/mapbox_and_R/tilemill_ss5.png"/>

This is where we will specify all of the styles to change the appearance of our
map. The first thing we need to do is to delete

```CSS
Map {
  background-color: #b8dee6;
}
```

which gets rid of the default blue background. Now our stylesheet should 
contain only the following text:

```CSS
#911probabilities {
  line-color:#594;
  line-width:0.5;
  polygon-opacity:1;
  polygon-fill:#ae8;
}
```

Go ahead and delete everything within the braces so we are left with just

```CSS
#911probabilities {
}
```
```#911probabilities``` is the ID for our shapefile layer, so when we want to
change the way it looks we will use this ID. If we are curious about what information
is contained in our shapefile, you can click on the layers button, and then click the 
icon that looks like a little spreadsheet. This will open up a table that allows
you to inspect the data connected to the shapefile. From this we can see that
```prob_911``` has values that range from nearly 0 to about 0.009. We are going to 
divide this range into 11 approximately evenly spaced categories, 
and then color each category to make it easy to identify
which areas of the city have the highest probabilities. 
The color palette we are going to use is one of the "diverging" color
palettes available from [Color Brewer][6].

One nice thing about CartoCSS is that
it allows you to apply styles conditionally using syntax of the form 

```CSS
[condition] { 
  style to be applied if condition is true;
}
```
We are interested in applying styles based on different values of ```prob_911``` so 
update the ```style.mss``` file so that it looks like this:

```CSS
#911probabilities {
  [prob_911 > 0] {
    polygon-fill:#a50026;
    polygon-opacity: 0.7;
  }
  [prob_911 < 0.007] {
    polygon-fill: #d73027;
  }
  [prob_911 < 0.005] {
    polygon-fill: #f46d43;
  }
  [prob_911 < 0.004] {
    polygon-fill: #fdae61;
  }
  [prob_911 < 0.003] {
    polygon-fill: #fee08b; 
  }
  [prob_911 < 0.0025] {
    polygon-fill: #ffffbf;
  }
  [prob_911 < 0.002] {
    polygon-fill: #d9ef8b;
  }
  [prob_911 < 0.0015] {
    polygon-fill: #a6d96a;
  }
  [prob_911 < 0.001] {
    polygon-fill: #66bd63;
  }
  [prob_911 < 0.0005] {
    polygon-fill: #1a9850;
  }
  [prob_911 < 0.0001] {
    polygon-fill: #006837;
  }
}
```

We can also add a legend. It requires a little bit of knowledge of HTML
and CSS, which is beyond the scope of this tutorial, but it is not
hard to learn, and a good crash course is available for free from 
[Codecademy][7]. For now, go to the bottom left corner of the window
and click the hand icon. 

<img src="/assets/article_images/mapbox_and_R/tilemill_ss6.png"/>

Then, in the legend field, copy and paste the following code:

```html
<div class='my-legend'>
<div class='legend-title'>Predicted Probabilities</div>
<div class='legend-scale'>
  <ul class='legend-labels'>
    <li><span style='background:#006837;'></span>0 - 0.0009</li>
    <li><span style='background:#1a9850;'></span>0.0001 - 0.00049</li>
    <li><span style='background:#66bd63;'></span>0.0005 - 0.0009</li>
    <li><span style='background:#a6d96a;'></span>0.001 - 0.00149</li>
    <li><span style='background:#d9ef8b;'></span>0.0015 - 0.0019</li>
    <li><span style='background:#ffffbf;'></span>0.002 - 0.00249</li>
    <li><span style='background:#fee08b;'></span>0.0025 - 0.0029</li>
    <li><span style='background:#fdae61;'></span>0.003 - 0.0039</li>
    <li><span style='background:#f46d43;'></span>0.004 - 0.0049</li>
    <li><span style='background:#d73027;'></span>0.005 - 0.0069</li>
    <li><span style='background:#a50026;'></span>> 0.007</li>
  </ul>
</div>
</div>

<style type='text/css'>
  .my-legend .legend-title {
    text-align: center;
    margin-bottom: 5px;
    font-weight: bold;
    font-size: 90%;
    }
  .my-legend .legend-scale ul {
    margin: 0;
    margin-bottom: 5px;
    padding: 0;
    float: left;
    list-style: none;
    }
  .my-legend .legend-scale ul li {
    font-size: 80%;
    list-style: none;
    margin-left: 0;
    line-height: 18px;
    margin-bottom: 2px;
    }
  .my-legend ul.legend-labels li span {
    display: block;
    float: left;
    height: 16px;
    width: 30px;
    margin-right: 5px;
    margin-left: 0;
    border: 1px solid #999;
    }
  .my-legend .legend-source {
    font-size: 70%;
    color: #999;
    clear: both;
    }
  .my-legend a {
    color: #777;
    }
</style>
```

Now if we hit the "save" button our changes will take effect and our map 
should now look something like this:

<img src="/assets/article_images/mapbox_and_R/tilemill_ss7.png"/>

With our map created, we are ready to export it, upload it to mapbox, 
and overlay it on an existing map. To do this, click on "Export" in the top
right corner, and select "MBTiles." 

<img src="/assets/article_images/mapbox_and_R/tilemill_ss8.png"/>

This will open a window with a few options.
The defaults won't really work for us. They assume you want to export the entire world
at a very high level of detail, which would create a massive file, and we don't want to 
do that. So we need to adjust zoom (level of detail), center and bounds. 
I suggest that you set the zoom to go from 0 to 16. That will still give you plenty of
detail at the street level, while keeping the file a manageable size. Then
for the center enter ```-95.363,29.7593,11``` and for the bounds enter 
```-95.7239,29.5209,-95.062,30.1523```, which will change it so that you export 
only the city of Houston. If you don't want to have to copy and paste this information
ever again, you can click the box that says "Save settings to project" in order to save
the center and bounds information. 

<img src="/assets/article_images/mapbox_and_R/tilemill_ss9.png"/>

With these settings in place, click export. This will export the files, which may take 
a few seconds, but then you will still need to save them somewhere on your computer, 
so click save and decide where you want to put them. 

<img src="/assets/article_images/mapbox_and_R/tilemill_ss10.png"/>

With that completed, we are now ready to upload these tiles to mapbox and add them 
to a map.


### Adding tiles to an existing map

Go to [mapbox.com][1] and make sure you're logged in, then click the "data" button
in the upper right corner so you get to a page that looks like this:

<img src="/assets/article_images/mapbox_and_R/mapbox_ss1.png"/>

Click "uploads" and when that page loads, choose the .mbtiles file that we exported
from TileMill and upload it. It will take a few seconds to upload, and then up to a 
few minutes for mapbox to process the tiles before we can use them. When that is 
completed, click the "Projects" button in the navigation bar and then select 
"New Project," which will open this window:

<img src="/assets/article_images/mapbox_and_R/mapbox_ss2.png"/>

There are a lot of options available here, but we're going to keep it pretty simple.
First we are going to select a map to place underneath our data. You can experiment
with the different map designs. I'm going to work with the "Outdoors" map. After 
selecting a map design, click the magnifying glass icon and type in "Houston, TX" to
focus the map on Houston. Then select "Data," click the icon on the far right that is
three horizontal lines stacked on each other, and toggle the button below it so it goes
from "features" to "layers."

<img src="/assets/article_images/mapbox_and_R/mapbox_ss3.png"/>

In order to add the tiles that we uploaded previously, click "toggle layers" and then
click on the appropriate layer. If you're doing this for the first time,
there should only be one option. Once you've done that, hit save, and you're finished.
You should now have access to a beautiful map that looks like this:

<iframe width='100%' height='500px' frameBorder='0'
src='https://a.tiles.mapbox.com/v4/jwmortensen.n3ljfm63/attribution,zoompan,zoomwheel,geocoder,share.html?access_token=pk.eyJ1Ijoiandtb3J0ZW5zZW4iLCJhIjoiQjJHSVp4NCJ9.AYH98hv0ksUCLvwmsJHfeQ'></iframe>


<hr>
<a name="tile-mill-footnote">1</a>: Mapbox 
recently introduced a new tool called Mapbox Studio and have stopped active development
on TileMill but unfortunately Mapbox Studio doesn't produce raster tiles. You can
recreate what we will do here in Mapbox Studio using a style, but Mapbox limits the number
of styles you can upload (only 1 for the free tier) whereas it treats raster files 
as data and allows you to upload as many as you want until you reach the storage limit for
your plan (100 mb for the free plan). This is why TileMill is used instead of Mapbox Studio.


[1]: https://www.mapbox.com/ "Mapbox"
[2]: https://www.mapbox.com/signup/ "Mapbox Registration"
[3]: https://www.mapbox.com/tilemill/ "TileMill"
[4]: https://github.com/jwmortensen/mapbox-and-r 'github repository with all the code'
[5]: https://github.com/mapbox/carto/blob/master/docs/latest.md "CartoCSS Reference"
[6]: http://colorbrewer2.org/ "Color Brewer"
[7]: https://www.codecademy.com/tracks/web "Codecademy"
