+++
authors = []
date = 2020-10-22T22:00:00Z
excerpt = "Long-term patterns of rainfall in and around Braulio Carrillo National Park, Costa Rica"
hero = "/images/prac4_f0_final.png"
timeToRead = 15
title = "Practical 4"

+++
**Practical 4: Long-term patterns of rainfall in and around Braulio Carrillo National Park in Costa Rica**

Access the completed practical script [here](https://code.earthengine.google.com/eacc7c0fad832d1fbba267538f137f20?noload=true)

**Learning Objectives**
By the end of this practical you should be able to:

1. Access long-term rainfall data
2. Summarise temporal data by region
3. Display results on a map and in time-series plots
4. Make an interactive map (GEE App)
5. Compare rainfall and vegetation 'greenness" (i.e. rainfall as a driver of ecosystem processes like vegetation dynamics)
6. Output data (csv, rasterStack) for analysis outside of GEE

***

**Introduction**
Rainfall plays a central role in a myriad of natural processes, including river health, the transportation of nutrients, soil moisture, vegetation dynamics, fire regimes, animal movement and distribution patterns and landscape heterogeneity. Within protected areas these processes function together to safeguard ecosystem integrity. In the face of current climate change predictions, the spatio-temporal patterns of rainfall is an increasingly important component to include in any ecological study (MacFadyen et al 2018). Here we explore patterns of monthly rainfall across Costa Rica and the Braulio Carrillo National Park from 1981 to 2019 (39 years). We'll summarise monthly and annual rainfall patterns using line charts and examine the long-term spatial patterns of rainfall using an interactive map. Lastly, we'll take a look at how the temporal patterns of annual rainfall compares to those of vegetation vigour or 'greenness', highlighting its importance as a bottom-up ecosystem driver.

***

**Data import**
The datasets we will use for this practical are all available on Google Earth Engine and can be accessed as follows (You can convert these to an import record, using `convert` from the pop-up message):

```js
var Countries = ee.FeatureCollection('USDOS/LSIB/2017');
var WDPA = ee.FeatureCollection('WCMC/WDPA/current/polygons');
var CHIRPS = ee.ImageCollection("UCSB-CHG/CHIRPS/PENTAD");
var MOD13Q1 = ee.ImageCollection("MODIS/006/MOD13Q1");
```

The first dataset, [LSIB 2017](https://developers.google.com/earth-engine/datasets/catalog/WCMC_WDPA_current_polygons), contains polygon representations of all international boundaries. The second, [WDPA](https://developers.google.com/earth-engine/datasets/catalog/WCMC_WDPA_current_polygons), contains polygons of all the world's protected areas. The third, [CHIRPS](https://developers.google.com/earth-engine/datasets/catalog/UCSB-CHG_CHIRPS_PENTAD), is a gridded rainfall time series dataset (Funk et al 2015) and the last, [MOD13Q1](https://developers.google.com/earth-engine/datasets/catalog/MODIS_006_MOD13Q1), provides vegetation indexes (NDVI and EVI) depicting vegetation 'greenness' per 250m pixel.

***

**Filtering data**
First define variables for the temporal window of interest, including a start-date, end-date and the range of years and months. We will use these variables later to filter and summarise the long-term data.

```js
var startDate0 = ee.Date.fromYMD(1981,1,1);
var startDate = ee.Date.fromYMD(2000,1,1);
var endDate = ee.Date.fromYMD(2019,12,31);
var years0 = ee.List.sequence(1981, 2019);
var years = ee.List.sequence(2000, 2019);
var months = ee.List.sequence(1, 12);
```

Then filter our polygon features to our areas of interest (AOI), namely Costa Rica and Braulio Carrillo protected area. At the same time, convert these two FeatureCollections to geometry objects as we'll need them later as function parameters for functions we'll build.

```js
// Country boundaries filtered to Costa Rica
var costaRica = Countries.filter(ee.Filter.inList('COUNTRY_NA', ['Costa Rica']));
// WDPA boundaries filtered to 'Braulio Carrillo' National Park in Costa Rica
var braulio = WDPA.filter(ee.Filter.stringContains('ORIG_NAME', 'Braulio Carrillo'));
    
// Convert features to geometry objects
var costaRica_geo = costaRica.geometry(); 
var braulio_geo = braulio.geometry(); 
```

Now filter the CHIRPS ImageCollection for rainfall (i.e. `'precipitation'`) and the MODIS MOD13Q1 product for the Enhanced Vegetation Index (EVI) instead of the Normalized Difference Vegetation Index (NDVI) used in the previous practical. At the same time, filter by date range and our AOI to speed up all analyses that follow.

```js
// Long-term rainfall data from CHIRPS
var rainAll = CHIRPS.select('precipitation')
    .filterDate(startDate, endDate)
    .filterBounds(costaRica_geo);
// print('Çheck rainAll', rainAll);

// Long-term EVI data from MODIS
var eviAll = MOD13Q1.select('EVI')
    .filterDate(startDate, endDate)
    .filterBounds(costaRica_geo);
```

***

**Processing**
To calculate the annual monthly sum of rainfall across Braulio Carrillo National Park from 1981 to 2019, reduce the monthly rainfall records by their sum per year as follows:

```js
var rainYr_list =  years.map(function(y){
  var rain_year = rainAll.filter(ee.Filter.calendarRange(y, y, 'year')).sum().rename('rain_yr');
  return rain_year.set('year', ee.Date.fromYMD(y,1,1)); 
});
```

This produces a list of 20 images, which you'll need to convert back to an ImageCollection. Then calculate the long-term annual rainfall patterns for Costa Rica as follows:

```js
// Convert the image List to an ImageCollection.
var rainYr = ee.ImageCollection.fromImages(rainYr_list);
// Calculate long-term annual rainfall clipped to Costa Rica
var rainAnnual = rainYr.mean().clip(costaRica);
```

***

**Visualisation**
Now let's plot these results on a map. First define the various map elements i.e. the `title` and `symbology` for the legend as follows:

```js
var title = ui.Label('Costa Rica: Annual Rainfall 1981 to 2018', {
  stretch: 'horizontal',
  textAlign: 'center',
  fontWeight: 'bold',
  fontSize: '20px'});

var rainYvis = {
  min: 2100, max: 3800, 
  palette: 'ffffff, 67d9f1, 1036cb'};
```

Next, we centre the map to Costa Rica - In this way all other layers will align with this parent map. Now add the long-term mean annual rainfall for Costa Rica and the Braulio Carrillo boundary. Lastly add the defined title.

```js
Map.centerObject(costaRica, 8);
Map.addLayer(rainAnnual, rainYvis, 'Long-term Annual Rainfall'); 
Map.addLayer(braulio,{color: 'grey'}, 'Braulio Carrillo',true, 0.8);  
Map.add(title);
```

![](/images/prac4_f1_final.png)
**Figure 1:** Map of long-term annual rainfall in Costa Rica from 1981 to 2019.

***

**Charting**
Now chart annual rainfall results but summarised for Braulio Carrillo National Park using a line chart. To do this, first define the chart parameters (e.g. title and axis labels) and then create the line chart.

```js
var opt_chart_annualPrecip = {
  title: 'Annual Rainfall: Braulio Carrillo National Park',
  pointSize: 2,
  hAxis: {title: 'Year'},
  vAxis: {title: 'Rainfall (mm)'},
};

var chart_annualPrecip = ui.Chart.image.series({
  imageCollection: rainYr,
  region:aoi_clip,
  reducer: ee.Reducer.mean(),
  scale: 5000,
  xProperty: 'year'
}).setOptions(opt_chart_annualPrecip);
// print(chart_annualPrecip);
```

We could use `print()` to see the chart in the GEE console but for this practical we'll add the chart to our map  layout instead. To do this, first make a panel to hold the chart in the map, then add the empty panel to the map and fill it with your chart:

```js
// Create a panel to hold the chart in the map
var panel = ui.Panel();
panel.style().set({
  width: '350px',
  position: 'bottom-left'
});
// Add empty panel to map
Map.add(panel);
// Add chart to panel
panel.add(chart_annualPrecip)
```

![](/images/prac4_f2_new.png)
**Figure 2:** Line chart of annual rainfall in Braulio Carrillo National Park from 1981 to 2019.

***

**Save your map online**
Now for the fun part! We can share this map with the world by creating a GEE Application (App). To save your map online as your first GEE App, follow the steps below:

1. Click the `Apps` button above
2. Select `NEW APP`
3. Give the App a Name and click `PUBLISH`. Leave everything else default
4. Your new URL will appear - Click this to see your first online interactive map (if you get an error message, follow the steps below instead).
5. If you see a `Not ready` page, give it a few minutes and try again

![](/images/prac4_f3a_fix.png)
**Figure 3:** Steps to publish interactive map online. Use URL to access.

If you get an error message, chances are you haven't accepted the terms and conditions of using GEE Apps in your Google Cloud Platform. To do so, follow these steps instead of the ones above.

 1. Click the `Apps` button above
 2. Select `NEW APP`
 3. Give the App a Name and click `PUBLISH`
 4. When the error appears, click the `Cloud Terms of Service` link
 5. This will open the `Cloud Platform Console`
 6. Go down to `App Engine` and select `Dashboard`
 7. You'll be asked to agree to the Terms of Service, `AGREE AND CONTINUE`
 8. Close your `Cloud Platform Console` and go back to GEE and click `NEW APP` again
 9. Give the App a Name and click `PUBLISH`. Leave everything else default
10. Your new URL will appear - Click this to see your first online interactive map
11. If you see a `Not ready` page, give it a few minutes or refresh the page and try again

![](/images/prac4_f3b.png)
**Figure 4:** Additional steps to publish interactive map online. Use URL to access.

**And voilà! Your first GEE App!**

![](/images/prac4_f5_final.png)
**Figure 5:** Your first online GEE App / interactive map interface.

***

**Relationship between annual rainfall and vegetation 'greenness'**
Taking things even further, we could also combine the calculation of annual max rainfall with annual maximum EVI for Costa Rica for the same period, 2000 to 2019, to see whether there's a relationship between the two. What do you think we'll see?

```js
// Calculate annual mean for every year for both Rainfall and EVI imageCollections
var annualRainEVI_list =  years.map(function(y){
  var evi_year = eviAll.filter(ee.Filter.calendarRange(y, y, 'year'))
    .max().multiply(0.0001).rename('evi'); // Need to scale by constant
  var rain_year = rainAll.filter(ee.Filter.calendarRange(y, y, 'year'))
    .max().rename('rain');
  return rain_year.addBands(evi_year).set('year', ee.Date.fromYMD(y,1,1));
});
```

Remember to convert the list that is returned, back to an ImageCollection, including a `flatten()` command as follows:

```js
// Convert the image List to an ImageCollection.
var annualRainEVI = ee.ImageCollection.fromImages(annualRainEVI_list.flatten());
```

Now plot both rainfall and EVI in a single chart.
To create a comparative line chart of rainfall and EVI summaries for Braulio Carrillo, first define the chart parameters and then create the line chart with two y-axes as follows:

```js
// Display a comparative line chart of rainfall and EVI for Braulio Carrillo 
// Set chart parameters e.g. title
var opt_annualRainEVI = {title: "Annual Max Rainfall vs. 'Greenness (EVI) for Braulio Carrillo", pointSize: 2,
    legend: {maxLines: 5, position: 'top'},
    series: { 0: {targetAxisIndex: 0},
              1: {targetAxisIndex: 1}},
        vAxes: {// Adds titles to each axis.
          0: {title: 'Max EVI (*0.0001)'},
          1: {title: 'Max Rainfall (mm)'}},};
          
// Build the chart and plot it          
var rain_ndvi_chart = ui.Chart.image.series({
  imageCollection: annualRainEVI.select(['evi', 'rain']),
  region:aoi_clip,
  reducer: ee.Reducer.mean(),
  scale: 5000,
  xProperty: 'year'
}).setOptions(opt_annualRainEVI);
print(rain_ndvi_chart);
```

![](/images/prac4_f6_final.png)
**Figure 6:** Dual axis chart of annual maximum rainfall versus vegetation 'greenness' or vigour using a MODIS Enhanced Vegetation Index (EVI) in Braulio Carrillo National Park from 2000 to 2019.

***

We could also directly check the correlation between the two variables as follows:

```js
// Compute a Pearson's product-moment correlation coefficient 
// and the 2-sided p-value test for correlation = 0.
var opt_correl = {title: "Correlation between Rainfall and EVI for Braulio Carrillo", pointSize: 2,
    legend: {maxLines: 5, position: 'top'},
    series: { 0: {targetAxisIndex: 0},
              1: {targetAxisIndex: 1}},
        vAxes: {// Adds titles to each axis.
          0: {title: 'Correlation - R2)'},
          1: {title: 'P-Value'}},};
// Build the chart and plot it          
var correl_chart = ui.Chart.image.series({
  imageCollection: annualRainEVI.select(['evi', 'rain']),
  region:aoi_clip,
  reducer: ee.Reducer.pearsonsCorrelation(),
  scale: 5000,
  xProperty: 'year'
}).setOptions(opt_correl);
print(correl_chart);
```

![](/images/prac4_new_f8.png)
**Figure 7:** Results of a linear least squares regression with rainfall as the independent variable and EVI as the dependent variable.

***

**Data Export**
To export the data shown in the created charts, you can simply `maximise` the chart and then click `Download` to export to formats `.CSV`, `.SVG` or `.PNG`.

![](/images/prac4_f5.png)
**Figure 8:** The easiest way to export data plotted in a chart is to click the `maximise` button on the chart in your console area (1) and then click `Download CSV` (2) to export a .csv table to your local hard-drive.

You can also script the export. This option will allow you to customise formats for your exported table. For example, a formatted date field using a reducer to get the mean rainfall value for Braulio Carrillo for each year. The exported csv table will then contain a column for both date and mean annual rainfall. Once the task is completed, you will find this csv file in your google drive.

```js
var csv_annualPrecip = rainYr.map(function(image){
  var year = image.get('year');
  var mean = image.reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: aoi_clip,
    scale: 5000
  });  
  return ee.Feature(null, {'mean': mean.get('rain_yr'),'year': year})
});

// Export a .csv table of year and mean rainfall
Export.table.toDrive({
  collection: csv_annualPrecip,
  description: 'annualRain',
  folder: 'testOTS',
  fileFormat: 'CSV'
});
```

![](/images/prac4_f8_fix.png)
**Figure 9:** Steps followed to complete a CSV export task using a script to initialise a table export to your local hard-drive.

The last step, as always, is to save the script.

***

**Practical 4 Exercise**
Repeat this practical but use NDVI instead of EVI and Germany instead of Costa Rica. You can also play around with different dates, keeping in mind the different date limits for each ImageCollection. To share your script, click on Get Link and then copy script path. Send your completed script to [ots.online.education@gmail.com](mailto:ots.online.education@gmail.com). If you're feeling adventurous, save the results as a new App and forward the URL link along with your script.

Do you have any feedback for this practical? Please complete this quick (2-5 min) survey [here](https://forms.gle/hT11ReQpvG2oLDxF7).

***

**References**
Funk C, Peterson P, Landsfeld M, Pedreros D, Verdin J, Shukla S, Husak G, Rowland J, Harrison L, Hoell A, Michaelsen J (2015) The climate hazards infrared precipitation with stations—a new environmental record for monitoring extremes. Scientific Data 2, 150066

MacFadyen S, Zambatis N, Van Teeffelen AJA, Hui C (2018) Long-term rainfall regression surfaces for the Kruger National Park, South Africa: A spatio-temporal review of patterns from 1981-2015. International Journal of Climatology 38(5): 2506-2519

***

**Bonus Section**
Similarly, you can calculate monthly rainfall for each year in Braulio Carrillo National Park from 1981 to 2019 by reducing the monthly rainfall records by their sum per year and month as follows:

```js
var rainMeanMY_list = years.map(function(y) {
  return months.map(function(m) {
  return rainAll
    .filter(ee.Filter.calendarRange(y,y, 'year')) // Filter images by year
    .filter(ee.Filter.calendarRange(m,m, 'month')) // Filter images by month
    .reduce(ee.Reducer.sum()).rename('rain_mnth') // Reduce the resulting image collection by the sum
    .set('yrmnth', ee.Date.fromYMD(y, m, 1)) // .set('year',ee.Date(ee.String(y)).format('YYYY'))
    .set('name','rainfall'); // Name the new property
});
});
```

Remember to convert the list that is returned, back to an ImageCollection and include a `flatten()` command as follows:

```js
// Convert the image List to an ImageCollection.
var rainMeanMY = ee.ImageCollection.fromImages(rainMeanMY_list.flatten());
```

Then make your line chart, the same way you did for the annual rainfall above.

```js
// Display a line chart of the annual, monthly rainfall for Braulio Carrillo
// First setup the chart properties 
var opt_rainMeanMY = {
  title: 'Annual Monthly Mean Rainfall for Braulio Carrillo',
  hAxis: {title: 'Date'},
  vAxis: {title: 'Rainfall (mm)'},
};
// And then the actual chart
var chart_rainMeanMY = ui.Chart.image.seriesByRegion({
  imageCollection: rainMeanMY, 
  regions: aoi_clip,
  reducer: ee.Reducer.mean(),
  scale: 5000,
  xProperty: 'yrmnth',
  seriesProperty: 'name'
}).setOptions(opt_rainMeanMY)
  .setChartType('LineChart');//'ColumnChart
print(chart_rainMeanMY);
```

![](/images/prac4_f7.png)
**Figure 10:** Line chart of annual monthly rainfall in Braulio Carrillo National Park from 1981 to 2019.

In addition, to the export options presented above and in practical 3. You could also export the results as a `rasterStack` with multiple layers representing the sum of monthly rainfall per year for Costa Rica. To do so, first create a list of band names for the `rasterStack` and apply the function `toBands()` to the ImageCollection. This will stack all bands into a single image. Each band will contain a unique name corresponding to, in this example, the year and the month of summed rainfall.

```js
// Create list of band names for rasterStack output
var band_names = rainMeanMY.map(function(image) {
      return ee.Feature(null, {'yrmnth': ee.Date(image.get('yrmnth'))
      .format('YYYY_MM')})})
      .aggregate_array('yrmnth')

// Apply the function toBands() on the image collection to stack all bands into one image
var stack_RainEVI = rainMeanMY.toBands().rename(band_names);
print('Check Names',stack_RainEVI); // Use print() to check your results

// Export a cloud-optimized GeoTIFF.
// i.e. rasterStack with 468 layers, representing monthly rainfall each year from 1981 to 2019
Export.image.toDrive({
  image: stack_RainEVI,
  folder: 'testOTS',
  description: 'Rainfall_Sums',
  scale: 5000,
  region: aoi_clip,
  fileFormat: 'GeoTIFF',
  formatOptions: {
    cloudOptimized: true
  }
});
```

***