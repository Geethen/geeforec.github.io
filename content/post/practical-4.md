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
6. Explore understanding of rainfall as a driver of ecosystem processes (e.g. comparison with vegetation dynamics e.g. previous practical #3)

***

**Introduction**

Rainfall plays a central role in a myriad of natural processes, including river health, the transportation of nutrients, soil moisture, vegetation dynamics, fire regimes, animal movement and distribution patterns and landscape heterogeneity. Within protected areas these processes function together to safeguard ecosystem integrity. In the face of current climate change predictions, the spatio-temporal patterns of rainfall is an increasingly important component to include in any ecological study (MacFadyen et al 2018). Here we explore patterns of monthly rainfall across Costa Rica and the Braulio Carrillo National Park from 2000 to 2019 (20 years). We'll summarise monthly and annual rainfall patterns using line charts and examine the long-term spatial patterns of rainfall using an interactive map. Time permitting, we'll take a look at how the temporal patterns of annual rainfall compares to patterns of vegetation vigour or 'greeness', highlighting it's importance as a bottom-up ecosystem driver.

***

**Data import**

The datasets we will use for this practical are all available on Google Earth Engine and can be accessed as follows:

```js
var costaRica = ee.FeatureCollection('USDOS/LSIB/2017');
var braulio = ee.FeatureCollection('WCMC/WDPA/current/polygons');
var rainAll = ee.ImageCollection("UCSB-CHG/CHIRPS/PENTAD");
var eviAll = ee.ImageCollection("MODIS/006/MOD13Q1");
```

The first dataset, [LSIB 2017](https://developers.google.com/earth-engine/datasets/catalog/WCMC_WDPA_current_polygons), contains polygon representations of all international boundaries. The second, [WDPA](https://developers.google.com/earth-engine/datasets/catalog/WCMC_WDPA_current_polygons), contains polygons of all the world's protected areas. The third, [CHIRPS](https://developers.google.com/earth-engine/datasets/catalog/UCSB-CHG_CHIRPS_PENTAD), is a gridded rainfall time series dataset and the last, [MOD13Q1](https://developers.google.com/earth-engine/datasets/catalog/MODIS_006_MOD13Q1), provides vegetation indexes (NDVI and EVI) depiciting vegetation 'greeness' per 250m pixel.

***

**Filtering data**

First define variables for the temporal window of interest, including a start-date, end-date and the range of years and months. We will use these variables later to filter and summarise the long-term data.

```js
var startDate = ee.Date.fromYMD(2000,1,1);
var endDate = ee.Date.fromYMD(2019,12,31);
var years = ee.List.sequence(2000, 2019);
var months = ee.List.sequence(1, 12);
```

Then filter our polygon features to our areas of interest (AOI), namely Costa Rica and Braulio Carrillo protected area. At the same time, convert these two FeatureCollections to geometry objects as we'll need them later as function parameters for functions we'll be building.

```js
var costaRica = ee.FeatureCollection('USDOS/LSIB/2017')
.filter(ee.Filter.inList('COUNTRY_NA', ['Costa Rica']));
var costaRica_geo = costaRica.geometry();

var braulio = ee.FeatureCollection('WCMC/WDPA/current/polygons')
.filter(ee.Filter.stringContains('ORIG_NAME', 'Braulio Carrillo'));
var braulio_geo = braulio.geometry();
```

Now filter the CHIRPS ImageCollection for rainfall (i.e. 'precipitation') and the MODIS MOD13Q1 product for the Enhanced Vegetation Index (EVI) instead of the Normalized Difference Vegetation Index (NDVI) used in the previous practical. At the same time, filter by date range and our AOI to speed up proceeding analyses.

```js
var rainAll = ee.ImageCollection("UCSB-CHG/CHIRPS/PENTAD")
.select('precipitation')
.filterDate(startDate, endDate)
.filterBounds(costaRica_geo);

var eviAll = ee.ImageCollection("MODIS/006/MOD13Q1")
.select('EVI')
.filterDate(startDate, endDate)
.filterBounds(costaRica_geo);
```

***

**Processing**

Now, to calculate the annual monthly sum of rainfall across Braulio Carrillo National Park from 2000 to 2019, we need to reduce the monthly rainfall record by their sum per year as follows:

```js
var rainYr_list = years.map(function(y) {
return rainAll
.filter(ee.Filter.calendarRange(y,y,'year')) 
.reduce(ee.Reducer.sum())
.rename('rain_mean') 
.set('year', ee.Date.fromYMD(y, 1, 1)) 
.set('name','rainfall');
});

// Convert the image List to an ImageCollection.
var rainYr = ee.ImageCollection.fromImages(rainYr_list);
print('Check rainYr', rainYr);
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

***

**Bonus Section**

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

**Practical 4 Exercise**

Repeat this practical but use NDVI instead of EVI and Germany instead of Costa Rica. You can also play around with different dates, keeping in mind the different date limits for each ImageCollection.
To share your script, click on Get Link and then copy script path. Send your completed script to [**email**](mailto:sandra@biogis.co.za)

***

**References**

Funk C, Peterson P, Landsfeld M, Pedreros D, Verdin J, Shukla S, Husak G, Rowland J, Harrison L, Hoell A, Michaelsen J (2015) The climate hazards infrared precipitation with stations—a new environmental record for monitoring extremes. Scientific Data 2, 150066

MacFadyen S, Zambatis N, Van Teeffelen AJA, Hui C (2018) Long-term rainfall regression surfaces for the Kruger National Park, South Africa: A spatio-temporal review of patterns from 1981-2015. International Journal of Climatology 38(5): 2506-2519