Intro to Cross-National Time-Series Data for International Relations and Comparative Politics
========================================================
Gather data from the World Development Indicators, Penn World Tables, and Polity IV. Merge datasets, interpolate variables, visualize, model, and generate automated reports in Latex.


## The World Development Indicators API in R

1. WDI are data collected by the World Bank
2. Hundreds of state-level variables
3. From 1960 to 2012
4. Import to R directly


```r
# install.packages('WDI')
require("WDI")
```

* To find the variable codes to call, first search with the ```WDIsearch()``` function.
* Then call data with the ```WDI()``` function.
* Super easy!

## Simple Demonstration

* Let's say we're interested in GDP growth across countries. First we'll search for "gdp growth" to see if the World Bank has a variable matching this search term.


```r
### Use this function to find indicator codes for things that interest you
WDIsearch("gdp growth")
```

```
##      indicator           name                                    
## [1,] "NV.AGR.TOTL.ZG"    "Real agricultural GDP growth rates (%)"
## [2,] "NY.GDP.MKTP.KD.ZG" "GDP growth (annual %)"
```


* To download the data straight from the web, we use the ```WDI()``` function, specifying which countries, variables, and years we're interested in.

```r
### Insert as many countries and variables as you want, within this range.
wb <- WDI(country = c("US", "FR"), indicator = c("NY.GDP.MKTP.KD.ZG"), start = 1960, 
    end = 2012)
```



## Now Let's Make a Real Dataset!

- Download a set of commonly used variables for all countries between 1960 and the most recent update of the WDI. Setting ```extra=TRUE``` retrieves some extra variables such as region, geocodes, and additional country codes. The additional country code turns out to merge better with other datasets (see below).


```r
wb <- WDI(country = "all", indicator = c("SE.XPD.TOTL.GB.ZS", "GC.DOD.TOTL.GD.ZS", 
    "BN.CAB.XOKA.GD.ZS", "UIS.XGOVEXP.FNCUR", "SP.POP.DPND", "AG.LND.TOTL.K2", 
    "NY.GDP.MKTP.CD", "NY.GDP.MKTP.PP.CD", "NY.GDP.MKTP.KD", "NY.GDP.PCAP.CD", 
    "SP.POP.TOTL", "IT.PRT.NEWS.P3", "IT.RAD.SETS", "NE.IMP.GNFS.ZS", "NE.EXP.GNFS.ZS", 
    "NE.TRD.GNFS.ZS", "BX.KLT.DINV.WD.GD.ZS", "BN.KLT.PRVT.GD.ZS", "NE.CON.GOVT.ZS", 
    "SL.IND.EMPL.ZS", "NV.IND.TOTL.ZS", "NY.GDP.MKTP.KD.ZG", "NY.GDP.PCAP.KD", 
    "GDPPCKD"), start = 1960, end = 2012, extra = TRUE)
```


## Clean the data

```r
wb$debt <- wb$GC.DOD.TOTL.GD.ZS  # central government debt as share of gdp
wb$curracct <- wb$BN.CAB.XOKA.GD.ZS  # current account balance as share of gdp
wb$edu <- wb$UIS.XGOVEXP.FNCUR  # public education spending as share of total govt spending
wb$dependency <- wb$SP.POP.DPND  # dependency-age population as share of total population
wb$land <- wb$AG.LND.TOTL.K2  # total land in square kilometers
wb$pop <- wb$SP.POP.TOTL  # total population
wb$radioscap <- (wb$IT.RAD.SETS/wb$pop) * 1000  # radios per 1000 people
wb$newspaperscap <- wb$IT.PRT.NEWS.P3  # newspapers in circulation per 1000 people
wb$gdp <- wb$NY.GDP.MKTP.KD  # gross domestic product, constant 2000 USD
wb$gdp2 <- wb$NY.GDP.MKTP.CD  # gross domestic product, current USD, millions
wb$gdp3 <- wb$NY.GDP.MKTP.PP.CD  # gross domestic product, purchasing-power parity, current
wb$gdpcap <- wb$NY.GDP.PCAP.KD  # gdp per capita constant 2000 USD
wb$gdpcap2 <- wb$GDPPCKD  # gross domestic product per capita
wb$gdpgrowth <- wb$NY.GDP.MKTP.KD.ZG  # change in gross domestic product
wb$imports <- wb$NE.IMP.GNFS.ZS  # imports of goods and services as share of gdp
wb$exports <- wb$NE.EXP.GNFS.ZS  # exports of goods and services as share of gdp
wb$trade <- wb$NE.TRD.GNFS.ZS  # total trade as share of gdp
wb$fdi <- wb$BX.KLT.DINV.WD.GD.ZS  # foreign-direct investment as share of gdp
wb$privatecapital <- wb$BN.KLT.PRVT.GD.ZS  # private capital flows as share of gdp
wb$spending <- wb$NE.CON.GOVT.ZS  # government consumption expenditure as share of gdp
wb$industry <- wb$SL.IND.EMPL.ZS  # employment in industry as share of total employment
wb$industry2 <- wb$NV.IND.TOTL.ZS  # value added in industry as share of gdp
```

