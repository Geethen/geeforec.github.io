+++
authors = []
date = 2020-10-04T22:00:00Z
draft = true
excerpt = "Long-term patterns of rainfall in and around Braulio Carrillo National Park, Costa Rica"
hero = "/images/prac4_f4.png"
timeToRead = 15
title = "Practical 4"

+++
**Practical 4: Long-term patterns of rainfall in and around Braulio Carrillo National Park in Costa Rica**

Access the completed practical script [here](https://code.earthengine.google.com/?scriptPath=users%2FBioGIS%2FbioGEE%3APractical_4%2Frainfall_costa_rica)

**Learning Objectives**

By the end of this practical you should be able to:

1. Access long-term rainfall data
2. Summarise temporal data by region
3. Generate interactive map
4. Generate time-series plots
5. Output data (csv, rasterStack) for analysis outside of GEE
6. Explore understanding of rainfall as a driver of ecosystem processes (e.g. comparison with EVI, similar to NDVI from previous practical #3 with JW)

***

Rainfall blah blah blah

**Data import**

The datasets that we will use for this practical are largely already available on Google Earth Engine. In addition to these datasets, we will practice how to import a local dataset into GEE.

```js
var costaRica = ee.FeatureCollection('USDOS/LSIB/2017');
var braulio = ee.FeatureCollection('WCMC/WDPA/current/polygons');
var rainAll = ee.ImageCollection("UCSB-CHG/CHIRPS/PENTAD");
var eviAll = ee.ImageCollection("MODIS/006/MOD13Q1");
```

The first four datasets imported correspond to those already available within GEE and are the country boundaries, protected area boundaries, long-term rainfall data, and the long-term MODIS EVI data respectively. Below we describe how to import a dataset available locally into GEE. You can download and save the required Braulio Carillo boundary on your local hard drive from [here](https://drive.google.com/file/d/1omD5vPk4LMQSnC2BHJCg6GlnmpzBsFQG/view?usp=sharing).

## ![](/images/prac4_f1.png)
**Figure 1:** The spatial distribution of forest change between 2000-2005 within Costa Rica with forest cover gain (green), loss (red) and forest cover that remained unchanged (no change, grey). To display the actual area coverage (sqkm), refer to the pie chart within the GEE console.

***

**Filtering data**

We first define variables for the temporal window of interest. We will use these variables for filtering the long-term data.

```js
var startYear = 2000;
var endYear = 2018;
var startDate = ee.Date.fromYMD(startYear,1, 1);
var endDate = ee.Date.fromYMD(endYear + 1, 12, 31);
var years = ee.List.sequence(startYear, endYear);
var months = ee.List.sequence(1, 12);

var costaRica = ee.FeatureCollection('USDOS/LSIB/2017')
.filter(ee.Filter.inList('COUNTRY_NA', ['Costa Rica']));
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

***

**Processing**

We first calculate the sum of rainfall on an annual basis within Costa Rica.

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
return img.addBands([evi_year, time]).set('year', y).set('month', 1)
.set('date', ee.Date.fromYMD(y,1,1))
.set('system:time_start', ee.Date.fromYMD(y,1,1));
}).flatten());
```

***

**Charting**

We will now chart the annual sum of rainfall for Braulio Carrillo 2000 to 2018. Here, we first define chart parameters (e.g. title and axis labels). Thereafter, we create the line chart that incorporates these pre-defined chart parameters.

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
  .setChartType('LineChart');
print(chart_annualPrecip);
```

It is also possible to plot both rainfall and EVI within a single chart. This may be valuable in understanding the relationship between these two variables. To create a comparative line chart of rainfall and EVI summaries for Braulio Carrillo. As in the previous chart, we first define chart parameters and then create the line chart with two y-axes using the processed summaries

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

***

**Visualisation**

In addition to creating charts, you may want to create a sharable image visualisation that is viewable on any electronic device. We can create GEE application) to achieve this. Here the first step is to define the various map elements to visualise results. This includes; defining a map title and legend parameters.

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

Next, we center the map to Costa Rica. All other layers will align with this parent map and add the data of interest. Specifically, we add the long-term mean rainfall raster and the Braulio Carrillo boundary. To conclude, we will add a zoom control button and the previously defined title.

```js
Map.centerObject(costaRica, 8);
Map.addLayer(rainMean, rainViz, 'Mean Annual Rainfall'); 
Map.addLayer(braulio,{color: 'grey'}, 'Braulio Carrillo',true, 0.8);  
Map.setControlVisibility({zoomControl: true});
Map.add(title);
```

To save this map online as a GEE app, follow the steps below:

1. Click the 'Apps' button above Select 'NEW APP'
2. Give the App a Name
3. Leave everything else default
4. Click 'PUBLISH' URL will appear - Click this to see your first online interactive map
5. If you see a 'Not ready' page, give it a few minutes and try again

![](/images/prac4_f2.png)
**Figure 1:** The spatial distribution of forest change between 2000-2005 within Costa Rica with forest cover gain (green), loss (red) and forest cover that remained unchanged (no change, grey). To display the actual area coverage (sqkm), refer to the pie chart within the GEE console.

***

**Data Export**

To export the data shown in the created charts, similar to practical 3, you may simply maximise the chart and then click to export to the available formats (csv, svg or png). Alternatively, you may script the export. this option benefits from having more options to customise the data export. For example, including numerous variables and, potentially a well- formatted date: time variable. In this practical, this is achieved by first using a reducer to get the mean rainfall value for Braulio Carrillo for each year and adding a date variable. The exported csv table will contain a column for both date and mean annual rainfall. you will find this csv file in your google drive.

![](/images/prac4_f3.png)
**Figure 1:** The spatial distribution of forest change between 2000-2005 within Costa Rica with forest cover gain (green), loss (red) and forest cover that remained unchanged (no change, grey). To display the actual area coverage (sqkm), refer to the pie chart within the GEE console.

```js
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

Export.table.toDrive({
collection: csv_annualPrecip,
description: 'annualRain',
folder: 'testOTS',
fileFormat: 'CSV'
});
```

In addition, to the export options presented above and in practical 3. You may also export the results as a rasterStack with multiple layers representing the sum of annual rainfall for Braulio Carrillo. We will first create a list of band names for the rasterStack output and apply the function toBands() to the image collection to stack all bands into a single image. Each band will contain a unique name corresponding to, in this example, the year of the annual sum.

```js
var band_names = annualPrecip.map(function(image) {
return ee.Feature(null, {'date': ee.Date(image.get('date'))
.format('YYYY_MM')})})
.aggregate_array('date')
print('Band Names', band_names);

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
```

As a last step, save the script.

**Practical 3 Exercise**

Repeat this practical but use the Landsat-8 dataset and provide a new area of interest. Play around with extending the filterDate duration, the size of your area of interest, and the scale used ui.Chart. Take note that the ImageCollection size may produce memory errors.

To share your script, click on Get Link and then copy script path. Send your completed script to **email**

***

**References**

Funk C, Peterson P, Landsfeld M, Pedreros D, Verdin J, Shukla S, Husak G, Rowland J, Harrison L, Hoell A, Michaelsen J (2015) The climate hazards infrared precipitation with stations—a new environmental record for monitoring extremes. Scientific Data 2, 150066. doi:10.1038/sdata.2015.66 2015