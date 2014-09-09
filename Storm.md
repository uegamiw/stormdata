# RepDataPA2
Wegamoon  
September 7, 2014  
#Download and preparement of data.

### Data

Before starting the analysis, download the file form [here](https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2). For more information
about this dataset see:

- National Weather Service [Storm Data Documentation][1]

- National Climatic Data Center Storm Events [FAQ][2]

[1]: https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf
[2]: https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2FNCDC%20Storm%20Events-FAQ%20Page.pdf


This file is compressed as a bzip2 file, then expand it on your working
directory and rename it to "stormData.csv"

### Data Processing

Read the storm data to a R. This process may take some second. 
Please be patient!


```r
if(!file.exists("stormData.csv.bz2")) {
        fileUrl <- "https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2"
        download.file(url = fileUrl, destfile = "stormData.csv.bz2", method = "curl")
}
stormdata <- read.csv(file = "stormData.csv.bz2")
```

There are {r ncol(stormdata)} fields in this file.


```r
names(stormdata)
```

```
##  [1] "STATE__"    "BGN_DATE"   "BGN_TIME"   "TIME_ZONE"  "COUNTY"    
##  [6] "COUNTYNAME" "STATE"      "EVTYPE"     "BGN_RANGE"  "BGN_AZI"   
## [11] "BGN_LOCATI" "END_DATE"   "END_TIME"   "COUNTY_END" "COUNTYENDN"
## [16] "END_RANGE"  "END_AZI"    "END_LOCATI" "LENGTH"     "WIDTH"     
## [21] "F"          "MAG"        "FATALITIES" "INJURIES"   "PROPDMG"   
## [26] "PROPDMGEXP" "CROPDMG"    "CROPDMGEXP" "WFO"        "STATEOFFIC"
## [31] "ZONENAMES"  "LATITUDE"   "LONGITUDE"  "LATITUDE_E" "LONGITUDE_"
## [36] "REMARKS"    "REFNUM"
```

The data from 2000 to present was extracted because there are too many 
variations in `EVTYPE` (the name of the types) field. 

