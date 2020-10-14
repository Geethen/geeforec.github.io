+++
authors = []
date = 2020-10-21T22:00:00Z
draft = true
excerpt = "Forest loss over multiple regions"
hero = "/images/prac3_hero_both.png"
timeToRead = 3
title = "Practical 5"

+++
**Practical 5:** Forest loss over multiple regions

Access the completed practical script [here](https://code.earthengine.google.com/?scriptPath=users%2Fjdmwhite%2FOTS-GEE4EC%3APractical_5%2FForest_loss_over_multiple_regions)

**Learning Objectives**

By the end of this practical you should be able to:

1. Filter, create and merge Feature Collections.
2. Manipulate a dataset.
3. Plot a time-series over multiple regions.
4. Create a table and export data as a csv.
5. Export a video of forest change using Landsat 8

In this practical we will be using the Hansen forest loss dataset to look at forest loss over time over a Protected Area and a neighbouring region outside of the Protected Area.

**Importing data**

We will start by loading in the World Database on Protected Areas and filtering the database by selecting specific protected area by its name. To find other protected areas, you can add the full FeatureCollection to the map and use the inspector tool to find their name.

```js
var WDPA = ee.FeatureCollection("WCMC/WDPA/current/polygons");
// Map.addLayer(WDPA, {},"WDPA");

// select specific PA by name 
var PAs = WDPA.filter(ee.Filter.or(
  ee.Filter.eq("NAME", "WaiWai")
  ));
print(PAs);
```

We will then create a polygon that neighbours the WaiWai protected area to give us an indication of whether protected areas have done a good job at protecting forests. To do this we can draw a polygon using the geometry tools or create our own list with the specified coordinates for the polygons. Once we have created our new polygon outside of the protected area, we can merge the two FeatureCollections together. Make sure you use the same label for each Feature, in this case 'NAME'. We can then add the two regions to our map.

```js
var features = [
  ee.Feature(ee.Geometry.Polygon([[-59.28733022709222,1.1991686972459115],
[-59.95749624271722,1.0398973999767975],
[-59.98496206302972,0.5524276267636883],
[-59.291450100139095,0.7213319252602786],
[-59.28733022709222,1.1991686972459115]]),
  {NAME: 'WaiWai_out'})
];

var out_FC = ee.FeatureCollection(features);

var regions = PAs.merge(out_FC);

Map.centerObject(regions);
Map.addLayer(regions, {},'regions');
```

Next we will add in the most recent Hansen global forest change dataset, clip it to our regions and extract the band for tree cover in 2000.

```js
var Hansen = ee.Image('UMD/hansen/global_forest_change_2019_v1_7').clip(regions);
print(Hansen, "Hansen");

var cover_2000 = Hansen.select(['treecover2000']);
```

**Manipulating data**

We want to select pixels for where tree cover in 2000 was 50% or over. Using the gte() (greater than or equal to) function, we will convert all numbers to a binary output, so that any value above 50% is converted to a 1 and anything below 0. We will then add it to our map.

```js
var trees_2000 = cover_2000.gte(50);
Map.addLayer(trees_2000_clip, {min: 0, max: 1, palette: ['white','#1e8f1d']}, "trees_2000");
```

Next step is select the loss year band from the Hansen data, create a mask where tree cover was lower than 50% or where there was no tree loss. We then apply this mask to our loss_year image. 

```js
var mask = loss_year.neq(0).and(trees_2000.eq(1));
var loss_year_null = loss_year.mask(mask);
```

To create a loss layer for each year, we first make a sequence of years and then map this over masked loss_year_null object. Next we convert the list of images into an ImageCollection.

```js
var years = ee.List.sequence(1,19);

var loss_by_year = years.map(function(year) {
    return loss_year_null.eq(ee.Image.constant(year)).set('year', year);
});

var loss_stack = ee.ImageCollection(loss_by_year);
print(loss_stack, "loss_stack");
```

**Visualize forest loss*

We can now sum the values to create a visualization of pixels over the full time period where forest loss has occurred. 

```js
var loss_stack_clip = loss_stack.sum().clip(regions);
Map.addLayer(loss_stack_clip, {palette: 'yellow'},"loss_stack");
```

Next step is to produce a time-series chart showing forest loss over our two regions over the full time period. Note that we now use image.seriesByRegion to specify there are multiple regions of interest.

```js
var forestlossPlot = ui.Chart.image.seriesByRegion(
    loss_stack, regions, ee.Reducer.sum(), 'lossyear', 30, 'year', 'NAME')
        .setChartType('LineChart')
        .setOptions({
          title: 'Forest loss over selected features',
          vAxis: {title: 'Forest loss summed per pixel'},
          lineWidth: 1,
          pointSize: 4
});

print(forestlossPlot);
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

![](/images/prac3_ndvi_band.png)

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

![](/images/prac3_ndvi_tasks.png)

Save your script.

**Practical 3 Exercise**

Repeat this practical but use the Landsat-8 dataset and provide a new area of interest. Play around with extending the filterDate duration, the size of your area of interest, and the scale used ui.Chart. Take note that the ImageCollection size may produce memory errors.

To share your script, click on Get Link and then copy script path. Send your completed script to **email**