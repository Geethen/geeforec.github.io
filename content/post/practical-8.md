+++
authors = []
date = 2020-10-28T22:00:00Z
excerpt = "Species distribution modeling (classification)"
hero = "/images/prac8_hero.png"
timeToRead = 3
title = "Practical 8"

+++
**Practical 8: Species distribution modelling**

Authored by: Joseph White

Access the completed practical script [here](https://code.earthengine.google.com/d8912b87a20af22464a7bb091e543fcb)

**Learning Objectives**

By the end of this practical you should be able to:

1. Build an Image of selected environmental covariates
2. Extract covariates.
3. Model fitting.
4. Discuss the need for model evaluation.

Species Distribution Models are a valuable tool in ecology and conservation, providing us with insights into global change impacts, allowing us to project potential future range shifts in species. Using geo-referenced species localities and environmental predictors we can determine the important environmental conditions for species at their known sites.

However, they need to be used carefully and with full understanding of the models used and outputs provided. For example, we may not have a full sample of species localities or not have all of the relevant environmental variables. This means we need to have a strong understanding of the species sampling, biogeography and potentially biotic interactions to accurately predict their distributions. Take a look at [this](https://damariszurell.github.io/SDM-Intro/) R tutorial for more information on SDMs.

SDMs are not a feature commonly used in GEE and therefore the documentation does not have full support as other algorithms or processes may have. With further use of this platform this is likely to change. The strong potential of GEE in the SDM space is the ability to process large amounts of data very quickly. Additionally, there is a large value in extracting covariate data to complete SDM analyses in other programs. 

**Importing data**

For this practical we will need two core datasets. Our geo-referenced species localities or presences and our geographic layers of environmental predictor variables. In addition to this we will need a chosen area of interest.

Our first step is to load in the LSIB countries dataset and a polygon of interest:

```js
var countries = ee.FeatureCollection("USDOS/LSIB_SIMPLE/2017");
var polygon = ee.Geometry.Polygon([
    [-87.14743379727207,-34.736435036461145],[-32.12790254727207,-34.736435036461145],[-32.12790254727207,16.642228542503663],
    [-87.14743379727207,16.642228542503663],[-87.14743379727207,-34.736435036461145]
    ]);
```

We then import the environmental covariates. The first group of covariates is from the WorldClim dataset. The WorldClim data provides average bioclimatic conditions for the entire globe between the period 1960-1991 and is commonly used in SDMs. The second group is from the TerraClimate dataset, which has information related to climatic water balance for global terrestrial surfaces from 1958-present. Lastly, we import SRTM for terrain data.

```js
// Load in bioclimatic variables from WorldClim 
var worldclim = ee.Image("WORLDCLIM/V1/BIO");
// Load in terraclim data
var terraclim = ee.ImageCollection("IDAHO_EPSCOR/TERRACLIMATE");
// Load in terrain data
var elev = ee.Image("USGS/SRTMGL1_003");
```

The last dataset we require is our species locality information. We will use data sourced from GBIF (Global Biodiversity Information Facility), which has a massive online collection of freely available biodiversity data. Our species of interest - _Solanum acuale_ - is a species commonly used Species Distribution Modeling tutorials. This data has been extracted from the **R dismo** package. This dataset has already been uploaded as an asset and made publicly available. We load this data in as an asset and then add it to our map. 

```js
var presences = ee.FeatureCollection("users/jdmwhite/solanum_acuale");
Map.addLayer(presences, {color: 'red'},'Solanum acuale presences', false);
```

**Pre-processing data**

We want to create a polygon that only includes terrestrial sources. We will use our imported polygon together with the countries database to create our area of interest. We use the intersection() function to 'clip' the countries dataset to the polygon. We then combine our new aoi into a single feature using the union() function, which effectively removes the countries borders. Lastly, we add our polygon and aoi to the map.

```js
var aoi = countries.filterBounds(polygon).map(function(f) {
      return f.intersection(polygon, 1);//1 refers to the maxError argument
    });
var aoi = aoi.union();
Map.addLayer(polygon,{}, "Polygon", false);
Map.addLayer(aoi, {},"Area of interest", false);
```

The next step is to do some pre-processing on our environmental covariates. As the TerraClimate data has monthly values over a large period, we want to find a mean value over this full time period. First we select the variables of interest to use and then find their mean value for each band.

```js
var terraclim = ee.Image(terraclim.select(['aet','def','pdsi','pet','soil']).mean());
```

For terrain, we use the elevation data to find both the aspect and slope for each pixel, using the ee.Terrain() group of functions. We add these bands together to make a full terrain dataset. 

```js
var terrain = elev.addBands(ee.Terrain.aspect(elev)).addBands(ee.Terrain.slope(elev));
```

We can now merge our full covariate dataset together, using the addBands() function. At the same time, we can now clip this to our aoi. Using this approach you can add in extra bands from any other dataset to your covariates.

```js
var vars = worldclim.addBands(terraclim).addBands(terrain).clip(aoi);
print('Check all covariates:', vars);
```

At this point it is valuable to calculate a correlation matrix and select only variables that are uncorrelated below a certain threshold. For GEE script on this, see [https://code.earthengine.google.com/27557ebe40e549f604ed1005e047b75a](https://code.earthengine.google.com/27557ebe40e549f604ed1005e047b75a "https://code.earthengine.google.com/27557ebe40e549f604ed1005e047b75a")

After running this code, you may want to select only a sub-sample of data. Alternatively, you want to create a principal component based on your highly correlated covariates. 

In this example for ease of computation, comment out the full covariate dataset above and go ahead and only use a sub-sample of worldclim covariates below:

```js
var vars = worldclim.select(['bio01','bio05','bio06','bio07','bio08','bio12','bio16','bio17']).clip(aoi);
print(vars, "Selected variables");
```

We now want to create pseudo-absence points, merge this together with our presence points and provide a binary presence property to each observation.  The first step is to make sure all of our presence points are within our region. We then add 1 to all presence localilities. We then need to create our random pseudo-absence points and add 0 to each one. We create the same number of pseudo-absence points as there are presences, but this may depend on which model you are training your data on. Note that the generation of pseudo-absence points has a lot of literature related to it and should be carefully considered before running any SDM. View [this](https://doi.org/10.1111/j.2041-210X.2011.00172.x) this article for more information on pseudo-absences in SDMs.

Lastly, we merge the datasets together to give a single FeatureCollection of all points.

```js
// Make sure all of the presence points are within our AOI
var filtered_locs = presences.filterBounds(aoi);
// How many presence points do we have?
print('No. of localities:', filtered_locs.size());

//add a presence property to each feature i.e. a value of 1 to represent presence
var Presence = filtered_locs.map(function(feature){
  return feature.set('Presence', 1);
});

//create random pseudo-absence points and add 0 value to the presence property for absence
// check the Docs for the randomPoints function; requires region, number of points to generate & a seed
var pAbsencePoints = ee.FeatureCollection.randomPoints(aoi, filtered_locs.size(), 42).map(function(feature){
  return feature.set('Presence', 0); // we then add 0s to all pseudo-absences
});
// You need to be careful with how you chose your pseudo-absences...


// Add pseudo-absence points to the map
Map.addLayer(pAbsencePoints, {color: "gray"}, "Pseudo-absence points", false);

//Merge the presence and pseudo-absence points into a single feature
var points = Presence.merge(pAbsencePoints);
print('Check the full points dataset:', points);
```

![](/images/prac8_p-a.png)

For later model evaluation, it is important to have a set of data for 'training' the model and another set of data for 'testing' the model. We do this by adding in a random column and filtering the dataset based on these new random numbers, selecting 80% of the data for training and 20% for testing.

```js
// To do this, we will split the data into 80% for training and 20% for testing
// Add a random column by default named 'random'
var new_table = points.randomColumn({seed: 42}); 
var training = new_table.filter(ee.Filter.lt('random', 0.80));
// print("Check training data:", training);
var test = new_table.filter(ee.Filter.gte('random', 0.80));
// print("Check test data:", test);
```

The last step in the data processing is to extract the values of each band of our predictor variables for each point in our dataset. We do this using the sampleRegions() function.

```js
var trainingData = vars.sampleRegions({
  collection: training,
  properties: ['Presence'],
  // geometries: true,
  scale: 1000
});
print('Check sampled data:', trainingData.limit(5));
```

At this point, many users may want to export the locality data, together with the covariate data to use it in a different program where there is more support for SDMs. So we will export the data before continuing. You can export it as a shapefile or csv.

```js
Export.table.toDrive({collection: trainingData, 
                          description: 'Solanum_acuale_sampled',
                          fileFormat: 'SHP'});
```

**Analysis: Fit our classifier using Random Forest**

There are several different options for classifiers in GEE, which can be viewed in the Docs tab by typing ee.Classifier. We will be using the smileRandomForest() function as it allows for variable importance values to be extracted (which is currently not available for MaxEnt models in GEE). There are several options one can add to fine-tune the model to your own specifications. For this function, the only argument that is required is the number of decision trees to use. We select the output mode of probability. However, for later model evaluation, you will need to select classification. 

```js
// Pull out the label and band names for the models
var label = 'Presence';
var bands = vars.bandNames();

// We need to specificy the number of trees required; we'll use 100 trees, which can be a good balance
// between under/over-fitting
var model = ee.Classifier.smileRandomForest({numberOfTrees: 100})
                            .setOutputMode('PROBABILITY') 
                            // .setOutputMode('CLASSIFICATION') 
                            .train(trainingData, label, bands);
print("Check model output:", model);
```

**Visualisation**

We will then extract the variable importance data and add it to a chart. To pull the information from the RandomForest model, we use explain() and then get() to pull the specific information we are interested in.

We then add this to a chart for visualization. The RandomForest in GEE uses the Gini index for variable importance measures. Reference the WorldClim dataset to see the names of each bioclim variable or alternatively rename the band names. 

```js
// Variable importance as Gini index
var importance = model.explain().aside(print,'explain model')
                  .get('importance').aside(print, 'Check model importance');

// Convert the importance values into a feature for plotting
var importance_plot = ee.FeatureCollection(ee.Feature(null, ee.Dictionary(importance)
                      // .rename(['bio01','bio05','bio06','bio07','bio08','bio12','bio16','bio17'],
                      //         ['Ann_mean_T','Max_T_warmest_month','Min_T_coldest_month',
                      //         'T_Ann_range','Mean_T_wettest_quarter','Ann_P',
                      //         'P_wettest_quarter','P_driest_quarter'])
                              ));
print('Check importance values:', importance_plot);

// Plot the resulting variable importance in a bar chart
var chart =
ui.Chart.feature.byProperty(importance_plot)
.setChartType('ColumnChart')
.setOptions({
title: 'Random Forest Variable Importance',
legend: {position: 'none'},
hAxis: {title: 'Covariates'},
vAxis: {title: 'Importance'}
});
print(chart);
```

![](/images/prac8_vi.png)

This is a simple step. We classify or predict the output of the model, based on selected predictor variables to find our probability of species occurrence. In this case, we use the full suite of variables used in the original model classification.

```js
var prediction = vars.classify(model);
```

First we will load in a custom palette for the visualization. Next step is to add our predicted species occurrence to our map.

```js
var palettes = require('users/gena/packages:palettes');
var palette = palettes.matplotlib.magma[7];

Map.addLayer(prediction, {palette: palette},'Probability of occurence (RF)');
```

![](/images/prac8_rf.png)

In many cases, you will not rely on a single model and it may be preferable to use many models and find an average over several models. A short example of how to do this, is to produce a new model - this time using MaxEnt - and finding the mean over our two predicted classifications.

First we classify the MaxEnt model. We then create a new ImageCollection of the two SDM models, before finding the mean of the two bands and then adding the ensemble prediction to the map.

```js
var model2 = ee.Classifier.gmoMaxEnt()
                            .setOutputMode('PROBABILITY')
                            .train(trainingData, label, bands);

var prediction2 = vars.classify(model2);
Map.addLayer(prediction2, {palette: palette},'Probability of occurence (MaxEnt)');

var collectionFromImages = ee.ImageCollection.fromImages(
  [ee.Image(prediction), ee.Image(prediction2)]);
print('collectionFromImages: ', collectionFromImages);

var ensemble_prediction = collectionFromImages.mean()

Map.addLayer(ensemble_prediction, {palette: palette},'Probability of occurence (ensemble)');
```

![](/images/prac8_ensemble.png)

A useful extra step is to add a legend to aid our visualisation. This needs to be coded from scratch and can look rather complicated. Be aware that to customise this code for your purpose, there are only a few arguments you may need to change, which have comments next to them. This includes the min and max values, the specific palette of choice, the legend title and lastly the legend position. This legend will work best in an interactive app scenario, as it is not straightforward to export it.

```js
    // Styling for the legend title.
    var LEGEND_TITLE_STYLE = {
      fontSize: '16px',
      // fontWeight: 'bold',
      stretch: 'horizontal',
      textAlign: 'left',
      margin: '4px',
    };
    
    function ColorBar(palette) {
      return ui.Thumbnail({
        image: ee.Image.pixelLonLat().select(0),
        params: {
          bbox: [0, 0, 1, 0.1],
          dimensions: '100x10',
          format: 'png',
          min: 0, // change min value
          max: 1, // change max value
          palette: palette_mag, // chose the correct palette
        },
        style: {stretch: 'horizontal', margin: '0px 8px'},
      });
    }
    
    // Returns our labeled legend, with a color bar and three labels representing
    // the minimum, middle, and maximum values.
    function makeLegend() {
      var labelPanel = ui.Panel(
          [ui.Label('0', {margin: '4px 8px'}), // change min label here
          ui.Label('',{margin: '4px 8px', textAlign: 'center', stretch: 'horizontal'}),
          ui.Label('1', {margin: '4px 8px'})], // change max label here
          ui.Panel.Layout.flow('horizontal'));
      return ui.Panel([ColorBar(palette_mag.palette), labelPanel]);
    }
    
    // Assemble the legend panel.
    Map.add(ui.Panel(
        [
          ui.Label('Probability of occurrence', LEGEND_TITLE_STYLE), makeLegend() // change title here
          ],
        ui.Panel.Layout.flow('vertical'),
        {width: '230px', position: 'bottom-center'})); // change location here to chose where to put legend
```

We can also export our SDM prediction Image as a PNG. Do to this, we use the getThumbURL() function. To increase the resolution of this image, change the dimensions argument for the width. The height will be automatically calibrated. This produces a url for a PNG thumbnail, which can then be downloaded to your computer. 

```js
    // Specify region by your polygon, define the chosen palette & set width size (height adjusts automatically).
    var thumbnail = prediction.getThumbURL({
      palette: palette_mag,
      dimensions: 1000, // change dimensions here for a higher res image
      region: polygon,
      format: 'png'
    });
    print('Output image:', thumbnail);
```

**Model evaluation**

An important last step in all classification modelling is to determine the accuracy of the probability of occurrence or presence/absence map. There are several methods available to produce metrics on model (in)accuracy. The code for this has been included here, so as to produce the a full workflow for GEE classification. However, the metrics will only be covered in depth in the next practical. 

To run these model evaluation metrics, you will need to go back to your ee.Classifier.smileRandomForest() function and change the setOutputMode() from 'PROBABILITY' to 'CLASSIFICATION'.

```js
    var Accuracy = model.confusionMatrix().accuracy();
    print('Training Data Accuracy:', Accuracy);
    // Test accuracy
    var testData = vars.sampleRegions({
      collection: test,
      properties: ['Presence'],
      // geometries: true,
      scale: 1000
    });
    print('Check test data:',testData);
    
    var Test = testData.classify(model);
    print(Test);
    print('ConfusionMatrix', Test.errorMatrix('Presence', 'classification'));
    print('TestAccuracy', Test.errorMatrix('Presence', 'classification').accuracy());
    print('Kappa Coefficient', Test.errorMatrix('Presence', 'classification').kappa());
    print('Producers Accuracy', Test.errorMatrix('Presence', 'classification').producersAccuracy());
    print('Users Accuracy', Test.errorMatrix('Presence', 'classification').consumersAccuracy());
```

Do you have any feedback for this practical? Please complete this quick (2-5 min) survey [here](https://forms.gle/hT11ReQpvG2oLDxF7).