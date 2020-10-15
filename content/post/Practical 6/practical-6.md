+++
authors = []
date = 2020-10-27T13:00:00Z
draft = true
excerpt = "Fire frequency in the Kruger National Park, South Africa"
hero = "/images/fire3.png"
timeToRead = 15
title = "Practical 6"

+++
**Practical 6: Fire frequency in the Kruger National Park, South Africa**

Access the completed practical script [here](https://code.earthengine.google.com/?scriptPath=users%2Fudemy%2FOTS_GEE%3Atue27_prac6_v3)

**Learning Objectives**

By the end of this practical you should be able to:

1.	Access terrain model data from SRTM
2.	Access monthly burn scar data
3.	Generate a hillshade using SRTM data
4.	Explore long-term patterns of fire frequency
5.	Build an animation and output for use in, for example, PowerPoint presentations

***

**Data import**

The datasets that we will use for this practical are largely already available on Google Earth Engine. In addition to these datasets, we will practice how to import a local dataset into GEE.

```js
var costaRica = ee.FeatureCollection('USDOS/LSIB/2017');
var braulio = ee.FeatureCollection('WCMC/WDPA/current/polygons');
var rainAll = ee.ImageCollection("UCSB-CHG/CHIRPS/PENTAD");
var eviAll = ee.ImageCollection("MODIS/006/MOD13Q1");
```

The first four datasets imported correspond to those already available within GEE and are the country boundaries, protected area boundaries, long-term rainfall data, and the long-term MODIS EVI data respectively. Below we describe how to import a dataset available locally into GEE. You can download and save the required Braulio Carillo boundary on your local hard drive from [here](https://drive.google.com/file/d/1omD5vPk4LMQSnC2BHJCg6GlnmpzBsFQG/view?usp=sharing).