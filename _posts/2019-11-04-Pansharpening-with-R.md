---
layout: post
title:  Pansharpening Remotely Sensed Data with R - An Introduction to Data Fusion
categories: [R, Remote Sensing, GIS, Information Visualization]
---

Data fusion is a method of combining different modes of data (whose degrees of heterogeneity can vary) into a single output. While the purpose of data fusion varies from project to project, the unifying goal is to extract information from the fused result that could not be easily obtained from either source alone. Bear in mind the quote, *"The whole is greater than the sum of its parts."*

Pansharpening is one common method of data fusion. Pansharpening is performed to increase the spatial resolution of remotely sensed imagery by fusing an image's panchromatic band with its multispectral bands. Due to the way in which a sensor captures an image, the panchromatic band always has a higher spatial resolution than its multispectral counterparts. With Landsat images, the panchromatic band has a spatial resolution of 15 meters, while the multispectral bands have a spatial resolution of 30 meters. ​

### Why use R for raster analysis? Why not use Python or some other software?
​
There is nothing wrong with using Python. You can certainly perform the same tasks in that language. However, performing raster analysis and geoprocessing is not a huge leap away from R's natural ability to work with matrices; R's primary function has traditionally been as a statistical, scientific, and data driven programming language. Since raster data is simply matrix data that has been given geographic context, it should be no surprise that R also works well as an alternative to other raster GIS software like TerrSet, ERDAS, or ENVI.

### The data and methods:

The R scripts below illustrate some simple pansharpening methods in data fusion.

The first attempts to follow the Intensity-Hue-Saturation (IHS) method described in the TerrSet/Idrisi help guide.

The other is a Brovey transformation method described in Jixian Zhang's 2010 publication, "Multi-Source Remote Sensing Data Fusion: Status and Trends" in the International Journal of Image and Data Fusion.

The sample Landsat data I used can be obtained from: [https://clarklabs.org/download/](https://clarklabs.org/download/){:target="_blank"}


#### IHS Transformation
```R
# Load packages
library(raster)

# Load Landsat images
pan <- raster("Data/etm_pan.rst")
etm2 <- raster("Data/etm2.rst")
etm3 <- raster("Data/etm3.rst")
etm4 <- raster("Data/etm4.rst")

# Resample the Images to the Panchromatic Band's resolution
etm2_resample <- resample(etm2, pan, method = "bilinear")
etm3_resample <- resample(etm3, pan, method = "bilinear")
etm4_resample <- resample(etm4, pan, method = "bilinear")

# Perform multiple regression, using panchromatic band as dependent variable
regress <- lm(values(pan)~values(etm2_resample) + values(etm3_resample) + values(etm4_resample))

# Create new raster using predicted values
regress_img <- raster(ncols = pan@ncols, nrows = pan@nrows, crs = crs(etm2), ext = extent(pan))
values(regress_img) <- predict(regress)

# Find the difference between the panchromatic band and the predicted values, and highlight differences via multiplication
d <- (pan - regress_img) * 2

# Add the differenced image values to the resampled bands
etm2_d <- d + etm2_resample
etm3_d <- d + etm3_resample
etm4_d <- d + etm4_resample

# "stack" the bands so they can be viewed as a composite image
ihs_stack <- stack(etm2_d, etm3_d, etm4_d)

# plot the composite image
plotRGB(ihs_stack, r = 3, g = 2, b = 1, stretch = "lin")
```

#### Brovey Transformation
```R
# find the average values of the multispectral (MS) bands
mul_avg <- (etm2_resample + etm3_resample + etm4_resample) / 3

# divide the pan band by the mean values of the MS bands and multiply that back into each MS band
brovey_2 <- pan / mul_avg * etm2_resample
brovey_3 <- pan / mul_avg * etm3_resample
brovey_4 <- pan / mul_avg * etm4_resample

brovey_stack <- stack(brovey_2, brovey_3, brovey_4)

plotRGB(brovey_stack, r = 3, g = 2, b = 1, stretch = "lin")
```

### Comparing Outputs:

<div style="display: flex;">
    <div style="flex: 50%; text-align: center;">
        <img src="/images/pansharpening/tcc_orig.png" alt="True Color Composite">
        <div style="font-weight: bold;">True Color Composite</div>
    </div>
    <div style="flex: 50%; text-align: center;">
        <img src="/images/pansharpening/pan_orig.png" alt="Panchromatic Band">
        <div style="font-weight: bold;">Panchromatic Band</div>
    </div>
</div>

<div style="display: flex;">
    <div style="flex: 50%; text-align: center;">
        <img src="/images/pansharpening/terrset-img_orig.png" alt="TerrSet IHS Method">
        <div style="font-weight: bold;">TerrSet IHS Method</div>
    </div>
    <div style="flex: 50%; text-align: center;">
        <img src="/images/pansharpening/r-ihs_1_orig.png" alt="Panchromatic Band">
        <div style="font-weight: bold;">IHS Method in R - Attempt at Emulating TerrSet Methodology</div>
    </div>
</div>

![Brovey Method](/images/pansharpening/brovey_orig.png){: width="375" height="300"}

<p style="text-align: center;"><b>Brovey Method</b></p>
<br/>

These images are zoomed in so you can more clearly compare methods. You can see that noise tends to increase from the spatial sharpening and there is some spectral distortion, which can make quantitative measurements of spectral change over time more difficult. These methods are generally most suitable for visual interpretation.

Also note the difference between the TerrSet output and the R IHS output. The R IHS output shows more visual contrast between features, but there is also more noise. If you want to perform this IHS method in R and want less noise, multiply the differenced values by a smaller value. However, be aware that this will also reduce visual contrast between features.

#### Works Referenced:
​
Jixian Zhang (2010) Multi-source remote sensing data fusion: status and trends, International Journal of Image and Data Fusion, 1:1, 5-24, DOI: 10.1080/19479830903561035