+++
authors = []
date = 2020-10-21T22:00:00Z
draft = true
excerpt = "Long-term patterns of rainfall in and around Braulio Carrillo National Park, Costa Rica"
hero = ""
timeToRead = 0
title = "Practical 4"

+++
**Practical 4: Long-term patterns of rainfall in and around Braulio Carrillo National Park, Costa Rica**

Access the completed practical script [here](https://code.earthengine.google.com/63bf79381841c0d81c3afaea76d08040)

**Learning Objectives**

By the end of this practical you should be able to:

1\. Access long-term rainfall data

2\. Summarise temporal data by region

3\. Generate interactive map

4\. Generate time-series plots

5\. Output data (csv, rasterStack) for analysis outside of GEE

6\. Explore understanding of rainfall as a driver of ecosystem processes (e.g. comparison with EVI, similar to NDVI from previous practical #3 with JW)

***

**Data import**

The datasets that will be required in this practical are largely already available on Google Earth Engine. In addition to these datasets, we will practice how to import a a local dataset into GEE.

var costaRica = ee.FeatureCollection('USDOS/LSIB/2017');

var braulio = ee.FeatureCollection('WCMC/WDPA/current/polygons');

var rainAll = ee.ImageCollection("UCSB-CHG/CHIRPS/PENTAD");

var eviAll = ee.ImageCollection("MODIS/006/MOD13Q1");

The first four datasets imported correspond to those already available within GEE and are the Country boundaries, Protected area boundaries, long-term rainfall data, and the long-term EVI data from Modis respectively. Below we describe how to import a local dataset.

![](/images/prac4_f1.png)

***

**Filtering data**

We first define variables for the temporal window of interest. we will use these variables for the filtering of the long-term data.

var startYear = 2000;

var endYear = 2018;

var startDate = ee.Date.fromYMD(startYear,1, 1);

var endDate = ee.Date.fromYMD(endYear + 1, 12, 31);

var years = ee.List.sequence(startYear, endYear);

var months = ee.List.sequence(1, 12);

var costaRica = ee.FeatureCollection('USDOS/LSIB/2017')

.filter(ee.Filter.inList('COUNTRY_NA', \['Costa Rica'\]));

var braulio = ee.FeatureCollection('WCMC/WDPA/current/polygons')

.filter(ee.Filter.stringContains('ORIG_NAME', 'Braulio Carrillo'));

var rainAll = ee.ImageCollection("UCSB-CHG/CHIRPS/PENTAD")

.select('precipitation')

.filterBounds(costaRica_geo);

var eviAll = ee.ImageCollection("MODIS/006/MOD13Q1")

.select('EVI')

.filterBounds(costaRica_geo);

var costaRica_geo = costaRica.geometry();

var braulio_geo = braulio.geometry();

var myBraulio_geo = myBraulio.geometry();

***

**Processing**

We first calculate the sum of rainfall on an annual basis within Costa Rica

var annualPrecip = ee.ImageCollection.fromImages(

years.map(function (year) {

var annual = rainAll

.filter(ee.Filter.calendarRange(year, year, 'year'))

.sum().rename('rain');

return annual

.clip(costaRica_geo)

.set('year', year).set('date', ee.Date.fromYMD(year, 1, 1))

.set('system:time_start', ee.Date.fromYMD(year, 1, 1));

}));

Calculate long-term annual mean rainfall, clipped to Costa Rica

var rainMean = rainMeanMY.mean().clip(costaRica);

Calculate annual mean Rainfall vs. EVI for Braulio Carrillo National Park

var annualRainEVI = ee.ImageCollection.fromImages(years.map(function(y){

var evi_year = eviAll.filter(ee.Filter.calendarRange(y, y, 'year'))

.max().multiply(0.0001).rename('evi');

var img = rainAll.filter(ee.Filter.calendarRange(y, y, 'year')).max().rename('rain');

var time = ee.Image(ee.Date.fromYMD(y,1,1).millis()).divide(1e18).toFloat();

return img.addBands(\[evi_year, time\]).set('year', y).set('month', 1)

.set('date', ee.Date.fromYMD(y,1,1))

.set('system:time_start', ee.Date.fromYMD(y,1,1));

}).flatten());