- [Database datail](http://www.ncdc.noaa.gov/stormevents/details.jsp?type=collection)


```r
library(lubridate)
stormdata$BGN_DATE <- mdy_hms(stormdata$BGN_DATE) # Convert to POSIXct format
stormdata <- stormdata[stormdata$BGN_DATE >= "2000-01-01",] # Subsetting
```


# Q1) Which type of events are most harmful with respent to population health?


The following fields are related in this analysis.

- `EVNAME`: name of the event

- `FATALITIES`: the number of the fatalities in its recode.

- `INJURIES`: the number of the injuries in its recode.


First, the sum of the fatalities and injuries in each event name was calculated.



```r
library(plyr)
healthsum <- ddply(stormdata, .(EVTYPE), summarise,
                fatalities = sum(FATALITIES),
                injuries = sum(INJURIES))
```

These data frame are reorderd by the number of fatalities and injuries,
and the top 20 and the sum of other cases are extracted.

The order of factor are arranged by the number of injuries and fatalities.


```r
library(plyr)
injOrd <- arrange(healthsum, desc(injuries))[1:20,c(1,3)]
injOrd$EVTYPE <- factor(injOrd$EVTYPE,
                        levels = injOrd[order(injOrd$injuries),"EVTYPE"])

fataOrd <- arrange(healthsum, desc(fatalities))[1:20, c(1,2)]
fataOrd$EVTYPE <- factor(fataOrd$EVTYPE,
                        levels = fataOrd[order(fataOrd$fatalities),"EVTYPE"])
```

## Result(1)
The number of injuries and fatalities from 2000 through 2011.


```r
library(ggplot2)
injOrd$injuries <- as.numeric(injOrd$injuries)
fataOrd$fatalities <- as.numeric(fataOrd$fatalities)

g1 <- ggplot(injOrd, aes(x = EVTYPE, y = injuries)) +
        geom_bar(stat = "identity") +
        ggtitle("INJURIES")+
        ylab(NULL) +
        xlab(NULL) +
        coord_flip()

g2 <- ggplot(fataOrd, aes(x = EVTYPE, y = fatalities)) +
        geom_bar(stat = "identity") +
        ggtitle("FATALITIES")+
        ylab(NULL) + 
        xlab(NULL) +
        coord_flip()

library(grid)
grid.newpage()
pushViewport(viewport(layout=grid.layout(1, 2)))
print(g1, vp=viewport(layout.pos.row=1, layout.pos.col=1))
print(g2, vp=viewport(layout.pos.row=1, layout.pos.col=2))
```

![plot of chunk plotting](./Storm_files/figure-html/plotting.png) 

Tornadoes outnumbers the other cases in both injuries and fatalities. 
The most harmful event seems to be a tornadoes.
Although the number of injuries because of ECESSIVE HEAT is relatively
small comparing to that of TORNADO, it may ealily lead to death.

# Q2) Which type of events have the greatest economic consequence?

I'm going to compare the amount of "crop damage" and "property damage" among
each types of events.
Following culumns are associated in this analysis.

- `EVTYPE` : event type

- `CROPDMG` : crop damage

- `CROPDMGEXP` : crop damage exponent

- `PROPDMG` : property damage

- `PROPDMGEXP` : property damage exponent

The coloumn `CROPDMGEXP` and `PROPDMGEXP` contain several factors, but most of
them are not used. Then `K`, `M`, `B` seems to mean "thousand", "million",
"billion".


```r
summary(stormdata$CROPDMGEXP)
```

```
##             0      2      ?      B      K      M      k      m 
## 250613      0      0      0      4 271351   1195      0      0
```

```r
summary(stormdata$PROPDMGEXP)
```

```
##             +      -      0      1      2      3      4      5      6 
## 189121      0      0      1      0      0      0      0      0      0 
##      7      8      ?      B      H      K      M      h      m 
##      0      0      0     29      0 328461   5551      0      0
```

For example, a record whose `CRPDMG` field has `12` and `CRPDMGEXP` field has
`M` says that the amount of crop damage is 12,000,000. 

In order to change this notation into real number,
`K`, `M` and `B` in exponent fields are replaced by character
`1`, `1000`, `1000000`, then `CRPDMG` and `PROPDMG` fields are replaced
by absolute amount of damage. (unit: million) 

Although a number of records has lack blank in `CROPDMGEXP` and `PROPDMGEXP`
field, these records are ingnored as `NA`s in this analysis.


```r
stormdata$CROPDMGEXP <- gsub("K", "0.001", stormdata$CROPDMGEXP)
stormdata$CROPDMGEXP <- gsub("M", "1", stormdata$CROPDMGEXP)
stormdata$CROPDMGEXP <- gsub("B", "1000", stormdata$CROPDMGEXP)

stormdata$PROPDMGEXP <- gsub("K", "0.001", stormdata$PROPDMGEXP)
stormdata$PROPDMGEXP <- gsub("M", "1", stormdata$PROPDMGEXP)
stormdata$PROPDMGEXP <- gsub("B", "1000", stormdata$PROPDMGEXP)

stormdata$CROPDMGEXP <- as.numeric(stormdata$CROPDMGEXP)
stormdata$PROPDMGEXP <- as.numeric(stormdata$PROPDMGEXP)

stormdata$CROPDMG <- stormdata$CROPDMG * stormdata$CROPDMGEXP
stormdata$PROPDMG <- stormdata$PROPDMG * stormdata$PROPDMGEXP
```

The amount of total damage in property and crop in each event type are
calculated.


```r
library(plyr)
damage <- ddply(stormdata, .(EVTYPE), summarise,
                crop = sum(CROPDMG, na.rm = TRUE),
                prop = sum(PROPDMG, na.rm = TRUE))
```

Total amount of damage was ...


```r
total <- damage$crop + damage$prop
damage <- cbind(damage, total)

damage <- arrange(damage, desc(total))[1:20,]

damage$EVTYPE <- factor(damage$EVTYPE,
                        levels = damage[order(damage$total),"EVTYPE"])
```

## Result (2)


```r
library(reshape2)
meltdamage <- melt(damage, id="EVTYPE", measure.vars = c("crop", "prop"))
meltdamage$value <- meltdamage$value/1000

library(ggplot2)
ggplot(meltdamage, aes(x = EVTYPE, y = value, fill = variable)) +
        geom_bar(stat = "identity") +
        ggtitle("Damage") +
        ylab("unit: billions") +
        xlab(NULL) +
        coord_flip() 
```

![plot of chunk plottingDMG](./Storm_files/figure-html/plottingDMG.png) 
