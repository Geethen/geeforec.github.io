+++
authors = []
date = 2020-10-27T13:00:00Z
excerpt = "Fire frequency in the Kruger National Park, South Africa"
hero = "/images/prac6_f4.png"
timeToRead = 15
title = "Practical 6"

+++
**Practical 6: Fire frequency in the Kruger National Park, South Africa**

Access the completed practical script [here](https://code.earthengine.google.com/?scriptPath=users%2FBioGIS%2FbioGEE%3APractical_6%2Ffires_knp)

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

Then we will run the function and summarise the burn scars by the day-of-year (doy) most frequently burnt, followed by the the frequency areas are burnt in Kruger annually from 2001 until 2018.

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

Similarly, we can summarise these results to represent the most frequently burnt day-of-year (doy) and the most frequently burnt areas in Kruger over the last 18 years (2001-2018).

```js
var modFires = cntFiresDOY.mode().clip(knp_geo);
var cntFires = cntFiresDOY.sum().clip(knp_geo);
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

***

**Data Export**

To export the data shown in the created charts, similar to practical 3, you may simply maximise the chart and then click to export to the available formats (csv, svg or png). Alternatively, you may script the export. this option benefits from having more options to customise the data export. For example, including numerous variables and, potentially a well- formatted date: time variable. In this practical, this is achieved by first using a reducer to get the mean rainfall value for Braulio Carrillo for each year and adding a date variable. The exported csv table will contain a column for both date and mean annual rainfall. you will find this csv file in your google drive.

![](/images/prac4_f3.png)

In addition, to the export options presented above and in practical 3. You may also export the results as a rasterStack with multiple layers representing the sum of annual rainfall for Braulio Carrillo. We will first create a list of band names for the rasterStack output and apply the function toBands() to the image collection to stack all bands into a single image. Each band will contain a unique name corresponding to, in this example, the year of the annual sum.

Do you have any feedback for this practical? Please complete this quick (2-5 min) survey [here](https://forms.gle/hT11ReQpvG2oLDxF7).