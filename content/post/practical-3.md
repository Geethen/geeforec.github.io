+++
authors = []
date = 2020-10-21T22:00:00Z
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

We start by creating a polygon. This can be done using the polygon tool.

We then filter the ImageCollection by time and space. Lastly, we filter it by the amount of cloud cover to remove excessively cloudy scenes.

```js
// Create image collection of S-2 imagery for the full year of 2019
var s2 = ee.ImageCollection('COPERNICUS/S2_SR')
//filter start and end date
.filterDate('2019-01-01', '2019-12-31')
//filter according to drawn boundary
.filterBounds(geometry)
// pre-filter to get less cloudy images (only keeps images with less than 20% cloudy pixels)
.filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',20))
;
```

**Write and map a function**

We will now use our first function. This function creates a cloud mask based on the metadata within each image of the collection. Look up the band information in the Sentinel-2 metadata. We create a variable name for the function called maskS2clouds. We then apply a set of functions for each image in the ImageCollection. Make sure the new variables within your function are consistent. This is the default cloud masking process provided by GEE for Sentinel 2 images, though this is frequently updated when better data becomes available. See: [https://developers.google.com/earth-engine/datasets/catalog/COPERNICUS_S2_SR](https://developers.google.com/earth-engine/datasets/catalog/COPERNICUS_S2_SR "https://developers.google.com/earth-engine/datasets/catalog/COPERNICUS_S2_SR")

We then use the map() function to apply a built-in algorithm or our own function over the Sentinel-2 collection.

```js
// Function to mask cloud from built-in quality band
// information on cloud
var maskS2clouds = function(image) {
  var qa = image.select('QA60');

  // Bits 10 and 11 are clouds and cirrus, respectively.
  var cloudBitMask = 1 << 10;
  var cirrusBitMask = 1 << 11;

  // Both flags should be set to zero, indicating clear conditions.
  var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
      .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

  return image.updateMask(mask);
};

// run the mask cloud function over each image in the s2 collection
var s2_cloudmask = s2.map(maskS2clouds); 
```

The above code is complicated, so let's make sure we are happy with what it is doing. Let's do this by plotting a unmasked vs. masked image.

```js
// Let's take a look at what this function is doing...

// Print the unmasked dataset sorting it from the most to least cloudy image
print(s2.sort('CLOUDY_PIXEL_PERCENTAGE', false), 'S2 collection clouds');

// There are two ways to plot the cloudiest image

// First select the image by it's ID
// Map.addLayer(ee.Image('COPERNICUS/S2_SR/20190618T020449_20190618T021219_T50HNH'), 
//               {min:0, max:3000, bands:['B4','B3','B2']}, 'Cloudy image ID');
// Second, we can select the most cloudy image by sorting the list in descending 
// order and selecting the first image
Map.addLayer(s2.sort('CLOUDY_PIXEL_PERCENTAGE', false).first(), 
              {min:0, max:3000, bands:['B4','B3','B2']}, 'Cloudy image first');

// Let's now plot the image with the cloud mask applied
Map.addLayer(s2_cloudmask.sort('CLOUDY_PIXEL_PERCENTAGE', false).first(), 
              {min:0, max:3000, bands:['B4','B3','B2']}, 'Cloud masked image');
```

Next we will make a custom function to clip the image to our AOI and add a band to the image containing NDVI values. We will use the normalizedDifference() function and apply it over the near infra-red and red bands. Lastly, we will rename the new band to ‘NDVI’. Note that the function is now nested inside the map function. Print out the new ImageCollection to view the new band.

```js
var s2_ndvi = s2_cloudmask.map(function(image) {
    var s2_clip = image.clip(geometry)
    return s2_clip.addBands(s2_clip.normalizedDifference(['B8', 'B4']).rename('NDVI'))
});

print(s2_ndvi, 's2 with NDVI');
```

![](/images/prac3_ndvi_band.png)

**Visualization**

Next we will create a time-series plot over the NDVI band of the ImageCollection. This is done using the ui.Chart series of functions. The function you chose depends on the type of data you are using. In this case, we are running an image series over a single region, so we will use ui.Chart.image.series(). The primary inputs here are: the ImageCollection, area of interest (geometry), a reducer, the band of interest, the scale and the x-axis property (which defaults to 'system:time_start'). In GEE, calculations that summarise your data are called reducers and can be called using the ee.Reducer series of functions. Here we will use ee.Reducer.median() to calculate the median NDVI values across our area of interest.

Lastly, we can specify details for the chart, including the type of chart and then label options. Run print() to see the output of the chart in your console. Hover your cursor over the chart to see interactive details. For more details on customizing your charts see: https://developers.google.com/chart/interactive/docs

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

![](/images/prac3_ndvi_chart.png)

It is useful to visualize this data, to see a representation of your variable of interest. Let's take a look at the seasonal lows and highs for NDVI. First, extract the NDVI band and create a new ImageCollection. Next, filter the NDVI collection for January and August and then reduce this to a single image by calculating the median value for each pixel over each month.

```js
var NDVI = s2_ndvi.select(['NDVI']);

var NDVI_jan = NDVI.filterDate('2019-01-01', '2019-02-01').median();
var NDVI_aug = NDVI.filterDate('2019-08-01', '2019-09-01').median();
```

There are several options for creating interesting visualizations. We can make a custom palette or alternatively we can load in pre-built palettes. Here we will load in pre-built palettes from another users GEE repository and select the Red-Yellow-Green palette and specify the number of color values we want (i.e. 9). To view more palettes see: https://github.com/gee-community/ee-palettes

```js
var palettes = require('users/gena/packages:palettes');
var ndvi_pal = palettes.colorbrewer.RdYlGn[9];
```

The next step is to view the NDVI images on our map. Note the use of palette here.

```js
Map.centerObject(geometry);

Map.addLayer(NDVI_jan, {min:0, max:1, palette: ndvi_pal}, 'NDVI Jan');
Map.addLayer(NDVI_aug, {min:0, max:1, palette: ndvi_pal}, 'NDVI Aug');
```

![](/images/prac3_ndvi_vis.png)

**Exporting data**

The last step is to export the values of the time-series into a csv. The easiest way to do this is to open the plot in another window, using the pop out button. You have options to download the values of the time series as a csv or alternatively export the plot itsefl as an svg or png.

A next step, which allows flexibility in analysis or visualizations in different softwares is to download a resulting image. GEE allows outputs to be exported to the users Google Drive account. This is limited by the storage size of your Google Drive account and the memory provided to each GEE users. We use the Export series of function. In this case for an image, we use Export.image.toDriver() and specify a number of variables. Once this line of code is run, you will need to go to your task tab to execute the task.

```js
Export.image.toDrive({
  image: NDVI_aug,
  description: 'NDVI Aug example',
  scale: 100 // this can go down to 10, to match S2 original resolution
});
```

![](/images/prac3_tasks.png)

Save your script.

**Practical 3 Exercise**

Repeat this practical but use the Landsat-8 dataset and provide a new area of interest. Play around with extending the filterDate duration, the size of your area of interest, and the scale used ui.Chart. Take note that the ImageCollection size may produce memory errors.

To share your script, click on Get Link and then copy the script path. Send your completed script to **ots.online.education@gmail.com.**

Do you have any feedback for this practical? Please complete this quick (2-5 min) survey [here](https://forms.gle/hT11ReQpvG2oLDxF7).