## Subsetting Data

Take a peak at the "head" (the first six rows) using the ```head()``` function. This is "panel" or "pooled" cross-sectional, time-series data. This is also called "country-year" format.

```r
head(wb[, 1:7])  # limit to columns 1 through 7
```

```
##   iso2c country year SE.XPD.TOTL.GB.ZS GC.DOD.TOTL.GD.ZS BN.CAB.XOKA.GD.ZS
## 1        Africa 1960                NA                NA                NA
## 2        Africa 1961                NA                NA                NA
## 3        Africa 1962                NA                NA                NA
## 4        Africa 1963                NA                NA                NA
## 5        Africa 1964                NA                NA                NA
## 6        Africa 1965                NA                NA                NA
##   UIS.XGOVEXP.FNCUR
## 1                NA
## 2                NA
## 3                NA
## 4                NA
## 5                NA
## 6                NA
```

The World Bank keeps data not only on countries, but on certain aggregates of countries. Aggregate units are distinguished by the category "Aggregates" in the factor variable "region" in the ```wb``` object (a dataframe). Then, each of the different aggregates are specified by the "country" variable. To keep things tidy, let's break the ```wb``` dataframe into a few separate dataframes. We'll make one for all the aggregates, one for regions in particular (or any other set of aggregates you're interested in; try replacing the region categories below with other categories in the "country" variable of the ```aggregate``` subset we're about to make...), and one for the good-old fashioned states.

```r

aggregates <- subset(wb, region == "Aggregates")

world <- subset(aggregates, country == "World")

regions <- subset(aggregates, country == "East Asia & Pacific (all income levels)" | 
    country == "Middle East & North Africa (all income levels)" | country == 
    "Sub-Saharan Africa (all income levels)" | country == "North America" | 
    country == "Latin America & Caribbean (all income levels)" | country == 
    "Europe & Central Asia (all income levels)" | country == "South Asia")

wb <- subset(wb, region != "Aggregates")
```

## Visualizing Panel Data
Below we investigate our data using one of the most powerful graphing packages in R: ggplot2. The formulas below are a quick tour of plots suited especially for the demands of panel data.

### Multiple Variables (On Single Scale) for One Unit Over Time

```r
require("ggplot2")

ggplot(world, aes(x = year)) + geom_line(aes(y = trade, colour = "Trade")) + 
    geom_line(aes(y = spending, colour = "Spending")) + scale_colour_discrete(name = "Variables") + 
    xlab("Year") + ylab("% of GDP")
```

![plot of chunk unnamed-chunk-8](figure/unnamed-chunk-8.png) 

```r
ggtitle("Total Trade and Government Spending Around the World, 1960-2012")
```

```
## $title
## [1] "Total Trade and Government Spending Around the World, 1960-2012"
## 
## attr(,"class")
## [1] "labels"
```


### One Time-Series Plot for One Variable and One Line per Unit

```r
ggplot(regions, aes(x = year, y = gdpcap, colour = country)) + geom_line() + 
    scale_colour_discrete(name = "Regions") + xlab("Year") + ylab("GDP Per Capita") + 
    ggtitle("GDP Per Capita by Region, 1960-2012")
```

![plot of chunk unnamed-chunk-9](figure/unnamed-chunk-9.png) 

### Panel of Time-Series Plots for One Variable and One Plot per Unit

```r
ggplot(wb[wb$region == "Europe & Central Asia (all income levels)", ], aes(x = year, 
    y = trade)) + geom_line(aes(group = country)) + facet_wrap(~country) + xlab("Year") + 
    ylab("Trade (% of GDP)") + ggtitle("Trade Levels in Europe and Central Asia, 1960-2012")
```

![plot of chunk unnamed-chunk-10](figure/unnamed-chunk-10.png) 


## Import Polity IV

```r
setwd("~/Dropbox/Data General/Polity IV")
polity <- read.csv("http://introquant.com/polityiv.csv")  # read in the Polity IV dataset
head(polity[, 1:10])
```

