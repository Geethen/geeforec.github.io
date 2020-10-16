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

1. Filter Feature Collections.
2. Manipulate a dataset.
3. Plot a time-series over multiple regions.
4. Create a table and export data as a csv.

In this practical we will be using the Hansen forest loss dataset to look at forest loss over time over all of Madagascar and then within Protected Areas.

**Importing data**

We will start by loading in the World Database on Protected Areas, which we will use later in Part B. 

```js
var WDPA = ee.FeatureCollection("WCMC/WDPA/current/polygons");
```

We then load in the International Boundaries dataset and filtering by selecting the country of interest. 

```js
var countries = ee.FeatureCollection('USDOS/LSIB_SIMPLE/2017');
var madag = countries.filter(ee.Filter.eq('country_na', 'Madagascar'));
```

Next we will add in the most recent Hansen global forest change dataset, clip it to our area of interest (Madagascar) and then select the bands of interest, namely tree cover in 2000, overall forest loss over the time period (2000-2019) and the forest loss per year band.

```js
var hansen2019 = ee.Image("UMD/hansen/global_forest_change_2019_v1_7").clip(madag);
var treeCover = hansen2019.select(['treecover2000']);
var lossImage = hansen2019.select(['loss']);
var lossYear = hansen2019.select(['lossyear']);
```

We can then add the tree cover 2000 and overall forest loss bands to our map. My masking each image with itself, values that are 0 become transparent pixels.

```js
Map.centerObject(madag, 6);
Map.addLayer(treeCover.mask(treeCover), {palette: ['000000', '00FF00'], min: 0, max: 100}, 'Forest Cover');
Map.addLayer(lossImage.mask(lossImage), {palette: ['FF0000']}, 'Forest Loss');
```

**Data processing & manipulation**

There are several steps required in processing this dataset. The goal here is for to become familiar with the different functions available and be able to manipulate ee.Objects into features, images, dictionaries and collections.

We start with filtering our tree cover dataset to only include forested areas (our threshold here is 30% tree cover). To do this we use the gte() function, which provides us with a binary output. Any value equal to or over 30% gets a 1 and any value below this gets a 0. We then mask our forest loss image, so as to only include forest loss and not just tree cover loss. 

```js
var forestCover = treeCover.gte(30);

// Create a mask to set nodata where lossyear = 0 and tree cover in 2000 < 30%; apply the mask
var mask = lossYear.neq(0).and(forestCover.eq(1));
var lossYear_mask = lossYear.mask(mask);
```

**Part A: find tree cover loss over all of Madagascar**

We will now get into the details of manipulating our data for visualizing it later. The first step is to create 1) a feature collection for exporting a csv and 2) an array for plotting forest loss in Madagascar for 2000 to 2019. 

As we currently have forest loss per pixel (30 m resolution), we want to convert our loss image to values of km2. We use the ee.Image.pixelArea() function for this, which gives an output in m2, which we then divide by 1e6 to get km2.

```js
var lossAreaImage = lossImage.multiply(ee.Image.pixelArea()).divide(1e6);
print(lossAreaImage, 'lossAreaImage');
```

We now apply a reducer over the region. First we add our loss image together with our loss year image and then sum values for each year (or group). The group() function requires a groupField argument, which in this case is the second band in our image (loss year) and is coded as 1.

The reduceRegion() function then requires our geometry and scale. We also add in the argument for maxPixels, to make sure we do not cap our GEE size limit. 
```js
// Reduce the loss area image to an object with a summed value for each year
var lossByYear = lossAreaImage.addBands(lossYear_mask).reduceRegion({
  reducer: ee.Reducer.sum().group({ // we use the group function to calculate sum per year, and select the groupField 1 to specify lossYear
    groupField: 1
    }),
  geometry: madag,
  scale: 30,
  maxPixels: 1e10 // due to the size of the geometry we may exceed the maxPixels allowed, so we increase this to a large value
});
print(lossByYear, 'lossByYear');
```

The output of the above processing is an ee.Object, which we now want to convert to a FeatureCollection for export. To do this we map over the function ee.Feature for each Object in the list to produce a list of features. The last step is to convert this to a FeatureCollection and export to our Google Drive account as a table.

```js
// Convert our loss by year dictionary to a list, and then convert each valye to a feature for output.
var output = ee.FeatureCollection(ee.List(lossByYear.get('groups'))).map(function(pair) {
  return ee.Feature(null, pair);
});
print(output, 'output');

// To export a table, we need to convert our features to a FeatureCollection
Export.table.toDrive(output,'Madagascar_yearly_forest_loss_km2');
```

We can then use this feature collection to plot the forest loss over all of Madagascar per year. The inputs for ui.Chart.Feature.byFeature() are the FeatureCollection, the xProperty and the yProperties.

```js
var chart = ui.Chart.feature.byFeature({
  features: output,
  xProperty: 'group',
  yProperties: 'sum'
}).setChartType('LineChart').setOptions({
    title: 'Yearly Forest Loss in Madagascar',
    hAxis: {title: 'Year'},
    vAxis: {title: 'Area (square km)'},
    legend: {position: "none"},
    lineWidth: 1,
    pointSize: 3
  });
print(chart);
```

**Part B: find forest loss over selected Protected Areas**

For this next part, we will create a similar dataset, but this time we will produce an ImageCollection with each Image representing the summed forest loss over each year. 

To start we will create a list of years and then use the masked loss year image that we used in Part A to produce a list of Images. We will then convert this list to an ImageCollection. Note: we need to convert our pixels to area again.

```js
var years = ee.List.sequence(1,19);

var loss_by_year = years.map(function(year) {
    return lossYear_mask.eq(ee.Image.constant(year)).multiply(ee.Image.pixelArea()).divide(1e6).set('year', year);
});
print(loss_by_year, 'loss by year');

var loss_stack = ee.ImageCollection(loss_by_year);
print(loss_stack, "loss_stack");
```

We now return to the World Database on Protected Areas. We want to filter out specific Protected Areas based on a property. In this case we use their name, coded as 'NAME'. To find out the names of these protected areas, either go to the WDPA webpage (https://www.protectedplanet.net/en) or alternatively add the full FeatureCollection to your map and use the inspector tool to find their names. Here we select 3 protected areas in Madagascar to demostrate the use of a time-series over multiple regions. 

```js
var selected_PAs = WDPA.filter(ee.Filter.or(
  ee.Filter.eq("NAME", "Marolambo"),
  ee.Filter.eq("NAME", "Corridor Forestier Ambositra Vondrozo"),
  ee.Filter.eq("NAME", "Befotaka Midongy")
  ));
// print(select_PAs);
Map.addLayer(selected_PAs.draw({color: 'white', strokeWidth: 2}),{}, 'Selected PAs');
```



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
var loss_year = Hansen.select(['lossyear']).clip(regions);
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

\*_Visualize forest loss_

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