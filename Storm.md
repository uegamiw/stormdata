# RepDataPA2
Wegamoon  
September 7, 2014  
#Download and preparement of data.

## Data Download.

Before starting the analysis, download the file form [here](https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2). For more information
about this dataset see:

- National Weather Service [Storm Data Documentation][1]

- National Climatic Data Center Storm Events [FAQ][2]

[1]: https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf
[2]: https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2FNCDC%20Storm%20Events-FAQ%20Page.pdf


This file is compressed as a bzip2 file, then expand it on your working
directory and rename it to "stormData.csv"

## Read the file.

Read the storm data to a R. This process may take some second. 
Please be patient!

```r
stormdata <- read.csv(file = "stormData.csv")
```

## Which type of events are most harmful?
This database report the number of fatalities and injuries in each event.
First, the sum of the fatalities and injuries in each event name was calculated.


```r
library(plyr)
healthsum <- ddply(stormdata, .(EVTYPE), summarise,
                fatalities = sum(FATALITIES),
                injuries = sum(INJURIES))
```

These data frame are reorderd by the number of fatalities and injuries,
and the top ten and the sum of other cases are extracted.

```r
library(plyr)
fatalOrd <- arrange(healthsum, desc(fatalities))[,1:2]
injOrd <- arrange(healthsum, desc(injuries))[,c(1, 3)]
otherFat <- c("Other", sum(fatalOrd$fatalities[11:nrow(fatalOrd)]))

top10fatal <- rbind(fatalOrd[1:10,], otherFat)
otherInj <- c("Other", sum(injOrd$injuries[11:nrow(injOrd)]))
top10injury <- rbind(injOrd[1:10,], otherInj)
```


```r
library(ggplot2)
library(reshape2)
combined <- rbind(melt(top10fatal, id= "EVTYPE"),
                 melt(top10injury, id = "EVTYPE")
                 )
combined$value <- as.numeric(combined$value)
ggplot(combined, aes(x = EVTYPE, y = value))+
        geom_bar(stat="identity")+
        facet_grid(~variable)
```

![plot of chunk plotting](./Storm_files/figure-html/plotting.png) 

The number of injuries and fatalities of tolnado is far more than other.


- How to arrange the order of factors?
- It might be better to merge the facet and colored.
