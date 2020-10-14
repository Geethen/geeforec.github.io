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

1. Access long-term rainfall data

2. Summarise temporal data by region

3. Generate interactive map

4. Generate time-series plots

5. Output data (csv, rasterStack) for analysis outside of GEE

6. Explore understanding of rainfall as a driver of ecosystem processes (e.g. comparison with EVI, similar to NDVI from previous practical #3 with JW)
---
**Data import**

The datasets that will be required in this practical are largely already available on Google Earth Engine. In addition to these datasets, we will practice how to import a a local dataset into GEE.
```js

var costaRica = ee.FeatureCollection('USDOS/LSIB/2017');
var braulio = ee.FeatureCollection('WCMC/WDPA/current/polygons');
var rainAll = ee.ImageCollection("UCSB-CHG/CHIRPS/PENTAD");
var eviAll = ee.ImageCollection("MODIS/006/MOD13Q1");
```

The first four datasets imported correspond to those already available within GEE and are the Country boundaries, Protected area boundaries, long-term rainfall data, and the long-term EVI data from Modis respectively. Below we describe how to import a local dataset.

![](/images/prac4_f1.png)
---

**Filtering data**

We first define variables for the temporal window of interest. we will use these variables for the filtering of the long-term data.
```js
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
```
---
**Processing**

We first calculate the sum of rainfall on an annual basis within Costa Rica
```js
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
```
Calculate long-term annual mean rainfall, clipped to Costa Rica. Calculate annual mean Rainfall vs. EVI for Braulio Carrillo National Park. 
```js
var rainMean = rainMeanMY.mean().clip(costaRica);
var annualRainEVI = ee.ImageCollection.fromImages(years.map(function(y){
var evi_year = eviAll.filter(ee.Filter.calendarRange(y, y, 'year'))
.max().multiply(0.0001).rename('evi');
var img = rainAll.filter(ee.Filter.calendarRange(y, y, 'year')).max().rename('rain');
var time = ee.Image(ee.Date.fromYMD(y,1,1).millis()).divide(1e18).toFloat();
return img.addBands(\[evi_year, time\]).set('year', y).set('month', 1)
.set('date', ee.Date.fromYMD(y,1,1))
.set('system:time_start', ee.Date.fromYMD(y,1,1));
}).flatten());
```
**Charting**
Create time-series line chart of annual rainfall sum for Braulio Carrillo 2000 to 2018
First create chart parameters (e.g. Title, axis labels). Now create the line chart using the processed summaries
```js
var opt_chart_annualPrecip = {
  title: 'Mean Annual Rainfall: Braulio Carrillo',
  hAxis: {title: 'Year'},
  vAxis: {title: 'Rainfall (mm)'},};

var chart_annualPrecip = ui.Chart.image.seriesByRegion({
  imageCollection: annualPrecip, 
  regions: myBraulio_geo,
  reducer: ee.Reducer.mean(),
  scale: 5000,
  xProperty: 'date',
  seriesProperty: 'band'
}).setOptions(opt_chart_annualPrecip)
  .setChartType('LineChart');//'ColumnChart
print(chart_annualPrecip);
```

Create comparative line chart of rainfall and EVI summaries for Braulio Carrillo 
First create chart parameters (e.g. Title, axis labels). Now create the line chart with 2 y-axises using the processed summaries
```js
var opt_annualRainEVI = {title: "Annual Max Rainfall vs. 'Greenness (EVI): Braulio Carrillo", pointSize: 3,
    legend: {maxLines: 5, position: 'top'},
    series: { 0: {targetAxisIndex: 0},
              1: {targetAxisIndex: 1}},
        vAxes: {Adds titles to each axis.
          0: {title: 'Max EVI (*0.0001)'},
          1: {title: 'Max Rainfall (mm)'}},};

var rain_ndvi_chart = ui.Chart.image.series({
  imageCollection: annualRainEVI.select(['evi', 'rain']),
  region:myBraulio_geo,
  reducer: ee.Reducer.mean(),
  scale: 5000
}).setOptions(opt_annualRainEVI);
print(rain_ndvi_chart);
```
---
**VISUALISATION**

Setup map elements to visualise results. Create a map title. Legend parameters (start with this but change later using the visual parameters pop-up).

```js
var title = ui.Label('Costa Rica: Mean Annual Rainfall 2000 to 2018', {
  stretch: 'horizontal',
  textAlign: 'center',
  fontWeight: 'bold',
  fontSize: '20px'});

var rainViz = {
  min: 175, max: 317, 
  palette: 'ffffff, 67d9f1, 1036cb'};
```
Center the map to Costa Rica and zoomed to preliminary scale. All other layers will align to this parent map.Add long-term mean rainfall. Add Braulio Carrillo boundary last, to overlay.Enable zooming on the top-left map.
```js
Map.centerObject(costaRica, 8);
Map.addLayer(rainMean, rainViz, 'Mean Annual Rainfall'); 
Map.addLayer(braulio,{color: 'grey'}, 'Braulio Carrillo',true, 0.8);  
Map.setControlVisibility({zoomControl: true});
```
Add title to the map
Map.add(title);
To save this map online as a GEE app, follow steps below
Click the 'Apps' button above
Select 'NEW APP'
Give the App a Name
Leave everything else default
Click 'PUBLISH'
URL will appear - Click this to see your first online interactive map
* If you see a 'Not ready' page, give it a few minutes and try again


Prepare data (outcome) for export
From charts you can just maximise chart and click to export to csv, svg or png formats

1	Console panel > Max chart view	2	Export table or figure
	 		 

OR script the export using a reducer to get the mean rainfall value for Braulio Carrillo for each year
var csv_annualPrecip = annualPrecip.map(function(image){
  var year = image.get('year');
  var month = image.get('month');
  var mean = image.reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: myBraulio_geo,
    scale: 5000
  });  
  return ee.Feature(null, {'mean': mean.get('rain'),
                          'year': year})
});

Export the new feature as a .csv table with date and mean rainfall value
Export.table.toDrive({
  collection: csv_annualPrecip,
  description: 'annualRain',
  folder: 'testOTS',
  fileFormat: 'CSV'
});

OR save the results as a rasterStack with multiple layers representing annual sums of annual rainfall for Braulio Carrillo
First create a list of band names for your rasterStack output
var band_names = annualPrecip.map(function(image) {
      return ee.Feature(null, {'date': ee.Date(image.get('date'))
      .format('YYYY_MM')})})
      .aggregate_array('date')
print('Band Names', band_names);

Apply the function toBands() to the image collection to stack all bands into one image, naming the layers appropriately
var stack_RainEVI = annualPrecip.toBands().rename(band_names);

Export a cloud-optimized GeoTIFF.
i.e. rasterStack with 19 layers, representing annual rain from 2000 to 2018
Export.image.toDrive({
  image: stack_RainEVI,
  folder: 'testOTS',
  description: 'Rainfall_Sums',
  scale: 5000,
  region: myBraulio_geo,
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});
