---
title: "Economic and Social Consequences<br>of Atmospheric Events in the U.S.A."
author: "Felipe Campelo, Ph.D."
date: "December 18, 2014"
output: html_document
keep_md: TRUE
---



## Synopsis
Storm events are responsible by major impacts in American society, both in terms of economic consequences and harm to the health and life of the population. In this report we investigate the economic and human damages caused by different storm events, based on data available for the 1950-2011 period. From this data, we found that tornados tend to have the largest impact on human health (measured in terms of total number of people dead or injured) and the second largest impact on economical activity (measured in terms of property and crop damages). The largest economical consequences are those of floods, amounting to over 150 billion USD in damages over the 61 years investigated.

## Introduction
In this report, we seek to obtain answers for two questions of major economic and social importance:

- Across the United States, which types of events are most harmful with respect to population health?  
- Across the United States, which types of events have the greatest economic consequences?

To address these two questions we will make use of data available from public databases (see next section for details). For this analysis, we will employ a somewhat simplified notion of _damage to human health_ as the total number of people killed or injured by a given storm event; and of _economic consequences_ as the total property and crop damages caused by that event.

## Data Processing
The data used in this analysis refers to the 
[U.S. National Oceanic and Atmospheric Administration's (NOAA) storm database][1], 
for the years between 1950 and 2011. The [actual data file][2] (**Warning**: 
link to large download) was provided by the [Reproducible Research][3] course 
instructors.

Two other relevant files to this analysis are the 
[National Weather Service Storm Data Documentation][4] and the 
[National Climatic Data Center Storm Events FAQ][5], both also provided by the 
course instructors.

### Downloading and loading data

```r
# If the data file isn't already on the working folder, download it from the server
if(!file.exists("StormData.csv.bz2")){
    download.file(url = "https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2",
                  destfile = "StormData.csv.bz2",
                  method = "curl",
                  quiet = TRUE)
    }

# If the file isn't already loaded, load it.
# (go get yourself some coffee while you wait - this may take a while.)
library(dplyr,warn.conflicts = FALSE)
if(!exists("stormdata.original")){
    stormdata.original<-tbl_df(read.csv(bzfile("StormData.csv.bz2"),strip.white=TRUE))
    }
```

This is a large dataset, with almost a million observations of 37 variables 
related to storm events. We will only use a subset of these variables for our
analysis.

```r
# Copy data to a working variable
stormdata<-stormdata.original

# Dimension of the data frame
dim(stormdata)
```

```
## [1] 902297     37
```

```r
# Get variable names
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

### Isolating the variables of interest
The columns we are interested in are:

-----------     |  -----------------------------------------------
**Variable**    | **Description**
EVTYPE          | Type of event  
FATALITIES      | Number of fatalities related to the event  
INJURIES        | Number of injuries related to the event  
PROPDMG         | Property damage (number only)  
PROPDMGEXP      | Property damage multiplier character  
CROPDMG         | Crop damage (number only)  
CROPDMGEXP      | Crop damage multiplier character

$\ $  

To avoid biasing our analysis due to unreliable records, it is important 
to detect and treat observations that are inconsistent. In this analysis, we 
will employ the following exclusion criteria to the data:

- Exclude any observations with FATALITIES < 0, INJURIES < 0, PROPDMG < 0, or CROPDMG < 0;  
**Reason:** _The number of people killed in a storm event cannot be smaller than zero, nor can the property and crop damage caused by that event._  

- Exclude any observations with PROPDMGEXP $\neq\{k,K,m,M,b,B\}$ or CROPDMGEXP PROPDMGEXP $\neq\{k,K,m,M,b,B\}$;  
**Reason:** _The [National Weather Service Storm Data Documentation][4] (Section 2.7, paragraph 3, page 12) states that:_  

        "Alphabetical characters used to signify magnitude include “K” for thousands, “M” for millions, and “B” for billions."  

Thus, any observation with a multiplier character that differs from the specified ones will be considered as uninterpretable.

Before we proceed to the cleaning of the data, it is important to notice that relatively few valid observations will be removed:


```r
valid_exp<-c("k","K","m","M","b","B")
# How many observations will be dropped by our exclusion criteria?
(bycase.removals<-c(
    sum(!(stormdata$PROPDMGEXP %in% valid_exp)&(stormdata$PROPDMG>0)),      # Invalid PROPDMGEXP and PROPDMG > 0
    sum((stormdata$PROPDMGEXP %in% valid_exp)&(stormdata$PROPDMG<0)),       # Valid PROPDMGEXP and invalid PROPDMG
    sum(!(stormdata$CROPDMGEXP %in% valid_exp)&(stormdata$CROPDMG>0)),      # Invalid CROPDMGEXP and CROPDMG > 0
    sum((stormdata$CROPDMGEXP %in% valid_exp)&(stormdata$CROPDMG<0)),       # Valid CROPDMGEXP and invalid CROPDMG
    sum(stormdata$FATALITIES<0),                                            # Invalid FATALITIES
    sum(stormdata$INJURIES<0)))                                             # Invalid INJURIES
```

```
## [1] 327   0  15   0   0   0
```

There are 327 inconsistencies due to invalid PROPDMGEXP values and 15 due to invalid CROPDMGEXP values. Considering that the database contains over 900,000 observations, the proportion of invalid observations is negligible, and we choose to ignore those for the current analysis.

Here we extract the relevant columns and apply our exclusion criteria. Notice that since we observed no inconsistencies due to negative values, it is not necessary to apply the "$\geq 0$" filter to the numeric columns of interest.


```r
# Select columns and filter
stormdata<-stormdata %>%
    select(EVTYPE,FATALITIES,INJURIES,PROPDMG,PROPDMGEXP,CROPDMG,CROPDMGEXP) %>%
    filter(PROPDMGEXP %in% valid_exp | PROPDMG==0) %>%
    filter(CROPDMGEXP %in% valid_exp | CROPDMG==0) %>%
    mutate(EVTYPE=factor(EVTYPE))
```

### Cleaning up the damage values
It is also more convenient for the analysis if the damages are expressed in actual USD values, so that we can more easily summarize the total damages.


```r
stormdata$PROPDMGEXP<-factor(toupper(stormdata$PROPDMGEXP)) # uppercase all PROPDMGEXP values
stormdata$CROPDMGEXP<-factor(toupper(stormdata$CROPDMGEXP)) # uppercase all CROPDMGEXP values

# Convert the values to thousands, millions, billions of dollars
stormdata$PROPDMG[stormdata$PROPDMGEXP=="K"]<-10^3*stormdata$PROPDMG[stormdata$PROPDMGEXP=="K"]
stormdata$PROPDMG[stormdata$PROPDMGEXP=="M"]<-10^6*stormdata$PROPDMG[stormdata$PROPDMGEXP=="M"]
stormdata$PROPDMG[stormdata$PROPDMGEXP=="B"]<-10^9*stormdata$PROPDMG[stormdata$PROPDMGEXP=="B"]
stormdata$CROPDMG[stormdata$CROPDMGEXP=="K"]<-10^3*stormdata$CROPDMG[stormdata$CROPDMGEXP=="K"]
stormdata$CROPDMG[stormdata$CROPDMGEXP=="M"]<-10^6*stormdata$CROPDMG[stormdata$CROPDMGEXP=="M"]
stormdata$CROPDMG[stormdata$CROPDMGEXP=="B"]<-10^9*stormdata$CROPDMG[stormdata$CROPDMGEXP=="B"]

# Remove *EXP columns
stormdata<-mutate(stormdata,PROPDMGEXP=NULL,CROPDMGEXP=NULL)
```

### Cleaning up the EVTYPE column
The [National Weather Service Storm Data Documentation][4] (Section 2.1.1, Table I, page 6) recognizes 48 categories for storm events. These categories are entered here as a vector:


```r
catnames<-c(
"Astronomical Low Tide", "Avalanche", "Blizzard", "Coastal Flood",
"Cold", "Wind", "Chill", "Debris Flow", "Dense Fog",
"Dense Smoke", "Drought", "DustDevil", "DustStorm",
"ExcessiveHeat", "ExtremeCold", "WindChill", "FlashFlood",
"Flood", "Frost", "Freeze", "Funnel Cloud",
"Freezing Fog", "Hail", "Heat", "Heavy Rain",
"Heavy Snow", "High Surf", "High Wind", "Hurricane (Typhoon)",
"IceStorm", "Lake-Effect Snow", "Lakeshore Flood", "Lightning",
"Marine Hail", "Marine High Wind", "Marine Strong Wind", "Marine Thunderstorm Wind",
"Rip Current", "Seiche", "Sleet", "Storm Surge",
"Tide", "Strong Wind", "Thunderstorm Wind", "Tornado",
"Tropical Depression", "Tropical Storm", "Tsunami", "Volcanic Ash",
"Waterspout", "Wildfire", "Winter Storm", "Winter Weather")
```

However, a quick examination of the EVTYPE column of the dataset shows that there are many more ways entries are registered:


```r
length(unique(stormdata$EVTYPE))
```

```
## [1] 982
```

This multitude of labels is possibly due to the long time span of the records, with entries as far back as 1950, evolving standards and multiple people entering records both digitally and on paper. 

First, lets isolate the observations that do not perfectly match one of the official categories. To avoid losing observations due to trivial typos, we can perform a few operations prior to the matching:  

- remove all white spaces from the event names, both in our reference vector and on the EVTYPE variable;  

- change all words to upper-case, both in our reference vector and on the EVTYPE variable;


```r
# define function to remove all white spaces from a character string
remspaces<-function(x){toupper(gsub("\\s","",x))}

# Remove spaces and transform to uppercase: catnames and EVTYPE
catnames<-sapply(catnames,remspaces)
stormdata$EVTYPE<-factor(sapply(stormdata$EVTYPE,remspaces))
```

Now lets isolate the observations that do not perfectly match one of the categories defined above, and check which proportion of the data is incorrectly labeled.


```r
# Isolate non-matching event types
stormdata.nomatch<-filter(stormdata,!(EVTYPE %in% catnames))

# What proportion of the data would we lose if we just discarded the non-matching observations?
dim(stormdata.nomatch)[1]/dim(stormdata)[1]
```

```
## [1] 0.2973873
```

Discarding the non-matching observations would lead to the loss of almost 30% of all observations, which could clearly affect our analysis. To try to recover a few of these observations, we can start by finding which entries are most common in the non-matching group. The first 10 are:

```r
nmtch.tbl<-sort(table(stormdata.nomatch$EVTYPE),decreasing = T)
print(nmtch.tbl[1:10])
```

```
## 
##              TSTMWIND     THUNDERSTORMWINDS        MARINETSTMWIND 
##                219945                 20633                  6175 
##    URBAN/SMLSTREAMFLD             HIGHWINDS       WILD/FORESTFIRE 
##                  3392                  1530                  1457 
##          FROST/FREEZE     WINTERWEATHER/MIX         TSTMWIND/HAIL 
##                  1343                  1104                  1028 
## EXTREMECOLD/WINDCHILL 
##                  1002
```

The large majority of non-matching names are due to relatively few labels, possibly a legacy from a previous standard. Just by consolidating the ten most frequent labels from the non-matching group, we would reduce the amount of lost data by:


```r
sum(as.numeric(nmtch.tbl[1:10]))/dim(stormdata.nomatch)[1]
```

```
## [1] 0.9604034
```

That's over 96% of the mis-labeled data being recovered. Lets do it:


```r
# Correct labels for the 10 most common occurrences of mislabeling
rightlabels10<-c("THUNDERSTORMWIND","THUNDERSTORMWIND",
                 "MARINETHUNDERSTORMWIND","FLOOD",
                 "HIGHWIND","WILDFIRE",
                 "FREEZE","WINTERWEATHER",
                 "THUNDERSTORMWIND","COLD")

# Fix 10 most common wrong labels 
for (i in 1:10){
    mislabel<-names(nmtch.tbl[i])
    stormdata$EVTYPE[which(stormdata$EVTYPE==mislabel[i])]<-rightlabels10[i]
}

# Isolate the remaining mislabeled observations
stormdata.nomatch<-filter(stormdata,!(EVTYPE %in% catnames))

# Proportion of still-mislabeled data
dim(stormdata.nomatch)[1]/dim(stormdata)[1]
```

```
## [1] 0.05353371
```

Now we have only about 5% of the observations being mislabeled. We can recover a few more just using a few simple operations:


```r
# Relabel events that simply have a letter "S" added after a valid label (e.g., "FORESTFIRES" instead of "FORESTFIRE").
stormdata$EVTYPE<-as.character(stormdata$EVTYPE)
for (i in 1:length(catnames)){
    stormdata$EVTYPE[which(stormdata$EVTYPE==paste0(catnames[i],"S"))]<-catnames[i]
}

# Remove events that are simply given as "summaries" - there are a few:
length(grep("summary",
            stormdata$EVTYPE,
            ignore.case = TRUE))
```

```
## [1] 76
```

```r
stormdata<-filter(stormdata,
                  !grepl("summary",
                         stormdata$EVTYPE,
                         ignore.case = TRUE))

# Relabel mislabeled hurricanes:
stormdata$EVTYPE[grep("hurricane",
                      stormdata$EVTYPE,
                      ignore.case = TRUE)]<-"HURRICANE(TYPHOON)"

# Again, isolate the remaining mislabeled observations
stormdata.nomatch<-filter(stormdata,!(EVTYPE %in% catnames))

# Proportion of still-mislabeled data
dim(stormdata.nomatch)[1]/dim(stormdata)[1]     
```

```
## [1] 0.02773432
```

Only 2.8% of the data is still mislabeled. Before we discard these completely, lets just check how much damage (property and crop) and how many casualties (fatalities/injuries) are still contained in the unlabeled data:


```r
# Absolute values
damage<-sum(stormdata.nomatch$PROPDMG)+sum(stormdata.nomatch$CROPDMG)
casualties<-sum(stormdata.nomatch$FATALITIES)+sum(stormdata.nomatch$INJURIES)

# Proportions
damage/(sum(stormdata$PROPDMG)+sum(stormdata$CROPDMG))
```

```
## [1] 0.05995419
```

```r
casualties/(sum(stormdata$FATALITIES)+sum(stormdata$INJURIES))
```

```
## [1] 0.02947299
```

Only about 3% of the casualties and about 6% of the total damage are contained in the unlabeled data, so we probably don't have to worry too much about discarding it - we have already passed the point of diminishing returns for our efforts into relabeling the EVTYPE variable, and will proceed using only those observations that are already correctly labeled. 


```r
stormdata<-filter(stormdata,(EVTYPE %in% catnames))
```

### Summarizing the data set
Finally, before we proceed to present our results, we can summarize the total amount of damage (both in terms of property and of human health) by event type.


```r
# Include columns for total damages and casualties and summarize results (in terms of these two new columns) by type of event.
stormdata<-stormdata %>%
    mutate(TotalDamage=CROPDMG+PROPDMG,
           TotalCasualties=FATALITIES+INJURIES) %>%
    group_by(EVTYPE) %>%
    summarize(Damages=sum(TotalDamage),
              Casualties=sum(TotalCasualties))
```

## Results
After tidying up, cleaning, and summarizing the available data, it is now possible to answer the two questions posed at the beginning of this report:

        Across the United States, which types of events are most harmful with respect to population health?

To answer this particular question, lets arrange the summarized dataset obtained in the previous section by the total number of casualties by event, and investigate graphically the cumulative health effects (again, defined as the total number of people dead or injured) of each event.


```r
# arrange by decreasing number of casualties
stormdata<-arrange(stormdata,desc(Casualties))

# Generate plot of casualties per event
library(ggplot2,quietly = TRUE,warn.conflicts = FALSE)
m<-ggplot(stormdata,aes(EVTYPE,Casualties))
m + geom_bar(stat="identity") + 
    scale_x_discrete(limits=stormdata$EVTYPE) +
    ggtitle("Total Casualties (1950-2011) by Event Type") +
    xlab("Event Type") + 
    ylab("Casualties") +
    annotate("rect",
             xmin = 2, xmax = 21.5, 
             ymin = 85000, ymax = max(stormdata$Casualties),
             alpha = .2) +
    annotate("text",
             x = c(2.25,2.25), 
             y = c(94000,89000), 
             label = c("Event type with most casualties: TORNADO", 
                       paste(max(stormdata$Casualties),"casualties")),
             hjust=0,
             fontface="bold") + 
    theme(axis.title.y = element_text(size = rel(1.25), angle = 90),
          axis.title.x = element_text(size = rel(1.25), angle = 0),
          axis.text.x = element_text(size = rel(.85),angle=45,hjust=1),
          axis.text.y = element_text(angle=0),
          plot.title = element_text(size = rel(1.5)))
```

<img src="figure/healthdamage-1.png" title="plot of chunk healthdamage" alt="plot of chunk healthdamage" style="display: block; margin: auto;" />

By far, tornados are the most damaging event (in terms of deaths and injuries) when integrated over time, surpassing all other events combined in terms of total number of casualties in the period under study.

        Across the United States, which types of events have the greatest economic consequences?

Similarly to what was done for the previous question, lets arrange the dataset by the total damage caused, and investigate graphically the cumulative economic consequences (property and crop damage) of each event.


```r
# arrange by decreasing damages
stormdata<-arrange(stormdata,desc(Damages))

# Generate plot of casualties per event
library(ggplot2,quietly = TRUE,warn.conflicts = FALSE)
m<-ggplot(stormdata,aes(EVTYPE,Damages/1e9))
m + geom_bar(stat="identity") + 
    scale_x_discrete(limits=stormdata$EVTYPE) +
    ggtitle("Total Damages (1950-2011) by Event Type") +
    xlab("Event Type") + 
    ylab("Damages (Billions USD)") +
     annotate("rect",
              xmin = 2, xmax = 21.5, 
              ymin = 130, ymax = max(stormdata$Damages/1e9),
              alpha = .2) +
     annotate("text",
              x = c(2.25,2.25), 
              y = c(145,135), 
              label = c("Event type with largest damages: FLOOD", 
                        paste("USD",round(max(stormdata$Damages/1e9),1),"billion")),
              hjust=0,
              fontface="bold") + 
    theme(axis.title.y = element_text(size = rel(1.25), angle = 90),
          axis.title.x = element_text(size = rel(1.25), angle = 0),
          axis.text.x = element_text(size = rel(.85),angle=45,hjust=1),
          axis.text.y = element_text(angle=0),
          plot.title = element_text(size = rel(1.5)))
```

<img src="figure/economicdamage-1.png" title="plot of chunk economicdamage" alt="plot of chunk economicdamage" style="display: block; margin: auto;" />

Floods are the most damaging event (in terms of economic losses) when integrated over time. Tornados, which hold the first place in terms of casualties, also figure prominently as the second most economically damaging event in the 1950-2011 period, with over 50 billion dollars in property and crop damages.

[1]: http://www.ncdc.noaa.gov/stormevents/ "NOAA Storm Events Database"
[2]: https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2 "Data file"
[3]: https://www.coursera.org/course/repdata "Reproducible Research"
[4]: https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf "Documentation"
[5]: https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2FNCDC%20Storm%20Events-FAQ%20Page.pdf "FAQ"
