---
layout: post
title:  "Forced Measurement and Uncertainty of Faint Astronomical Sources"
date:   2024-5-26 10:00:00 -0600
categories: 10 Min Read
---
Recently we [published a new method to measure extremely faint astronomical sources where we generalized the concept of forced photometry](https://arxiv.org/abs/2405.12212). In simple words, given a reasonably good guess of the flux, centroid, shape and size of an astronomical sources, it seems one can recover a statistically better guess. The code can be found [here](https://github.com/anirban1195/measure) and in [PhoSim](https://bitbucket.org/phosim/phosim_release/downloads/?tab=tags) in sub-folder tools/measure/measure.cpp. The paper also contains some useful equations about uncertainties in measured flux, centroid, size and elliptcity as function of counts, area and background level.

We are using this for weak lensing shear measurements on the galaxy cluster Abell 2390 with images obtained from WIYN-ODI. It seems this cluster is a case of extreme late stage merger similar to Bullet Cluster. This has also been hinted in publications analyzing/using X-Ray images from Chandra.  Publication coming soon !  
