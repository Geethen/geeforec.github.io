+++
authors = []
date = 2020-10-03T13:00:00Z
excerpt = "Fire frequency in the Kruger National Park, South Africa"
hero = "/images/prac6_f0.png"
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

**Introduction**
Fire is blah blah blah

***

**Data import**

In addition to datasets available on Google Earth Engine, in this practical we will learn how to import shapefiles from your local hard-drive into GEE by uploading new assets (Fig. 1).

```js
var dem = ee.Image("CGIAR/SRTM90_V4");
var FireCCI = ee.ImageCollection('ESA/CCI/FireCCI/5_1');
```

The first dataset, [SRTM90](https://developers.google.com/earth-engine/datasets/catalog/CGIAR_SRTM90_V4), is a Digital Elevation Model (DEM) from the Shuttle Radar Topography Mission (SRTM). The second, [FireCCI51](https://developers.google.com/earth-engine/datasets/catalog/ESA_CCI_FireCCI_5_1), is a Fire_cci Burned Area pixel product version 5.1 from MODIS. Figure 1 below, describes how to import the boundary shapefile for the Kruger National Park (Kruger) from files stored locally on your hard-drive. You can download and save the required files from [here](https://drive.google.com/file/d/1omD5vPk4LMQSnC2BHJCg6GlnmpzBsFQG/view?usp=sharing).

![](/images/prac6_f1.png)
**Figure 1:** Process to upload a shapefile into GEE as a new assest imported into the script as a FeatureColection 

***

**Filtering data**

First define your variables for the temporal and spatial windows of interest. We will use these variables to filter our data before processing.

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

Now build a function to remove all burn scars from the fire dataset that have a confidence interval of less than 50%. 
```js
// Define a function to remove all fires <50% confidence interval
var confMask = function(img) {
  var conf = img.select('ConfidenceLevel');
  var level = conf.gt(50);
  return img.updateMask(level).select('BurnDate'); //return only 'BurnDate' band
};
```

Run the function and summarise the burn scars by the day-of-year (doy) most frequently burnt, followed by the the frequency areas are burnt in Kruger annually from 2001 until 2018.

```js
// Most frequently burnt DOY
var fireDOY_list = years.map(function(year) {
  return fire
    .filterMetadata('year', 'equals', year) // Filter image collection by year
    .map(confMask) // Apply confidence mask >50%
    .reduce(ee.Reducer.mode()).rename('fire_doy') // Reduce image collection by most common DOY
    .set('year', year) // Set composite year as an image property
    .set('system:time_start', ee.Date.fromYMD(year, 1, 1));
});
var doyFires = ee.ImageCollection.fromImages(fireDOY_list); // Convert the image List back to an ImageCollection

// Frequency of days burnt
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

Summarise these results to represent the most frequently burnt day-of-year (doy) and the frequency areas have burnt in Kruger over the last 18 years (2001-2018).

```js
var modFires = doyFires.mode().clip(knp_geo);
var cntFires = cntFiresDOY.sum().clip(knp_geo);
```

***

**Charting**

To plot these results, first define your chart parameters (e.g. title and axis labels), then create the line chart, incorporating these pre-defined chart parameters and `print` it to the console as follows:

```js
// Chart parameters
var opt_cntFireMonth = {
  title: 'Monthly fire frequencies: Kruger National Park 2001 to 2018',
  pointSize: 3,
  hAxis: {title: 'Year'},
  vAxis: {title: 'Number of fires'},
};

// Plot day count of monthly fires
var cntFireMonth_chart = ui.Chart.image.series({ // ui.Chart.image.byRegion
  imageCollection: fire.select('BurnDate'),
  region: knp_geo,
  reducer: ee.Reducer.countDistinct(),
  scale: 250
}).setOptions(opt_cntFireMonth).setChartType('LineChart');
print(cntFireMonth_chart);
```

![](/images/prac6_f2a.png)
**Figure 2:** Line chart the number of days a fire occured in Kruger from 2001 to 2018.

***

**Visualisation**

To visualise the long-term summaries of your results, first setup your map elements as you've done in previous practicals.

```js
// Define legend parameters for unique DOY
// Light colours are earlier in the year, dark colours are later
var visDOY = {
    min:1, max:366, 
    palette:['eef5b7','99f74f','4ff7b0','4fc2f7',
    '3940db','7239db','db39db','db395c','7c1229']};

// Define a legend for fire frequency
// Light colours are areas that burn less frequently, 
// dark colours are areas that burn often
var visCnt = {
    min:1, max:12, 
    palette:['eef5b7','99f74f','4ff7b0','4fc2f7','3940db',
             '7239db','db39db','db395c','7c1229']};

Map.centerObject(knp, 7);
Map.addLayer(modFires, visDOY, 'Most frequently burnt day in Kruger (2001-2018)',true, 0.8)
Map.addLayer(cntFires, visCnt, 'Fire frequency: Kruger Park (2001-2018)',true, 0.8)
Map.addLayer(knp,{color: 'grey'}, 'Kruger',true, 0.8);  // Add Kruger boundary
```

![](/images/prac6_f3.png)
**Figure 3:** Map with layers indicating the most frequently burnt doy-of-year (doy) and the fire requency in Kruger from 2001 to 2018.

***

**Hillshade and Animation**


```js
//display hillshading and slope
var hillshade = ee.Terrain.hillshade(dem);
// print('Check hillshade',hillshade);

// Set the clipped SRTM image as background
var srtmVis = hillshade.visualize(srtmParams);

// Define GIF visualization parameters
var gifParams = {
  'region': knp_geo,
  'dimensions': 500,
  'crs': 'EPSG:3857', // Check projection
  'framesPerSecond': 1
};

// Define the hillshade background legend parameters
var srtmParams = {
  min: 120, // Elevation min 100
  max: 200, // Elevation max 840
  gamma: 1,
};

// Create RGB visualization images for use as animation frames.
var srtmFires = doyFires.map(function(img) {
  return srtmVis
        .paint(knp, 'grey', 1) // Add KNP boundary
        .blend(img.visualize(visDOY).clipToCollection(knp)); // Blend hillshade with knp and clip
});

// Print the GIF URL to the console
var myGIF = srtmFires.getVideoThumbURL(gifParams);
print(myGIF);

// Render the GIF animation in the console
print(ui.Thumbnail({
  image: srtmFires,
  params: gifParams,
  style: {
    position: 'bottom-right',
    width: '180px'
  }}));
```

![](/images/prac6_f4.gif)
**Figure 4:** Animation of days fires occured in Kruger from 2001 to 2018. Light colours represent fires that happened earlier in the year, while dark colours are those that burnt in later months.

***

**Data Export**

To export the data shown in the created charts, similar to practical 3, you may simply maximise the chart and then click to export to the available formats (csv, svg or png). Alternatively, you may script the export. this option benefits from having more options to customise the data export. For example, including numerous variables and, potentially a well- formatted date: time variable. In this practical, this is achieved by first using a reducer to get the mean rainfall value for Braulio Carrillo for each year and adding a date variable. The exported csv table will contain a column for both date and mean annual rainfall. you will find this csv file in your google drive.

As a last step, save the script.

***

**Practical 6 Exercise**

Repeat this practical but use NDVI instead of EVI and Germany instead of Costa Rica. You can also play around with different dates, keeping in mind the different date limits for each ImageCollection.
To share your script, click on Get Link and then copy script path. Send your completed script to [**email**](mailto:sandra@biogis.co.za).

Do you have any feedback for this practical? Please complete this quick (2-5 min) survey [here](https://forms.gle/hT11ReQpvG2oLDxF7).