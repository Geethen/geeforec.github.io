+++
authors = []
date = 2020-10-28T22:00:00Z
draft = true
excerpt = "Species distribution modeling (classification)"
hero = "/images/prac8_hero.png"
timeToRead = 3
title = "Practical 8"

+++
**Practical 8: Species distribution modeling (classification)**

Access the completed practical script [here](https://code.earthengine.google.com/?scriptPath=users%2Fjdmwhite%2FOTS-GEE4EC%3APractical_8%2FSpecies_distribution_modeling)

**Learning Objectives**

By the end of this practical you should be able to:

1. Have a basic understanding of classifications and their value to Species Distribution Models (SDMs).
2. Fit a random forest model on bioclimatic variables.
3. Predict the distribution of a chosen species.
4. Consider variable importance & understand need for model evaluation. 

**Importing data**

For this dataset we will need two core datasets. Our species localities or presences and our environmental predictor variables. In addition to this we will need a chosen area of interest. 

Our first step is to load the presence data for our species of interest - _Bradypus variegatus_ - a species commonly used Species Distribution Modeling tutorials. This data has been extracted from the **R dismo** package. This dataset has already been uploaded as an asset and made publicly available. 

```js
var presences = ee.FeatureCollection("users/jdmwhite/bradypus");
```

Next step is load in the countries dataset, as well as a polygon to filter this. We will use filterBounds() to extract only the area we are interested in. We then use union to merge all of the countries into a single feature and then map, together with the presence data. 

```js
var countries = ee.FeatureCollection("USDOS/LSIB_SIMPLE/2017");

// Create a polygon.
var polygon = ee.Geometry.Polygon([
[-87.14743379727207,-34.736435036461145],[-32.12790254727207,-34.736435036461145],[-32.12790254727207,16.642228542503663],
[-87.14743379727207,16.642228542503663],[-87.14743379727207,-34.736435036461145]
]);

var countries_clip = countries.filterBounds(polygon).map(function(f) {
  return f.intersection(polygon, 1);//1 refers to the maxError argument
});

var countries_clip = countries_clip.union();
Map.addLayer(countries_clip, {},"Area of interest");
Map.addLayer(presences, {color: 'red'},'Bradypus variegatus localities');
```

We then load in the bioclimatic variables from the WorldClim dataset.

```js
var worldclim = ee.Image("WORLDCLIM/V1/BIO").clip(countries_clip);
print(worldclim);
```

A crucial step in many classification approaches to make sure that your predictor variables are uncorrelated. Here is a piece of code that provides this by producing a correlation matrix of pearson correlation coefficient, though we will not run it in this practical, as it is rather slow. Can you work out why this code may be slow? (Hint: client vs. server side functions).

```js
// // Correlation matrix
// var BandsBioClim=['bio01','bio05','bio06','bio07','bio08','bio12','bio16','bio17'];
// var ClimList=ee.List(BandsBioClim).getInfo(); //Needs .getInfo() >>takes longer

// var numBand= 8; 
// var CorThresh =0.8; //Set correlation threshold
// var Matrix = ee.List([]); 

// function GenerateMatrix (InMatrix) {
//   for (var i = 0; i < numBand; i++) {
//     var getBandA = ClimList[i];
//     for (var k = 0; k < numBand; k++) {
//       if (k!==i){
//         var getBandB = ClimList[k];
//         var pearson2 = (ee.Image(worldclim)).select([getBandA, getBandB])
//           .reduceRegion({
//             reducer: ee.Reducer.pearsonsCorrelation(),
//               geometry: geometry,
//               scale: 3000
//               });
//         var Cor = pearson2.get('correlation');
//         var CorVal= Cor.getInfo();
//         if (CorVal > CorThresh) {
//         // print(getBandA, getBandB, CorVal); //WANT INFO ON
//           var Row=ee.List([[getBandA, getBandB, CorVal]]);
//           //print("Row is:",Row),
//           InMatrix=InMatrix.add(Row);
//           //print("Matrix Inside:",InMatrix);
//         }// Ends IF (k not i)
//     } //ENDS IF (correlated more than 80%)
//   }  // ENDS k-loop
//   } // ENDS i-loop
// return InMatrix;
// } // End GenerateMatrix

// print("Matrix Outside",GenerateMatrix(Matrix));
```

Based on the **R dismo** tutorials, these are the bioclim variables typically used for Bradypus SDMs. 

```js
var vars = worldclim.select(['bio01','bio05','bio06','bio07','bio08','bio12','bio16','bio17'])
```

**Processing data**

We now want to create pseudo-absence points, merge this together with our presence points and provide a binary presence property to each observation.  The first step is to make sure all of our presence points are within our region. We then add 1 to all presence localilities. We then need to create our random pseudo-absence points and add 0 to each one. We create the same number of pseudo-absence points as there are presences, but this may depend on which model you are training your data on. Lastly, we merge the datasets together to give a single FeatureCollection of all points. 

```js
var filtered_locs = presences.filterBounds(countries_clip);
print('No. of localities', filtered_locs.size());

var Presence = filtered_locs.map(function(feature){
  return feature.set('Presence', 1);
});
print('Check for the Presence property', Presence.limit(5));

var pAbsencePoints = ee.FeatureCollection.randomPoints(countries_clip, 116, 42).map(function(feature){
  return feature.set('Presence', 0); // we then add 0s to all pseudo-absences
});
Map.addLayer(pAbsencePoints, {color: "gray"}, "Pseudo-absence points");

var points = Presence.merge(pAbsencePoints);
print('Check the total no. of points', points.size());
```



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