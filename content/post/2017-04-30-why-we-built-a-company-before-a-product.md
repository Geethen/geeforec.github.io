---
title: Practical 2
date: 2020-10-19T22:00:00+00:00
hero: "/images/prac_2_f1.png"
excerpt: Spectral indices, atmospheric interference and water detection
timeToRead: 3
authors: []

---
**Practical 2: Spectral indices, atmospheric interference and water detection**

Access the completed practical script [here](https://code.earthengine.google.com/69f9fe758e00f7caba12f4f88352b49e)

---
**Learning Objectives**
By the end of this practical you should be able to:

1. Compute the Normalised Difference Vegetation Index (NDVI) and Normalised Difference Water Index (NDWI).
2. Interpret both the NDVI and NDWI layer.
3. Use these two spectral indices to detect water.
4. Understand the influence of atmospheric interference on reflectance and detecting water.
---
**Importing and Filtering**
Import the Sentinel-2, level 1C data and rename it s21c. Thereafter, import the level 2A product and rename it s22a. Lastly, add a marker on Theewaterskloof dam. Building from the previous practical where you imported and filtered Sentinel-2 data, we will repeat these steps.
```js
var filtered = s21c.filterBounds(Theewaterskloof)
.filterDate('2019-05-01','2019-08-14')
.median();
```
Here we compute a median image as opposed to selecting the first image
(in the first practical). A median image is preferable when you reduce
an entire image collection since a cloud-free and shadow-free image will
be returned. Clouds have high reflectance values while shadows have low
reflectance values. Like other atmospheric interferences, clouds and
their associated shadows are largely undesirable and should be removed
prior to any analysis.

At this point you have converted an image collection to a single median
image. Next, we will compute the NDVI and NDWI spectral indices. GEE has
a dedicated function for this.
```js
var NDVI = filtered.normalizedDifference(['B8','B4']);
var NDWI = filtered.normalizedDifference(['B3','B8']);
```
---
**_Visualisation_**

We first specify a palette to be used when visualising the indices. We
can create a colour palette using strings (e.g., ‘green’, ‘red’) or we
can use hex colour codes. The NDVI and NDWI spectral indices both range
from -1 to 1. However, for NDVI we specify a minimum of zero to improve
the visualisation of NDVI. This may not always be necessary.
```js
var vis = ['FFFFFF', 'CE7E45', 'DF923D', 'F1B555', 'FCD163','99B718',
'74A901', '66A000', '529400', '3E8601', '207401', '056201',
'004C00', '023B01', '012E01', '011D01', '011301'];

Map.addLayer(NDVI,{min: 0, max: 1, palette: vis},'NDVI');
Map.addLayer(NDWI,{min: -1, max: 1, palette: vis},'NDWI');
```
---
**_Interpreting spectral indices_**

![](/images/prac2_f2.png)

## **Figure 2:** Spectral indices take advantage of the spectral properties of land cover. For instance, as highlighted in the theory lecture, vegetation has a high reflectance in the Near-Infrared (NIR) region while having a low reflectance in the red portion of the electromagnetic (EM) spectrum. As a result of using these bands to compute the NDVI, the index corresponds to the greenness of vegetation and has been shown to be correlated to various vegetation parameters such as vegetation health, nutrient levels, and plant phenophase. Similarly, NDWI is mainly sensitive to water.
---
**Detecting water**
There are numerous methods available to detect surface water and this area of research is still an active one. In this course, you will be introduced to two of these general approaches i.e. supervised classification and thresholding. For this practical, you will use a very simple thresholding-based approach that employs the computed NDVI and NDWI spectral indices.
```js
    Map.addLayer(NDWI.gt(NDVI),{},'water_1c');
```
## In the above code snippet, the function gt() returns a 1 if the first value is greater than the second, creating a binary raster i.e. a value of 1 is returned when NDWI values are greater than those of the corresponding NDVI pixel. This inequality can largely be useful to detect open surface water. However, upon inspection, you will come across inevitable omission and commission errors.
---
**_The influence of atmospheric effects on water detection._**
Recently, there has been a drive towards Analysis Ready Data (ARD) i.e. in part, this includes data that has already been corrected for atmospheric interferences, however, there is a considerable amount of uncertainty and variance associated with the results of different atmospheric correction algorithms. This, together with the fact that atmospherically corrected Sentinel-2 data is not available for areas outside of Europe from 2015 (the start of the archive) to 2018 i.e. level 1C (atmospherically uncorrected) data is only available. It is therefore important to understand the effects of atmospheric interference on water detection.

![](/images/prac2_f3.png)

**Figure 3:** The influence of atmospheric interference on the detection
of water. Water detected from atmospherically corrected Sentinel-2,
level 2A (top), reference water detection based on the long-term surface
water from Landsat-8 made available through the Global Surface Water
product (centre), water detected from atmospherically uncorrected
Sentinel-2, level 1C data (bottom).
---
**_Practical 2 Exercise_**

Repeat the steps in this practical for the level 2A data that you
imported and renamed as s22a at the beginning of this practical.
Thereafter, compare the water detection results and patterns with the
s21c image results. Submit your final script.