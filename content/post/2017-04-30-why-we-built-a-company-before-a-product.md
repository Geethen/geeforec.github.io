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

**Learning Objectives**

By the end of this practical you should be able to:

1. Compute the Normalised Difference Vegetation Index (NDVI) and Normalised Difference Water Index (NDWI).
2. Interpret both the NDVI and NDWI layer.
3. Use these two spectral indices to detect water.
4. Understand the influence of atmospheric interference on reflectance and detecting water.

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

**var NDVI =** filtered.normalizedDifference(\['B8','B4'\]);

**var NDWI =** filtered.normalizedDifference(\['B3','B8'\]);

  .normalizedDifference   Computes (band 1- band 2) / (band 1+ band 2); band 1 is the band specified first in the list and band 2 is the second.
  ----------------------- ------------------------------------------------------------------------------------------------------------------------
                          

***Visualisation***

We first specify a palette to be used when visualising the indices. We
can create a colour palette using strings (e.g., ‘green’, ‘red’) or we
can use hex colour codes. The NDVI and NDWI spectral indices both range
from -1 to 1. However, for NDVI we specify a minimum of zero to improve
the visualisation of NDVI. This may not always be necessary.

**var vis =** \['FFFFFF', 'CE7E45', 'DF923D', 'F1B555', 'FCD163',
'99B718',

'74A901', '66A000', '529400', '3E8601', '207401', '056201',

'004C00', '023B01', '012E01', '011D01', '011301'\];

**Map.addLayer(**NDVI,{min: 0, max: 1, palette: vis},'NDVI'**);**

**Map.addLayer(**NDWI,{min: -1, max: 1, palette: vis},'NDWI'**);**

![](media/image7.png){width="4.0625in"
height="3.96875in"}***Interpreting spectral indices***

**Figure 2:** Spectral indices take advantage of the spectral properties
of land cover. For instance, as highlighted in the theory lecture,
vegetation has a high reflectance in the Near-Infrared (NIR) region
while having a low reflectance in the red portion of the electromagnetic
(EM) spectrum. As a result of using these bands to compute the NDVI, the
index corresponds to the greenness of vegetation and has been shown to
be correlated to various vegetation parameters such as vegetation
health, nutrient levels and plant phenophase. Similarly, NDWI is mainly
sensitive to water.

**Detecting water**

There are numerous methods available to detect surface water and this
area of research is still an active one. In this course you will be
introduced to two of these general approaches i.e. supervised
classification and thresholding. For this practical, you will use a very
simple thresholding-based approach that employs the computed NDVI and
NDWI spectral indices.

**Map.addLayer(**NDWI.gt(NDVI),{},'water\_1c'**);**

  .gt   Returns a binary raster, with values of 1 when the inequality is satisfied and a 0 if it is not. Belongs to a family of inequalities, such as (gte, lte, lt, eq, neq)
  ----- -----------------------------------------------------------------------------------------------------------------------------------------------------------------------
        

In the above code snippet, the function gt() returns a 1 if the first
value is greater than the second, creating a binary raster i.e. a value
of 1 is returned when NDWI values are greater than those of the
corresponding NDVI pixel. This inequality can largely be useful to
detect open surface water. However, upon inspection you will come across
inevitable omission and commission errors.

***The influence of atmospheric effects on water detection.***

![](media/image13.png){width="3.4166666666666665in"
height="4.114583333333333in"}Recently, there has been a drive towards
Analysis Ready Data (ARD) i.e. in part, this includes data that has
already been corrected for atmospheric interferences, however, there is
a considerable amount of uncertainty and variance associated with the
results of different atmospheric correction algorithms. This, together
with the fact that atmospherically corrected Sentinel-2 data is not
available for areas outside of Europe from 2015 (the start of the
archive) to 2018 i.e. level 1C (atmospherically uncorrected) data is
only available. It is therefore important to understand the effects of
atmospheric interference on water detection.

**Figure 3:** The influence of atmospheric interference on the detection
of water. Water detected from atmospherically corrected Sentinel-2,
level 2A (top), reference water detection based on the long-term surface
water from Landsat-8 made available through the Global Surface Water
product (centre), water detected from atmospherically uncorrected
Sentinel-2, level 1C data (bottom).

***Practical 2 Exercise***

