+++
authors = []
date = 2020-10-27T13:00:00Z
draft = true
excerpt = "Fire frequency in the Kruger National Park, South Africa"
hero = "/images/prac6_f4.png"
timeToRead = 15
title = "Practical 6"

+++
**Practical 6: Fire frequency in the Kruger National Park, South Africa**

Access the completed practical script [here](https://code.earthengine.google.com/?scriptPath=users%2Fudemy%2FOTS_GEE%3Atue27_prac6_v3)

**Learning Objectives**

By the end of this practical you should be able to:

1. Access terrain model data from SRTM
2. Access monthly burn scar data
3. Generate a hillshade using SRTM data
4. Explore long-term patterns of fire frequency
5. Build an animation and output for use in, for example, PowerPoint presentations

***

**Data import**

The datasets that we will use for this practical are largely already available on Google Earth Engine. In addition to these datasets, we will practice how to import a local dataset into GEE.

```js
var dem = ee.Image("CGIAR/SRTM90_V4");
var FireCCI = ee.ImageCollection('ESA/CCI/FireCCI/5_1');
```

The first two datasets imported correspond to those already available within GEE and are the Shuttle Radar Topography Mission (SRTM) digital elevation dataset and the MODIS Fire_cci Burned Area pixel product version 5.1 (FireCCI51). Below we describe how to import a dataset available locally into GEE. You can download and save the required boundary shapefile for the Kruger National Park (Kruger) from [here](https://drive.google.com/file/d/1omD5vPk4LMQSnC2BHJCg6GlnmpzBsFQG/view?usp=sharing).

## ![](/images/prac6_f1.png)

***

**Filtering data**

We first define variables for the temporal and spatial windows of interest. We will use these variables to filter our data before processing.

```js
var startDate = ee.Date.fromYMD(2001,1,1);
var endDate = ee.Date.fromYMD(2018,12,31);
var years = ee.List(fire.aggregate_array('year')).distinct().sort();

var knp_geo = knp.geometry();

var srtm = dem.clipToCollection(knp);

var fire = FireCCI
    .filterBounds(knp_geo)
    .filterDate(startDate, endDate)
    .map(function(img) { // This function adds year as property to each image
      return img.set('year', ee.Image(img).date().get('year'));
    });
```

***

**Processing**

We will build a function to remove all burn scars from the fire dataset that have a confidence interval of less than 50%.

```js
var confMask = function(img) {
  var conf = img.select('ConfidenceLevel');
  var level = conf.gt(50);
  return img.updateMask(level).select('BurnDate'); //return only 'BurnDate' band
};
```

Then we will run the function and summarise the burn scars by the day-of-year (doy) most frequently burnt, followed by the the frequency areas are burnt in Kruger from 2001 until 2018. 

```js
var fireDOY_list = years.map(function(year) {
  return fire
    .filterMetadata('year', 'equals', year) // Filter image collection by year
    .map(confMask) // Apply confidence mask >50%
    .reduce(ee.Reducer.mode()) // Reduce image collection by most common DOY
    .set('year', year) // Set composite year as an image property
    .set('system:time_start', ee.Date.fromYMD(year, 1, 1));
});
var doyFires = ee.ImageCollection.fromImages(fireDOY_list); // Convert the image List back to an ImageCollection

var fireCnt_list = years.map(function(year) {
  return fire
    .filterMetadata('year', 'equals', year) // Filter image collection by year
    .map(confMask) // Apply confidence mask >50%
    .reduce(ee.Reducer.countDistinct()) // Reduce image collection by count distinct doy
    .set('year', year) ));// Set composite year as an image property
    .set('system:time_start', ee.Date.fromYMD(year, 1, 1
});
var cntFiresDOY = ee.ImageCollection.fromImages(fireCnt_list); // Convert the image List back to an ImageCollection
```

