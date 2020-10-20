---
title: Practical 7
date: 2020-10-27T22:00:00+00:00
hero: "/images/prac_7_f1.png"
excerpt: Binary change detection, computing area and charting
timeToRead: 8
authors: []

---
**Practical 7: Binary change detection, computing area and charting**

Access the completed practical script [here](https://code.earthengine.google.com/fae000aed0572d8d6a6b9f0ca8192517)

**Learning Objectives**

By the end of this practical you should be able to:

1\. Understand and implement a binary change detection across a country of interest between two time periods.

2\. Evaluate the area of the identified changes across the country of interest.

3\. Plot the computed area coverage using a pie chart.

**The end product**

![](/images/prac_7_f1.png)

**Figure 1:** The spatial distribution of forest change between 2000-2005 within Costa Rica with forest cover gain (green), loss (red) and forest cover that remained unchanged (no change, grey). To display the actual area coverage (sqkm), refer to the pie chart within the GEE console.

**Importing and pre-processing**

There are three datasets that you require for this practical. First you will add the Global Forest Cover Change (GFCC) dataset and rename this ‘treeCover’. Next, you will import your first two vector datasets. The first is a feature collection that stores country boundaries and the second is a collection of global protected area boundaries. You can find the first country boundary dataset by searching ‘Boundary’ and selecting the LSIB detailed layer version. Rename this layer ‘countries’. You can find the second table by searching World database of Protected Areas (WDPA) polygons and renaming this layer as ‘PAs’ in your imports section.

To easily select Costa Rica from its feature collection using the added marker, you will use filterBounds. This is the same function you previously used to filter an image collection.

```js
var country = countries.filterBounds(geometry).union();
var treeCov2000 = treeCover
.filterBounds(country)
.filter(ee.Filter.eq('year',2000))
.mosaic()
.select('tree_canopy_cover')
.gt(30)
.clip(country); 
```

A threshold of 30% tree canopy cover was selected since this threshold was used in the seminal paper that introduced the GFCC dataset i.e. this is a suggested threshold that separates Forest from non-Forest cover (Kim et al., 2014). These same functions can be applied to the 2005 tree cover layer. To be able to carry out a change detection for a binary layer (forest and non-forest), we will need to limit the change detection analysis to the forest areas from the two epochs. We are not concerned with non-forest areas that remained as non-forest. Rather we are interested in detecting forest losses, forest gains and forest cover that remained forest cover from the year 2000 to the year 2005. From a coding perspective, this requires us to create a mask that combines the forest cover from both epochs. This will allow our analysis to be limited to forest areas. Thereafter, to determine forest change we can use a simple subtraction.

```js
var mask = treeCov2000.firstNonZero(treeCov2005).selfMask()
var coverChange = treeCov2005.subtract(treeCov2000).updateMask(mask);
```

**Visualisation**

```js
Map.addLayer(treeCov2000,{},'Forest Cover 2000',false);
Map.addLayer(treeCov2005,{},'Forest Cover 2005',false);
var palette =['red','grey','green'];
Map.addLayer(coverChange,{palette:palette},'Forest Cover Change 2000-2005',false);
```

Note, we use the false argument to prevent the layers from loading (the default action) when the script is run. This may be useful when you do not want to view all the layers at once. Also, a palette can be specified by directly providing a colour name, as opposed to hex colour codes.

**Area calculation and plotting**

There are multiple ways to go about calculating the area of each of the three classes. We will look at three of these ways. This will also help you to become familiar with the common trade-offs between computational requirements the amount and the complexity of code you need to write.

**Option 1**

The first option is a more brute force approach whereby we create a separate layer for each of the classes (loss, gain and no forest change ) and calculate their corresponding area. We can use the code snippet below for all three of the layers by simply manipulating the first argument of the first function i.e., each number (-1 ,0 ,1) correspond to a single class in the coverChange raster. More specifically, loss, gain and no forest change respectively.

```js
var noChange = coverChange.eq(0).selfMask();
var areaImage = noChange.multiply(ee.Image.pixelArea())
var area = areaImage.reduceRegion({
reducer: ee.Reducer.sum(),
geometry: country.geometry(),
scale: 30,
maxPixels: 1e10
})
var noChangeAreaSqKm = ee.Number(
area.get('tree_canopy_cover')).divide(1e6).round()
print(noChangeAreaSqKm)
```

**Option 2**

The second option to calculate area for each of the three classes requires \~ a third of the amount of code used in the first option by using the group function.

```js
var areaImage = ee.Image.pixelArea().addBands(coverChange);
var areas = areaImage.reduceRegion({
reducer: ee.Reducer.sum().group({
groupField: 1,
groupName: 'class',
}),
geometry: country.geometry(),
scale: 30,
maxPixels: 1e10
});
var classAreas = ee.List(areas.get('groups'));
var classAreaLists = classAreas.map(function(item) {
var areaDict = ee.Dictionary(item);
var classNumber = ee.Number(areaDict.get('class')).format();
var area = ee.Number(
areaDict.get('sum')).divide(1e6).round();
return ee.List(\[classNumber, area\]);
});
var result = ee.Dictionary(classAreaLists.flatten());
print(result);
```

**Option 3: Calculating and plotting the areas of each class**

Unlike the previous two options, this option computes the areas for each class and presents the results in a pie chart.

```js
var labels =[ 'Gain','no change','loss' ];
var areaClass = coverChange.eq([1, 0, -1]).rename(labels);
var palette = ['green','red','grey'];
var areaEstimate = areaClass.multiply(ee.Image.pixelArea()).divide(1e6);
var chart = ui.Chart.image
.regions({
image: areaEstimate,
regions: country,
reducer: ee.Reducer.sum(),
scale: 100,
})
.setChartType('PieChart').setOptions({
width: 250,
height: 350,
title: 'Area by class (sqkm)',
is3D: true,
colors: palette,
});
print(chart);
```

The ‘best’ option to use is often case-specific and is dependent on your goal, it is likely that you prefer to create plots within R owing to the powerful and flexible visualisation libraries such as ggplot2. In that case, the first option is more ideal since I found it to be the most computationally efficient of the three options. More specifically, I used a lower scale value of 100 m for the last two options whilst I used the native 30 m scale value for the first option. Note: different scale values result in different final area estimates owing to the influences of spatial resolution. More specifically, when using the 100 m scale value, there was an inflated area value reported for the majority no-change class, and an underestimation in the area values for the two minority classes. This may largely be attributed to the different area to perimeter ratios associated with the different scale values.

**Practical 4 Exercise**

* Try to understand the influence of varying scale values on the reported area values for each of the classes and their potential influence on conservation efforts. We will discuss your thoughts in our first discussion session.
* Using the protected area table that you imported at the beginning of this practical, investigate the afforestation/deforestation patterns within some of these protected areas.

**References**

Kim, D.H., Sexton, J.O., Noojipady, P., Huang, C., Anand, A., Channan, S., Feng, M. and Townshend, J.R., 2014. Global, Landsat-based forest-cover change from 1990 to 2000. _Remote Sensing of Environment_, _155_, pp.178-193.

Do you have any feedback for this practical? Please complete this quick (2-5 min) survey [here](https://forms.gle/hT11ReQpvG2oLDxF7).