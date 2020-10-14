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

**Importing data**

We start by creating a polygon. This can be done using the polygon tool or by specifying the coordinates for each point of the polygon as shown below. We then filter the ImageCollection by time and space.

```js
var geometry = ee.Geometry.Polygon([
[116.44967929649718,-33.98379973082278],[117.14593784141906,-33.98379973082278],
[117.14593784141906,-33.62548866260113],[116.44967929649718,-33.62548866260113],
[116.44967929649718,-33.98379973082278]]);

var s2 = ee.ImageCollection('COPERNICUS/S2')
.filterDate('2019-01-01', '2019-12-31')
.filterBounds(geometry);
```

**Write and map a function**

We will now write our first function. This function creates a mask cloud based on the metadata within each image of the collection. Look up the band information in the Sentinel-2 metadata. We create a variable name for the function as maskcloud. We then apply a set of functions for each image in the ImageCollection. Make sure the new variables within your function are consistent. First, we clip the image by our area of interest. Then we select the cloud mask band and lastly, return an image that has the mask applied to it. 

We then use the map() function to apply a built-in algorithm or our own function over the Sentinel-2 collection. 

```js
var maskcloud = function(image) {
var clipped = image.clip(geometry);
var QA60 = clipped.select(['QA60']); // select the cloud mask band
return clipped.updateMask(QA60.lt(1)); // mask image at all pixels that are not zero
};

var s2_cloudmask = s2.map(maskcloud); 
```

Next we will make a custom function to add a band to the image containing NDVI values. We will use the normalizedDifference() function and apply it over the near infra-red and red bands. Lastly, we will rename the new band to ‘NDVI’. Note that the function is now nested inside the map function. Print out the new ImageCollection to view the new band.

```js
var s2_ndvi = s2_cloudmask.map(function(image) {
return image.addBands(image.normalizedDifference(['B8', 'B4']).rename('NDVI'))
});

print(s2_ndvi, 's2 with NDVI');
```

**Visualization**

Next we will create a time-series plot over the NDVI band of the ImageCollection. This is done using the ui.Chart series of functions. The function you chose depends on the type of data you are using. In this case, we are running an image series over a single region, so we will use ui.Chart.image.series(). The primary inputs here are: the ImageCollection, area of interest (geometry), a reducer, the band of interest, the scale and the x-axis property (which defaults to 'system:time_start'). In GEE, calculations that summarise your data are called reducers and can be called using the ee.Reducer series of functions. Here we will use ee.Reducer.mean() to calculate the mean NDVI values across our area of interest. 

Lastly, we can specify details for the chart, including the type of chart and then label options. Run print() to see the outpot of the chart in your console. Hover your cursor over the chart to see interactive details. For more details on customizing your charts see: https://developers.google.com/chart/interactive/docs

```js
var plotNDVI = ui.Chart.image.series(s2_ndvi, geometry, ee.Reducer.mean(), // we use an image based chart, with image, geom & reducer
'NDVI', 500, 'system:time_start') // band, scale, x-axis property, label
              .setChartType('LineChart').setOptions({
                title: 'NDVI time series',
                hAxis: {title: 'Date'},
                vAxis: {title: 'NDVI'}
});

print(plotNDVI);
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