```
##     cyear ccode scode     country year flag fragment democ autoc polity
## 1 7001800   700   AFG Afghanistan 1800    0       NA     1     7     -6
## 2 7001801   700   AFG Afghanistan 1801    0       NA     1     7     -6
## 3 7001802   700   AFG Afghanistan 1802    0       NA     1     7     -6
## 4 7001803   700   AFG Afghanistan 1803    0       NA     1     7     -6
## 5 7001804   700   AFG Afghanistan 1804    0       NA     1     7     -6
## 6 7001805   700   AFG Afghanistan 1805    0       NA     1     7     -6
```

## Easily deal with country codes using the ```countrycode``` package

```r
require("countrycode")
polity$iso3c <- countrycode(polity$ccode, "cown", "iso3c")  # convert COW code to ISO 3-character code as in WDI
```

## Subset Variables of Interest and Merge with WDI

```r
dem <- subset(polity, select = c("iso3c", "year", "democ", "autoc", "polity2"))  # make subset of key variables
df <- merge(wb, dem, by = c("iso3c", "year"))  # 7,161 country-years matched, yay
```

## Standardize Time-Series on Different Scales

```r
require(arm)
df$dem.z <- rescale(df$polity2)  # subtract out the mean and divide by two standard deviations
df$trade.z <- rescale(df$trade)
```

## Visualize Standardized Time-Series By Country

```r
ggplot(df[df$region == "Middle East & North Africa (all income levels)", ], 
    aes(x = year)) + geom_line(aes(y = trade.z, group = country, colour = "Trade")) + 
    geom_line(aes(y = dem.z, group = country, colour = "Polity IV")) + facet_wrap(~country) + 
    scale_colour_discrete(name = "Standardized Variables") + xlab("Year") + 
    ylab("Standardized deviations from mean") + ggtitle("Trade & Democracy in Middle East & North Africa, 1960-2012")
```

![plot of chunk unnamed-chunk-15](figure/unnamed-chunk-15.png) 


## Modeling pooled data

```r
require(plm)
model <- plm(spending ~ trade + polity2 + dependency + gdpgrowth + lag(spending), 
    index = c("iso3c", "year"), model = "within", effect = "twoways", data = df)
```

```
## series GDPPCKD,gdpcap2 are NA and have been removed
```

```r
ls(model)
```

```
## [1] "args"         "call"         "coefficients" "df.residual" 
## [5] "formula"      "model"        "residuals"    "vcov"
```

```r
summary(model)
```

```
## Twoways effects Within Model
## 
## Call:
## plm(formula = spending ~ trade + polity2 + dependency + gdpgrowth + 
##     lag(spending), data = df, effect = "twoways", model = "within", 
##     index = c("iso3c", "year"))
## 
## Unbalanced Panel: n=154, T=3-51, N=5777
## 
## Residuals :
##    Min. 1st Qu.  Median 3rd Qu.    Max. 
## -32.400  -0.730  -0.058   0.640  23.500 
## 
## Coefficients :
##               Estimate Std. Error t-value Pr(>|t|)    
## trade          0.00532    0.00170    3.12   0.0018 ** 
## polity2        0.01955    0.00781    2.50   0.0123 *  
## dependency     0.00450    0.00393    1.15   0.2519    
## gdpgrowth     -0.06130    0.00534  -11.47   <2e-16 ***
## lag(spending)  0.83046    0.00703  118.13   <2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1 
## 
## Total Sum of Squares:    90200
## Residual Sum of Squares: 24900
## R-Squared      :  0.724 
##       Adj. R-Squared :  0.697 
## F-statistic: 2916.2 on 5 and 5568 DF, p-value: <2e-16
```

## Panel-Corrected Standard Errors

```r
require(lmtest)
coeftest(model, vcov = function(x) vcovBK(x, type = "HC1", cluster = "time"))
```

```
## 
## t test of coefficients:
## 
##               Estimate Std. Error t value Pr(>|t|)    
## trade          0.00532    0.00232    2.29    0.022 *  
## polity2        0.01955    0.00985    1.98    0.047 *  
## dependency     0.00450    0.00377    1.20    0.232    
## gdpgrowth     -0.06130    0.00792   -7.74  1.2e-14 ***
## lag(spending)  0.83046    0.01871   44.39  < 2e-16 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
```

## Generate Summary Tables

```r
require(reporttools)
tableContinuous(modvars[, sapply(modvars, is.numeric)], font.size = 12)
tableNominal(modvars[, !sapply(modvars, is.numeric)], font.size = 12)
```

## Generate Publication-Quality Results Tables

```r
require(apsrtable)
model <- lm(spending ~ trade + polity2 + dependency + gdpgrowth + lag(spending), 
    data = df)
writeLines(apsrtable(model), file("model.tex"))
```

## Save Your Dataset to a File

```r
# set your working directory to whatever folder you wish
write.csv(df, "cstsdata.csv")
```

