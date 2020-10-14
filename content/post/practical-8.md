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

Species Distribution Models are a valuable tool in ecology and conservation, providing us with insights into global change impacts, allowing us to project potential future range shifts in species. Using geo-referenced species localities and environmental predictors we can determine the important environmental conditions for species at their known sites.

However, they need to be used carefully and with full understanding of the models used and outputs provided. For example, we may not have a full sample of species localities or not have all of the relevant environmental variables. This means we need to have a strong understanding of the species sampling, biogeography and potentially biotic interactions to accurately predict their distributions. Take a look at [this](https://damariszurell.github.io/SDM-Intro/) R tutorial for more information on SDMs. 

SDMs are not a feature commonly used in GEE and therefore the documentation does not have full support as other algorithms or processes may have. With further use of this platform this is likely to change. The strong potential of GEE in the SDM space is the ability to process large amounts of data very quickly.

**Importing data**

For this practical we will need two core datasets. Our geo-referenced species localities or presences and our geographic layers of environmental predictor variables. In addition to this we will need a chosen area of interest.

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

We then load in the bioclimatic variables from the WorldClim dataset. The WorldClim data provides average bioclimatic conditions for the entire globe between the period 1960-1991 and is commonly used in SDMs.

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

If we wanted to add more predictor variables that we thought may be important in predicting _Bradypus_ distributions, we can call another ImageCollection. In this example, which we won't use, we call the TerraClimate dataset, find the mean value for all bands over the full time period, select the variables we are interested in and clip it to our area of interest. We would then merge this together with the WorldClim data using addBands().

```js
// var terraclim = ee.Image(ee.ImageCollection("IDAHO_EPSCOR/TERRACLIMATE").mean()).select(['aet','def','pdsi','pet','soil']).clip(countries_clip);
// var vars = worldclim.addBands(terraclim);
```

Based on the **R dismo** tutorials, these are the bioclim variables typically used for _Bradypus_ SDMs.

```js
var vars = worldclim.select(['bio01','bio05','bio06','bio07','bio08','bio12','bio16','bio17'])
```

**Processing data**

We now want to create pseudo-absence points, merge this together with our presence points and provide a binary presence property to each observation.  The first step is to make sure all of our presence points are within our region. We then add 1 to all presence localilities. We then need to create our random pseudo-absence points and add 0 to each one. We create the same number of pseudo-absence points as there are presences, but this may depend on which model you are training your data on. Note that the generation of pseudo-absence points has a lot of literature related to it and should be carefully considered before running any SDM. View [this](https://doi.org/10.1111/j.2041-210X.2011.00172.x) this article for more information on pseudo-absences in SDMs.

Lastly, we merge the datasets together to give a single FeatureCollection of all points.

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

The last step in the data processing is to extract the values of each band of our predictor variables for each point in our dataset. We do this using the sampleRegions() function.

```js
var sampleData = vars.sampleRegions({
  collection: points,
  properties: ['Presence'],
  scale: 1000
});
```

**Fit our classifier using Random Forest**

There are several different options for classifiers in GEE, which can be viewed in the Docs tab by typing ee.Classifier. We will be using the smileRandomForest() function as it allows for variable importance values to be extracted (which is currently not available for MaxEnt models in GEE). There are several options one can add to fine-tune the model to your own specifications. For this function, the only argument that is required is the number of decision trees to use.

```js
var label = 'Presence';

var model = ee.Classifier.smileRandomForest({numberOfTrees: 1000})
                            .setOutputMode('PROBABILITY')
                            .train(sampleData, label, bands);
print(model, "model output")
```

We will then extract the variable importance data and add it to a chart. To pull the information from the RandomForest model, we use explain() and then get() to pull the specific information we are interested in.

We then add this to a chart for visualization. The RandomForest in GEE uses the Gini index for variable importance measures.

```js
var importance = model.explain().get('importance');
print(importance, "variable importance");

// Convert the importance values into a feature for plotting
var importance_plot = ee.Feature(null, ee.Dictionary(importance));

// Plot the resulting variable importance in a bar chart
var chart =
ui.Chart.feature.byProperty(importance_plot)
.setChartType('ColumnChart')
.setOptions({
title: 'Random Forest Variable Importance',
legend: {position: 'none'},
hAxis: {title: 'Bands'},
vAxis: {title: 'Importance'}
});
print(chart);
```

**Model classification/prediction**

This is a simple step. We classify or predict the output of the model, based on selected predictor variables. In this case, we use the full suite of variables used in the original model classification.

```js
var prediction = vars.classify(model);
```

**Visualize the predicted distribution**

First we will load in a custom palette for the visualization. Next step is to add it to our map.

```js
var palettes = require('users/gena/packages:palettes');
var palette = palettes.matplotlib.magma[7];

Map.addLayer(prediction, {palette: palette},'Probability of occurence');
```

**Ensemble methods**

In many cases, you will not rely on a single model and it may be preferable to use many models and find an average over several models. A short example of how to do this, is to produce a new model - this time using MaxEnt - and finding the mean over our two predicted classifications. 

```js
var model2 = ee.Classifier.gmoMaxEnt()
                            .setOutputMode('PROBABILITY')
                            .train(sampleData, label, bands);

var prediction2 = vars.classify(model2);
```

**Model evaluation**

An important last step in all classification modelling is to determine the accuracy of the probability of occurrence or presence/absence map. There are several methods available to produce metrics on model (in)accuracy, which will be discussed in the next practical. 

Save your script.

**Practical 8 Exercise**

For this excercise, use an alternative dataset of species localities. This time *Solanum acuale*, a plant species known as Wild potato from South America. Produce a distribution map using the same WorldClim data as in the practical above. 

Load in the dataset to a new script using the below asset id:

```js
ee.FeatureCollection('users/jdmwhite/solanum_acuale')
```

To share your script, click on Get Link and then copy script path. Send your completed script to **email**