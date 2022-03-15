---
layout: post
title: Project Reflections
---

### Overall, what did you achieve in your project?
I built a water shortage prediction model based on groundwater data.

### What are two aspects of your project that you are especially proud of?
At the start of the project, I did not expect the scale of the work needed to clean and transform the raw data. As such, I'm especially proud to have managed to gather the disparate data sources into a cohesive structure based on geography and time. I think this data work would provide a useful platform if I or others want to extend this type of research in the future. I'm also proud of the fact that I built a working model that produced statistically significant results, and in particular the fruitfulness of the using time-lagged measurements as predictors. I think some of the results could genuinely be of some use as a reference point for water management agencies trying to build a risk assessment framework.

### What are two things you would suggest doing to further improve your project?
An obvious way to improve my project would be to add more predictors like climate features, other hydrological conditions, and local economic conditions. I think these would markedly improve model accuracy. Secondly, I would improve the method of geographical aggregation - possibly using GeoPandas and shapefiles instead of coordinates. Thirdly, I would incorporate time and spatial lags in the same model instead of in separate models as I have it now. I think this would have been a simple fix had I a little more time.

### How does what you achieved compare to what you set out to do in your proposal?
Overall, this project falls far short of what I planned in my proposal. Some of the aforementioned potential imporvements were part of my proposal. Additionally, my proposal included components for user querying and for data scraping, which would have been nice additions but not essential. On the other hand, the data aggregation work and the methods to deal with spatial correlations went significantly beyond my proposal.

### What are three things you learned from the experience of completing your project?
1. I learned a lot about the PySAL package and its included methods to deal with spatial structures. For example, beyond the KNN-based weighting algorithm I used, I learned various other distance-based or adjacency-based methods to incorporate spatial lags depending on the nature of the data. As someone who has not had any exposure to the concept of spatial correlation and spatial regressions, I learned a lot from this exercise.
2. I got more familiar with the basic methods used when working with time series data. For example, to clean my measurements dataset, I had to resample at different frequencies, decide between interpolation and forward-filling to deal with missing data, compute first differences, and create lagged observations.
3. I'm glad to have learned a lot about the subject matter despite having little background. I'm still no expert in hydrology, but I'm happy that I gleaned some useful facts about the sort of modelling done in serious hydrological research and about the issues affecting water resource management.

### How will your experience completing this project will help you in your future studies or career?
I'll be going into economics research after I graduate with a view of doing a PhD, so the skills and tools I used in this project will open me up to a wider range of datasets I can use in my future research. In particular, spatial econometrics are pertinent to the fields of urban economics and widely used in economic history, both of which I'm interested in. Another transferable skill was organizing my code/data and managing project folders. Because of the amount of datasets I was using early on in the project, I had to come up with ways to keep track of the code for merging and cleaning my datasets. I'll have plenty more opportunities in the future to work on similar data projects, and the experience of this project will save me a lot of effort.