Repeat the steps in this practical for the level 2A data that you
imported and renamed as s22a at the beginning of this practical.
Thereafter, compare the water detection results and patterns with the
s21c image results. Submit your final script.
```js
import React from "react";
import { graphql, useStaticQuery } from "gatsby";
import styled from "@emotion/styled";

import * as SocialIcons from "../../icons/social";
import mediaqueries from "@styles/media";

const icons = {
  dribbble: SocialIcons.DribbbleIcon,
  linkedin: SocialIcons.LinkedinIcon,
  twitter: SocialIcons.TwitterIcon,
  facebook: SocialIcons.FacebookIcon,
  instagram: SocialIcons.InstagramIcon,
  github: SocialIcons.GithubIcon,
};

const socialQuery = graphql`
  {
    allSite {
      edges {
        node {
          siteMetadata {
            social {
              name
              url
            }
          }
        }
      }
    }
  }
`;

function SocialLinks({ fill = "#73737D" }: { fill: string }) {
  const result = useStaticQuery(socialQuery);
  const socialOptions = result.allSite.edges[0].node.siteMetadata.social;

  return (
    <>
      {socialOptions.map(option => {
        const Icon = icons[option.name];

        return (
          <SocialIconContainer
            key={option.name}
            target="_blank"
            rel="noopener"
            data-a11y="false"
            aria-label={`Link to ${option.name}`}
            href={option.url}
          >
            <Icon fill={fill} />
          </SocialIconContainer>
        );
      })}
    </>
  );
}
```

But it takes more than good ideas to build and grow a business. It takes people to bring them into reality. Are those people collaborating and sharing their expertise, or are they in conflict and keeping it to themselves?

# This is a primary heading

Do they have the resources necessary to execute on their ideas? Or are they constantly under pressure to pluck only the lowest-hanging fruit through bare minimum means, while putting their greatest ambitions on the back-burner?

> Blockquotes are very handy in email to emulate reply text.
> This line is part of the same quote.

But it takes more than good ideas to build and grow a business. It takes people to bring them into reality. Are those people collaborating and sharing their expertise, or are they in conflict and keeping it to themselves?

> This is a very long line that will still be quoted properly when it wraps. Oh boy let's keep writing to make sure this is long enough to actually wrap for everyone. Oh, you can _put_ **Markdown** into a blockquote.

These are the circumstances that suffocate creativity and destroy value in an organization. That’s why I knew that if I was going to start a company, our first product would have to be the company itself. These are the circumstances that suffocate creativity and destroy value in an organization. That’s why I knew that if I was going to start a company, our first product would have to be the company itself.

## This is a secondary heading

```jsx
import React from "react";
import { ThemeProvider } from "theme-ui";
import theme from "./theme";

export default props => (
  <ThemeProvider theme={theme}>{props.children}</ThemeProvider>
);
```

These are the circumstances that suffocate creativity and destroy value in an organization. That’s why I knew that if I was going to start a company, our first product would have to be the company itself. These are the circumstances that suffocate creativity and destroy value in an organization. That’s why I knew that if I was going to start a company, our first product would have to be the company itself. These are the circumstances that suffocate creativity and destroy value in an organization. That’s why I knew that if I was going to start a company, our first product would have to be the company itself. These are the circumstances that suffocate creativity and destroy value in an organization. That’s why I knew that if I was going to start a company, our first product would have to be the company itself.

***

Hyphens

***

Asterisks

***

Underscores

These are the circumstances that suffocate creativity and destroy value in an organization. That’s why I knew that if I was going to start a company, our first product would have to be the company itself. These are the circumstances that suffocate creativity and destroy value in an organization. That’s why I knew that if I was going to start a company, our first product would have to be the company itself.

Do they have the resources necessary to execute on their ideas? Or are they constantly under pressure to pluck only the lowest-hanging fruit through bare minimum means, while putting their greatest ambitions on the back-burner?

Emphasis, aka italics, with _asterisks_ or _underscores_.

Strong emphasis, aka bold, with **asterisks** or **underscores**.

Combined emphasis with **asterisks and _underscores_**.

Strikethrough uses two tildes. ~~Scratch this.~~

1. First ordered list item
2. Another item
   ⋅⋅* Unordered sub-list.
3. Actual numbers don't matter, just that it's a number
   ⋅⋅1. Ordered sub-list
4. And another item.

⋅⋅⋅You can have properly indented paragraphs within list items. Notice the blank line above, and the leading spaces (at least one, but we'll use three here to also align the raw Markdown).

⋅⋅⋅To have a line break without a paragraph, you will need to use two trailing spaces.⋅⋅
⋅⋅⋅Note that this line is separate, but within the same paragraph.⋅⋅
⋅⋅⋅(This is contrary to the typical GFM line break behaviour, where trailing spaces are not required.)

* Unordered list can use asterisks
* Or minuses
* Or pluses