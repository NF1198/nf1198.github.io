---
layout: post
title:  "Fixing Laptop Color with ColorHug2"
date:   2017-10-08 22:24:00 -0600
categories: ColorHug2, ICC, calibration
customjs:
 - https://ajax.googleapis.com/ajax/libs/jquery/3.1.0/jquery.min.js
---

Back in 2012, I purcahsed a Samsung 9-series laptop 
([NT900X4D-A58](http://www.samsung.com/sec/support/model/NT900X4D-A58)). 
The 9-series are supposed to be top-of-line laptops but one point has bothered
me from day-one: the screen color was horrible.

This laptop screen had a horrible bluish tint. It killed my eyes after only a few
hours of use. Colors were consistenly distorted which made the laptop impossible
to use for any kind of image processing. It might have been just the Korean
models that suffered from this problem, but all models I tested before purchasing
had the same issue.

I suffered with this problem for 5 years, ultimatley coming to resent the laptop. 

[ColorHug2](http://www.hughski.com/) to the rescue! ColorHug is an open-source
colorimeter, offered at a far cheaper price than similar commercial offerings.
At the heart of the ColorHug2 is the [Mazet](http://www.mazet.de) True Color 
Sensor -- at truly remarkable sensor!

Calibrating the display has given new life to my laptop. I only wish I had
found the ColorHug2 sooner!

Using [DisplayCal](https://displaycal.net/), I generated the following calibration curves to correct the display.

![NT900X4D Calibration Curves](/assets/nt900x4d-color/CalibrationCurves-NT900X4D.png)

*NT900X4D Calibration Curves*

![NT900X4D White Comparison - Simulate](/assets/nt900x4d-color/white-NT900X4D-A58-simulated.png)

*NT900X4D simulated white point (left: original, right: corrected)*

## Simulated Color Checker

I've also tried to simulate what colors looked like before calibration:

![NT900X4D Before Calibration - Simulated](/assets/nt900x4d-color/ColorChecker-1600x900-NT900X4D-A58-simulated.png)

*NT900X4D simulated color checker - before calibration*

Here is the correct color checker image (sRGB):

![ColorChecker sRGB](/assets/nt900x4d-color/ColorChecker-1600x900.png)

*sRGB color checker*

## Conclusion

The ColoHug2 can give new life to aging monitors and help you color-match multi-monitor
setups. I highly recommend it!

Download the screen calibration profile:
[NT900X4D-A58 Color Calibration](/assets/nt900x4d-color/NT900X4D-A58 2017-10-10 19-10 D10300 sRGB M-F 3xCurve+MTX.7z)


