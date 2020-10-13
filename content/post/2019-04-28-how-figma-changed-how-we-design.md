---
title: Practical 9
date: 2019-04-28
hero: "/images/hero-5.jpg"
excerpt: 'Supervised learning 2: Land cover classification.'
timeToRead: 8
authors:
- Thiago Costa

---
**Practical 9, part 1: Supervised learning 2: Land cover classification.**

Access the completed practical, part 1 script [here](https://code.earthengine.google.com/63bf79381841c0d81c3afaea76d08040)

**Learning Objectives**

By the end of this practical you should be able to:

1. Describe the workflow for a typical supervised classification workflow.

2. Create reference data for classes of interest.

3. Fit a random forest model on spectral variables to map the distribution of water, built-up areas, tree-cover and other (neither of the three classes).

4. Evaluate model accuracy and improve model accuracy

  
**The end product**

![](/images/prac9_f1.png)

**Figure 1:** The distribution of water, built-up areas, tree-cover and other (neither of the three classes) pixels at a 30 m resolution for a heterogenous coastal area in the Western cape, South Africa based on Landsat-8 and Sentinel-1 radar imagery.

**Importing**

For this practical, you will be required to import Landsat-8 surface reflectance and Sentinel-1 Ground Range Detected imagery (GRD) data. Rename these as l8sr and s1 respectively. You have been provided with a reference points dataset (four feature collections within the imports section). However, you have the functionality within GEE to create your own reference points for each of the four classes of interest i.e. water, built-up areas, tree-cover and other.

**Creating custom reference points**

In the scenario that you are creating your own reference data points, you can use the following steps. Click on the add Markers button. Thereafter, hover over the geometry imports section and click on the cog wheel besides the geometry and setup the various options by changing the ‘Name’ to the class of interest, changing the ‘import as’ option to a feature collection and adding a property called label with a unique integer for each class. Once set, click ok and go to the map area and add reference points for the applicable class in appropriate locations. ‘Appropriate locations’ is dependent on the spatial resolution of your imagery to be classified and the time period of interest. For instance, when working with Landsat-8, 30 m imagery, it is ideal if you add markers for areas that are dominated by the class of interest. A higher resolution base map together with a RGB or FCI can be highly useful for guiding your reference point selection. Note, use a base map that overlaps with the period of concern.

![](/images/prac9_f2.png)

**Figure 2:** An example of setting up a feature collection for reference point collection of a class named water. Note, the labels for the land cover classes need to be zero indexed.

**Filtering**

Create a Landsat-8 surface reflectance composite that covers the AOI (shown by the provided geometry) for the period of 1 June 2016 to 30th September 2016, add ndvi and mndwi bands. Lastly, compute the median image, taking into consideration the GEE scaling factor and clipping to the extent of geometry.

```js
var filtered = l8sr
.filterBounds(geometry)
.filterDate(startDate, endDate)
.map(function(image){
var ndvi = image.normalizedDifference(['B5', 'B4']).rename(['ndvi']);
var mndwi = image.normalizedDifference(['B3', 'B7']).rename(['mndwi']);
return image.select('B.*').addBands([ndvi,mndwi]);
}).median()
.divide(10000)
.clip(geometry);
```

Similarly, filter the Sentinel-1 data to the AOI for the period of June to September 2016 as above. In addition, filter the s1 image collection to those images that have captured in Interferometric Wide (IW) mode and contain Vertical Transmit and Vertical Receive (VV) and Vertical Transmit and Horizontal Receive polarised bands. Lastly, for each image, select all bands that start with a V and reduce the resulting images to the mean image.

```js
var s1 = s1.filterBounds(geometry)

.filterDate(startDate, endDate)

.filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))

.filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))

.filter(ee.Filter.eq('instrumentMode', 'IW'))

.select("V.")

.mean();
```

Create a final composite image with the bands of both the Landsat-8 composite and Sentinel-1 composite.

```js
var composite = filtered.addBands(s1).aside(print).clip(geometry);
```

Preparing the train and test datasets

The next step is to split this data into a training and testing partition. The training partition is used to fit the model whilst the test data is used to evaluate the accuracy of the model. Here, we use a split of 80% train and 20% test. We set the seed to ensure we end up with roughly the same train and test partition in the case of re-running the model.

```js
var new_table = new_table.randomColumn({seed: 1});
var training = new_table.filter(ee.Filter.lt('random', 0.80)).aside(print);
var test = new_table.filter(ee.Filter.gte('random', 0.80)).aside(print);
```

Extracting the spectral signatures

At this point, we have our response and explanatory variables prepared and we can move on to extracting the spectral signatures for each of the points. The ‘tileScale’ argument may be useful when working with large computations that fail due to computation errors.

```js
var Features = composite.sampleRegions({
collection: training,
properties: ['label'],
scale: 30,
tileScale:16
});
```

**Model fitting**

```js
**var trainedClassifier =** ee.Classifier.randomForest(100)
.train({
features: Features,
classProperty: 'label',
inputProperties: composite.bandNames()
});
```

**Inference for entire AOI**

```js
var classified = composite.classify(trainedClassifier);
```

**Model evaluation**

The training accuracy largely overestimates model accuracy since this data has been used to train the model. We therefore use a test set (data unseen by the model) to get a more reliable estimate of model accuracy.

```js
var trainAccuracy = trainedClassifier.confusionMatrix().accuracy();
print('Train Accuracy', trainAccuracy);
Test accuracy
var test = composite.sampleRegions({
collection: test,
properties: \['label'\],
scale: 30,
tileScale: 16
});
var Test = test.classify(trainedClassifier);
print('ConfusionMatrix', Test.errorMatrix('label', 'classification'));
print('TestAccuracy', Test.errorMatrix('label', 'classification').accuracy());
print('Kappa Coefficient', Test.errorMatrix('label', 'classification').kappa());
print('Producers Accuracy', Test.errorMatrix('label', 'classification').producersAccuracy());
print('Users Accuracy', Test.errorMatrix('label', 'classification').consumersAccuracy());
```

Visualisation
Visualising the RGB image, reference points and classified image

```js
Map.centerObject(points.geometry().bounds(), 13);
Map.addLayer(composite,rgb_vp, 'Landsat-8 RGB');
```

```js
showTrainingData();
function showTrainingData(){
var colours = ee.List(["darkblue","darkgreen","yellow", "orange"]);
var lc_type = ee.List(["water", "tree cover","built-up","other"]);
var lc_label = ee.List([0, 1, 2, 3]);
var lc_points = ee.FeatureCollection(
lc_type.map(function(lc){
var colour = colours.get(lc_type.indexOf(lc));
return points.filterMetadata("label", "equals", lc_type.indexOf(lc).add(1))
.map(function(point){
return point.set('style', {color: colour, pointShape: "diamond", pointSize: 3, width: 2, fillColor: "00000000"});
});
})).flatten();

Map.addLayer(classified, {min: 0, max: 3, palette: ["darkblue","darkgreen","yellow", "orange"]}, 'Classified image', false);
Map.addLayer(lc_points.style({styleProperty: "style"}), {}, 'TrainingPoints', false);
}
```

**Practical 9, part 2: Supervised learning 2: Improving land cover classification.**

Access the completed practical, part 2 script [here](https://code.earthengine.google.com/017a363f7e3766b60ba17bc0a3ebc62c)

**Learning Objectives**

By the end of this practical you should be able to:

1. Understand the role and importance of high-quality training data.
2. Use an objective approach (experimental) to improve the selection of training data.
3. Determine the area of applicability for a model.

**The end product**

![](/images/prac9_f3.png)

**Figure 1:** The area of applicability for a model that discriminated water and non-water pixels at a 10 m resolution for a heterogenous area near Hartbeespoort Dam, South Africa based on Sentinel-2 imagery.

Reference

The area of applicability concept is based is an experimental concept still under review and has been implemented based on the pre-print version available [here](https://arxiv.org/abs/2005.07939).

Meyer, H. and Pebesma, E., 2020. Predicting into unknown space? Estimating the area of applicability of spatial prediction models. arXiv preprint arXiv:2005.07939.