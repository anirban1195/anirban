---
layout: post
title:  "Working with UVIT"
date:   2019-12-16 10:00:00 -0600
categories: jekyll update
---
UVIT is an Ultra Violet telescope on board ASTROSAT. It stands for Ultra Violet Imaging Telescope and is made up of 3 separate telescopes. Here I will mainly talk about the data processing pipeline. The raw data is processed by ISSDAC and the result is a Level 1 data file. The Level 1 data files contain all the necessary information to make the final image (which is done by the Level 2 pipeline). The basic philosophy is this. We have a list of photons and the time and position of their detection. If we simply plot the positions of the photons we should get the image. There is a small problem though. The position of each photon is in detector coordinates and telescope moves during observation. If we can somehow correct for this movement (also called drift and jitter), we should get our image and that is pretty much the basic framework. the pipeline. Now that we have the big picture lets dive in deeper. 
With the Level 1 product we get the following things 

* The FUV and NUV data in folder uvtF and uvtN.
* The VISible data in uvtV folder. 

These pretty much contain all the data you will need to make a decent image. These are fed through Data Ingest that creates a list of photons for FUV and NUV along with their time and position of detection. For VIS it creates images.

# Drift Series
Drift Series is the way the telescope moves during an observation. To find this we look at the VIS frames created by Data Ingest and identify the bright stars in each frame. We track these stars through the entire observation to determine the drift series. VIS frames are approximately spaces 1s apart . It is important we keep track of the movement of these bright stars as a function of time (usually MJD as per the VIS clock) . Sometimes due to a variety of reasons some data is missing and we have to skip frames. Skipping a few frames is fine, but if you skip too many frames then we have inaccurate drift series and hence bad angular resolution

# Making the Image
Making the image is the next step. For each photon detection, look at the time of detection and from the drift series find the drift vector form an arbitrary fixed reference and move the photon position by this vector. Doing this for all photons and plotting the final position give us the final image. 

However there are a bunch of factors that makes life more complicated 

* Presence of unexplained stripes in the VIS images.
* Different clocks of NUV , FUV and VIS. Sometimes the clocks are not completely synchronized.

# Results and Unresolved Issues
With every known systematic taken into account we get spectacular images. 
![NGC 2336]({{site.url}}{{site.baseurl}}/images/header_img.png)
*This is NGC 2336 in NUV. This was one of the very first observations of UVIT as extensively used for performance verification*

The target angular resolution was 1.8" but analyzing a typical star shows angular resolution of about 1.3"-1.4". Each pixel is about 0.3". 
![A Typical Star]({{site.url}}{{site.baseurl}}/images/typicalStar.png)

The FUV images are not as spectacular as NUV since there are too few photons. One of the most mysterious things we encountered (and is still unexplained) is these crosshair patterns. 
![Crosshair]({{site.url}}{{site.baseurl}}/images/crosshair.png)
It is a really subtle effect and without the correct color scheme in ds9 is is invisible to the naked eye. At first glance seems to be a faint reflection of some sort of mechanical support, but that is just a speculation. 


