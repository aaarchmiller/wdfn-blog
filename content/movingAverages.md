---
author: Laura DeCicco
date: 2016-10-25
slug: moving-averages
draft: True
title: Moving Averages on NWIS Data
type: post
categories: Data Science
image: static/moving-averages/unnamed-chunk-6-1.png
tags: 
  - R
  - dataRetrieval
 
description: Using the R-packages dataRetrieval, dplyr, and ggplot2, a simple discription on how to create a moving average plot.
keywords:
  - R
  - dataRetrieval
 
 
 
---
<a href="mailto:ldecicco@usgs.gov"><i class="fa fa-envelope-square fa-2x" aria-hidden="true"></i></a>
<a href="https://twitter.com/DeCiccoDonk"><i class="fa fa-twitter-square fa-2x" aria-hidden="true"></i></a>
<a href="https://github.com/ldecicco-usgs"><i class="fa fa-github-square fa-2x" aria-hidden="true"></i></a>
<a href="https://scholar.google.com/citations?hl=en&user=jXd0feEAAAAJ"><i class="ai ai-google-scholar-square ai-2x" aria-hidden="true"></i></a>
<a href="https://www.researchgate.net/profile/Laura_De_Cicco"><i class="ai ai-researchgate-square ai-2x" aria-hidden="true"></i></a>
<a href="https://www.usgs.gov/staff-profiles/laura-decicco"><i class="fa fa-user fa-2x" aria-hidden="true"></i></a>

