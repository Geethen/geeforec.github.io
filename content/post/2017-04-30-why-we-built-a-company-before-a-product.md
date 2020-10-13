---
title: Why we built a company before building a product and why it worked
date: 2017-04-30
hero: "/images/hero-6.jpg"
excerpt: Creating a new website for Hopper, one of the top 4 most downloaded travel
  apps in the U.S, along with Uber, Lyft.
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
'''js
**var filtered =** s21c.filterBounds(Theewaterskloof)

.filterDate('2019-05-01','2019-08-14')

.median();
'''
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