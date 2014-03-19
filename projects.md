---
layout: page
title : Projects
theme :
---


#### ☞  Wind Speed/Direction Analysis Tool with Fourier/Kalman Filtering and Vector Analysis

*Sep. 2011 ~ May. 2012, Fudan University*

This software is implemented to provide various analysis methods of the raw data, including extracting, calculating, analyzing and display them through curve graphs, bar graphs, tables, etc.. While the program provides basic data operations including excluding large or small values caused by mistakes and cutting time series while total time range is large make it possible to zoom in or out to watch the curve graphs vertically and horizonally, it also offers data analysis with Digital Fileters. Finite Impulse Reponse (FIR) Digital Filters supported by Fast Fourier Transform (FFT) and Inverse Fast Fourier Transform (IFFT) with Window Methods and Kalman Filters are available to deal with data values collected. And Kalman Filtering has been used to cut down the error of values and to make the curves smoother. This article has made descriptions to the two filters in detail. In the statistical analysis of these data values, vector calculation method was used to make statistical results more applicable to their physical features.

![timespan.png]({{BASE_PATH}}/images/projects/timespan.png)

*Zoom in on the axis of time span*

![fft.png]({{BASE_PATH}}/images/projects/fft.png)

*Fast Fourier Transform Filtering with different size of windows*
    
![kalman-speed.png]({{BASE_PATH}}/images/projects/kalman-speed.png)

*Before (up)/After (down) Kalman Filtering the data of wind speed*

![kalman-dir.png]({{BASE_PATH}}/images/projects/kalman-dir.png)

*Before (up)/After (down) Kalman Filtering the data of wind direction*

<br />
**\[Keywords\]**  Fast  Fourier  Transform  (FFT),  Inverse  Fourier Transform (IFFT), Kalman Filtering, Data Visualization, Vector Analysis, C++

For detailed reference, please read: [Wind Speed/Direction Analysis Tool with Fourier/Kalman Filtering and Vector Analysis]({{BASE_PATH}}/files/GraduationThesis.pdf)

--------------------------------------------------------------------------------------------

#### ☞  I.O.T Application System Based on CC430 at 433MHz

*Dec. 2010 ~ May. 2011, Fudan University*

The project is a prototype system for I.O.T applications. The project is supposed to be applied in a facility agriculture scenario for intelligent irrigation, which is mainly based on the Crop Growth Models algorithm design. 

We designed and implemented our own networking protocal for communication between CC430 transceivers, Crop Growth Models, UI interface, database, etc. 

For detailed reference, please read: [IOT Application System with Crop Growth Models in Facility Agriculture](http://ieeexplore.ieee.org/xpl/articleDetails.jsp?tp=&arnumber=6316590&searchWithin%3Dp_Authors%3A.QT.Xiangyu+Hu.QT.)
