+++
authors = []
date = 2020-10-21T22:00:00Z
draft = true
excerpt = "NDVI time-series over a single region"
hero = "/images/prac3_hero_both.png"
timeToRead = 3
title = "Practical 3"

+++
**Practical 3: NDVI time-series over a single region**

Access the completed practical script [here](https://code.earthengine.google.com/?scriptPath=users%2Fjdmwhite%2FOTS-GEE4EC%3APractical_3%2FNDVI_timeseries_single)

**Learning Objectives**

By the end of this practical you should be able to:

1. Write a custom function.
2. Map (loop) it over a collection.
3. Plot a time-series.
4. Export data as a csv and image.

**Access your code editor**

The first step is to access the GEE code editor. This can be done from the earth engine [home page](https://earthengine.google.com/) by going to platform --> code editor. Alternatively, you can access it directly at https://code.earthengine.google.com/

**Part A**

**Importing datasets**

![](/images/practical_1_importing_image.png)

There are two ways to import datasets into the GEE code editor. We will run through both of these in this practical. The first method is to use the search bar. We will be using the NASA SRTM Digital Elevation Data 30m in the first half of this practical. In the search bar, type in elevation and select the SRTM dataset. This will also bring up the metadata for the chosen dataset. Take a look at the information provided regarding the processing of the data, dataset time periods, resolution of the bands, scaling factors (which are unique to Google's ingestion of the data) and reference to the data source or journal article.

![](/images/practical_1_importing_image2.png)

Import the SRTM data by selecting 'Import' in the bottom right hand corner. Once the dataset is in your Imports section of the code editor, rename it 'srtm'.

An alternative and a more reproducible method is to call the dataset directly into your code editor. For this dataset, which has no temporal component, we use the function ee.Image(), insert the dataset string between the brackets and save the image as an object using: “var srtm = “. We can then print the image to the console to explore the details of the data using the print() function.

```js
var srtm = ee.Image("USGS/SRTMGL1_003");

print(srtm);
```

**Visualization**

We now want to add the SRTM data to the map. We do this using the Map.addLayer() function.

```js
Map.addLayer(srtm);
```

You should notice two things: 1) the visualization shows very little detail and 2) we have output the image for the full global dataset.

First, let’s center the interactive map on a chosen point. We can use two approaches here a) we can find the latitude/longitude coordinates using the Inspector tool and then copy and paste these values into the Map.setCenter() function, together with the zoom level.

```js
Map.setCenter(-84.006204, 10.431206, 10);
```

Let’s now look at visualization parameters. The SRTM visualization parameters need to be changed to produce better image. This is done within the Map.addLayer() function, by changing the minimum and maximum values.

```js
Map.addLayer(srtm, {min: 0, max: 3500});
```

To clip the dataset to a smaller region, which can be important with big datasets, we need a specified area of interest. Use the polygon tool to create a geometry for clipping the SRTM data. This is polygon is called a feature. More than one feature make a FeatureCollection. Add the clipped data to the map, but this time we will add in a label for the image layer.

```js
var srtm_clip = srtm.clip(polygon);
Map.addLayer(srtm_clip, {min: 0, max: 3500}, 'Elevation above sea level');
```

Lastly, let’s further customise the visualisation, by adding in a colour palette.

```js
Map.addLayer(srtm_clip, {min: 0, max: 3500, palette: ['blue','yellow','red']},'Elevation above sea level (palette)');
```

The last step is to save your script. First, create your own repository and provide a name (e.g. GEE4EC). Secondly, save the script as Practical 1a.

**Part B**

Start a new script and name it Practical 1b.

While we could process data for a large region and over a long time period, this is typically not required, slows down your script and may exceed the amount of memory provided to each user by GEE.

Using the Sentinel-2 dataset, let’s filter the dataset temporally and spatially, using filterDate() and filterBounds().

The first step is to load the full Sentinel-2 dataset, using ee.ImageCollection() and provide an area of interest using ee.Geometry.Point()

```js
var s2_coll = ee.ImageCollection("COPERNICUS/S2");

var aoi = ee.Geometry.Point([-35.008609532093686,-7.846847534023778])
```

We now want to apply filters to reduce our data to a time and space that we’re interested in. As Sentinel-2 is measuring surface reflectance and often has clouds, we also want to filter our ImageCollection based on properties related to clouds. We sort the data into ascending order, which then allows us to select the first image, using the .first() function, of the sorted collection, which will have the least cloud cover.

```js
var s2 = ee.Image(s2_coll
.filterDate("2018-01-01","2019-12-31")
.filterBounds(aoi)
.sort("CLOUD_COVERAGE_ASSESSMENT")
.first()
// Alternatively, you can calculate a median value for all pixels over the time period
// .median()
);

print(s2, 'Sentinel 2 image');
```

We have now reduced the ImageCollection (several images together) into a single Image. Now add the Image to the map. We will first center the map on our area of interest and then add the Sentinel-2 image to the map, selecting the red, green and blue bands in the visualisation parameters options to make a true colour composite image.

In the Sentinel-2 metadata the red, green and blue bands are represented by B4, B3 and B2, respectively. We will need to add these into the correct channels.

```js
Map.centerObject(aoi, 13);

Map.addLayer(s2, {bands:['B4','B3','B2']}, 'No defined vis parameters');
```

![](/images/practical_1_no_vis.png)

Without specifying the minimum and maximum values, the image does not display correctly. Go to layers box and select the wheel icon and manually define the vis parameters. Use the custom drop-down menu and stretch the minimum and maximum values to 100% and click apply. We can also do this by selecting the min and max values within the visualization parameters.

```js
Map.addLayer(s2, {bands:['B4','B3','B2'], min:0, max: 3000}, 'True-colour');
```

![](/images/practical_1_true.png)

We can use different bands to highlight specific properties that we may be interested in. Using the near infra-red band (B8 for Sentinel-2), we can highlight the high reflectance that healthy vegetation has in this band.

```js
Map.addLayer(s2, {bands:['B8','B4','B3'], min:0, max: 3000}, 'False-colour');
```

![](/images/practical_1_false.png)

As a last step, save the script.

**Practical 1 Exercise**

Repeat this practical but use the Landsat-8 dataset. Produce a false colour image using the near infra-red band for a region in your country. Hint: the band names for Landsat-8 and Sentinel-2 satellites are different. Explore the dataset metadata to find the correct band names.

```js
ee.ImageCollection("LANDSAT/LC08/C01/T1_TOA ");
```

To share your script, click on Get Link and then copy script path. Send your completed script to **email**

![](/images/practical_1_script_path.png)