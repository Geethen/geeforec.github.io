+++
authors = []
date = 2020-10-21T22:00:00Z
excerpt = "Forest loss over multiple regions"
hero = "/images/prac5_hero.png"
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

![](/images/prac5_cover.png)

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

![](/images/prac5_loss_full.png)

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

We can now add this to a chart. We will use the image.seriesByRegion() chart option. This requires your ImageCollection, FeatureCollection, a reducer, the band of interest, the scale for the reducer, the xProperty and lastly the seriesProperty.

```js
var forestlossPlot = ui.Chart.image.seriesByRegion(
    loss_stack, selected_PAs, ee.Reducer.sum(), 'lossyear', 30, 'year', 'NAME')
        .setChartType('LineChart')
        .setOptions({
          title: 'Yearly Forest Loss over selected PAs',
          hAxis: {title: 'Year'},
          vAxis: {title: 'Area (square km)'},
          lineWidth: 1,
          pointSize: 3
});
print(forestlossPlot);
```

![](/images/prac5_loss_PAs.png)

Save your script.

**Practical 5 Exercise**

Repeat this practical but use an alternative country as a region of interest for Part A and use three alternative Protected Areas for Part B. 

Send your completed script to **email**