This post will show simple way to create a moving average plot of discharge. The goal is to reproduce the graph at this link: [PA Graph](http://pa.water.usgs.gov/drought/indicators/sw/images/f30_01538000.html)

First we get the data using the [dataRetrieval](https://CRAN.R-project.org/package=dataRetrieval) package. The siteNumber and parameterCd could be adjusted for other sites or measured parameters. In this example, we are getting discharge (parameter code 00060) at a site in PA.

``` r
library(dataRetrieval)

#Retrieve daily Q
siteNumber<-c("01538000")
parameterCd <- "00060" #Discharge
dailyQ <- readNWISdv(siteNumber, parameterCd) 
dailyQ <- renameNWISColumns(dailyQ)
stationInfo <- readNWISsite(siteNumber)
```

Next, we calculate a 30-day moving average on all of the flow data:

``` r
library(dplyr)

#Check for missing days, if so, add NA rows:
if(as.numeric(diff(range(dailyQ$Date))) != (nrow(dailyQ)+1)){
  fullDates <- seq(from=min(dailyQ$Date),
                   to = max(dailyQ$Date), by="1 day")
  fullDates <- data.frame(Date = fullDates, 
                          agency_cd = dailyQ$agency_cd[1],
                          site_no = dailyQ$site_no[1],
                          stringsAsFactors = FALSE)
  dailyQ <- full_join(dailyQ, fullDates,
                      by=c("Date","agency_cd","site_no")) %>%
    arrange(Date)
}

ma <- function(x,n=30){stats::filter(x,rep(1/n,n), sides=1)}

dailyQ <- dailyQ %>%
  mutate(rollMean = as.numeric(ma(Flow)),
         day.of.year = as.numeric(strftime(Date, 
                                           format = "%j")))
```

We can use the `quantile` function to calculate historical percentile flows. Then use the `loess` function for smoothing. The argument `smooth.span` defines how much smoothing should be applied. To get a smooth transistion at the start of the graph:

``` r
summaryQ <- dailyQ %>%
  group_by(day.of.year) %>%
  summarize(p75 = quantile(rollMean, probs = .75, na.rm = TRUE),
            p25 = quantile(rollMean, probs = .25, na.rm = TRUE),
            p10 = quantile(rollMean, probs = 0.1, na.rm = TRUE),
            p05 = quantile(rollMean, probs = 0.05, na.rm = TRUE),
            p00 = quantile(rollMean, probs = 0, na.rm = TRUE)) 
summary.0 <- summaryQ %>%
  mutate(Date = as.Date(day.of.year - 1, 
                        origin = "2014-01-01"))

summary.1 <- summaryQ %>%
  mutate(Date = as.Date(day.of.year - 1, 
                        origin = "2015-01-01"))
summary.2 <- summaryQ %>%
  mutate(Date = as.Date(day.of.year - 1, 
                        origin = "2016-01-01"),
         day.of.year = day.of.year + 365)

summaryQ <- bind_rows(summary.0, summary.1, summary.2) 

smooth.span <- 0.3

summaryQ$sm.75 <- predict(loess(p75~day.of.year, 
                       data = summaryQ, 
                       span = smooth.span))
summaryQ$sm.25 <- predict(loess(p25~day.of.year, 
                       data = summaryQ, 
                       span = smooth.span))
summaryQ$sm.10 <- predict(loess(p10~day.of.year, 
                       data = summaryQ, 
                       span = smooth.span))
summaryQ$sm.05 <- predict(loess(p05~day.of.year, 
                       data = summaryQ, 
                       span = smooth.span))
summaryQ$sm.00 <- predict(loess(p00~day.of.year, 
                       data = summaryQ, 
                       span = smooth.span))

summaryQ <- select(summaryQ, Date, day.of.year,
                   sm.75, sm.25, sm.10, sm.05, sm.00) %>%
  filter(Date >= as.Date("2015-01-01"))

latest.years <- dailyQ %>%
  filter(Date >= as.Date("2015-01-01")) %>%
  mutate(day.of.year = 1:nrow(.))
```

Finally, create the graph using the `ggplot2` package. Base-R options are also possible. The bulk of the putzy work in this script is to get the graph style just right. The following script shows a very simple way to re-create the graph in `ggplot2` with no effort on imitating desired style.

``` r
library(ggplot2)
library(scales)

mid.month.days <- c(15, 45, 74, 105, 135, 166,
                    196, 227, 258, 288, 319, 349)

simple.plot <- ggplot(data = summaryQ, aes(x = day.of.year)) +
  geom_ribbon(aes(ymin = sm.25, ymax = sm.75, 
                  fill = "Normal")) +
  geom_ribbon(aes(ymin = sm.10, ymax = sm.25,
                  fill = "Drought Watch")) +
  geom_ribbon(aes(ymin = sm.05, ymax = sm.10, 
                  fill = "Drought Warning")) +
  geom_ribbon(aes(ymin = sm.00, ymax = sm.05, 
                  fill = "Drought Emergency")) +
  scale_y_log10(limits = c(1,1000)) +
  geom_line(data = latest.years, 
              aes(x=day.of.year, y=rollMean, color = "30-Day Mean"),size=2) +
  geom_vline(xintercept = 365) 

simple.plot
```

<img src='/static/moving-averages/unnamed-chunk-4-1.png'/ title='Simple 30-day moving average daily flow plot' alt='30-day moving average daily flow plot, no effort on style' class=''/>

Next, we can play with various options to do a better job:

``` r
styled.plot <- simple.plot+
  scale_x_continuous(breaks = c(mid.month.days,365+mid.month.days),
                   labels= rep(c("J","F","M","A","M",
                                 "J","J","A","S","O",
                                 "N","D"),2),
                   expand = c(0, 0),
                   limits = c(0,730)) +
  expand_limits(x = 0) +
  annotate(geom = "text", 
           x = c(182,547), 
           y = 1, 
           label = c("2015","2016"), size = 4) +
  theme_bw() + 
  theme(axis.ticks.x=element_blank(),
        panel.grid.major = element_blank(), 
        panel.grid.minor = element_blank()) +
  labs(list(title=paste0(stationInfo$station_nm,"\n",
                         "Provisional Data - Subject to change\n",
                         "Record Start = ", min(dailyQ$Date),
                         "  Number of years = ",
                         as.integer(as.numeric(difftime(time1 = max(dailyQ$Date), 
                                  time2 = min(dailyQ$Date),
                                  units = "weeks"))/52.25),
                         "\nDate of plot = ",Sys.Date(),
                         "  Drainage Area = ",stationInfo$drain_area_va, "mi^2"),
            y = "30-day moving ave", x="")) +
  scale_fill_manual(name="",breaks = c("Normal",
                             "Drought Watch",
                             "Drought Warning",
                             "Drought Emergency"),
                    values = c("red",
                               "orange",
                               "yellow",
                               "darkgreen")) +
  scale_color_manual(name = "", values = "black") +
   theme(legend.position="bottom")

styled.plot
```

<img src='/static/moving-averages/unnamed-chunk-5-1.png'/ title='Detailed 30-day moving average daily flow plot' alt='30-day moving average daily flow plot' class=''/>

Questions
=========

Please direct any questions or comments on `dataRetrieval` to: <https://github.com/USGS-R/dataRetrieval/issues>