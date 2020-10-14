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