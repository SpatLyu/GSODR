Fetch, Clean and Correct Altitude in GSOD ‘isd\_history.csv’ Data
================
Adam H. Sparks
2020-04-17

# Introduction

The isd\_history file details station metadata including the start and
stop years used by GSODR to pre-check requests before querying the
server for download and the country code used by GSODR when subsetting
for requests by country. The following changes are made to the raw data
file for inclusion in *GSODR*:

  - isd\_history where latitude or longitude are `NA` or both 0 are
    removed

  - isd\_history where latitude is \< -90˚ or \> 90˚ are removed

  - isd\_history where longitude is \< -180˚ or \> 180˚ are removed

  - A new field, STNID, a concatenation of the USAF and WBAN fields, is
    added

# Data Processing

## Set up workspace

``` r
if (!require("sessioninfo")) {
  install.packages("sessioninfo", repos = "https://cran.rstudio.com/")
}

if (!require("skimr")) {
  install.packages("skimr", repos = "https://cran.rstudio.com/")
}

if (!require("countrycode"))
{
  install.packages("countrycode",
                   repos = c(CRAN = "https://cran.rstudio.com"))
}

if (!require("data.table")) {
  install.packages("data.table", repos = "https://cran.rstudio.com/")
}
```

## Download and clean data

``` r
# download data
isd_history <- fread("https://www1.ncdc.noaa.gov/pub/data/noaa/isd-history.csv")
```

## Add/drop columns and save to disk

``` r
# add STNID column
isd_history[, STNID := paste(USAF, WBAN, sep = "-")]
setcolorder(isd_history, "STNID")
setnames(isd_history, "STATION NAME", "NAME")

# drop stations not in GSOD data
isd_history[, STNID_len := nchar(STNID)]
isd_history <- subset(isd_history, STNID_len == 12)

# remove stations where LAT or LON is NA
isd_history <- na.omit(isd_history, cols = c("LAT", "LON"))

# remove extra columns
isd_history[, c("USAF", "WBAN", "ICAO", "ELEV(M)", "STNID_len") := NULL]
```

## Add country names based on FIPS

``` r
isd_history <-
  isd_history[countrycode::codelist, on = c("CTRY" = "fips")]

isd_history <- isd_history[, c(
  "STNID",
  "NAME",
  "LAT",
  "LON",
  "CTRY",
  "STATE",
  "BEGIN",
  "END",
  "country.name.en",
  "iso2c",
  "iso3c"
)]

# clean data
isd_history[isd_history == -999] <- NA
isd_history[isd_history == -999.9] <- NA
isd_history <- isd_history[!is.na(isd_history$LAT) & !is.na(isd_history$LON), ]
isd_history <- isd_history[isd_history$LAT != 0 & isd_history$LON != 0, ]
isd_history <- isd_history[isd_history$LAT > -90 & isd_history$LAT < 90, ]
isd_history <- isd_history[isd_history$LON > -180 & isd_history$LON < 180, ]

# set colnames to upper case
names(isd_history) <- toupper(names(isd_history))
setnames(
  isd_history,
  old = "COUNTRY.NAME.EN",
  new = "COUNTRY_NAME"
)

# set country names to be upper case for easier internal verifications
isd_history[, COUNTRY_NAME := toupper(COUNTRY_NAME)]
```

## View and save the data

``` r
str(isd_history)
```

    ## Classes 'data.table' and 'data.frame':   26677 obs. of  11 variables:
    ##  $ STNID       : chr  "008268-99999" "409000-99999" "409010-99999" "409030-99999" ...
    ##  $ NAME        : chr  "WXPOD8278" "DARWAZ" "KHWAHAN" "KHWAJA-GHAR" ...
    ##  $ LAT         : num  33 38.4 37.9 37.1 37.1 ...
    ##  $ LON         : num  65.6 70.8 70.2 69.4 70.5 ...
    ##  $ CTRY        : chr  "AF" "AF" "AF" "AF" ...
    ##  $ STATE       : chr  "" "" "" "" ...
    ##  $ BEGIN       : int  20100519 19730304 19730629 20010925 19730304 20171229 19730701 19730101 19800316 19730101 ...
    ##  $ END         : int  20120323 20070905 20070608 20010925 20130703 20171229 20090511 20130313 20010828 20200406 ...
    ##  $ COUNTRY_NAME: chr  "AFGHANISTAN" "AFGHANISTAN" "AFGHANISTAN" "AFGHANISTAN" ...
    ##  $ ISO2C       : chr  "AF" "AF" "AF" "AF" ...
    ##  $ ISO3C       : chr  "AFG" "AFG" "AFG" "AFG" ...
    ##  - attr(*, ".internal.selfref")=<externalptr>

``` r
# write rda file to disk for use with GSODR package
save(isd_history,
     file = "../inst/extdata/isd_history.rda",
     compress = "bzip2")
```

# Notes

## NOAA Policy

Users of these data should take into account the following (from the
[NCEI
website](http://www7.ncdc.noaa.gov/CDO/cdoselect.cmd?datasetabbv=GSOD&countryabbv=&georegionabbv=)):

> “The following data and products may have conditions placed on their
> international commercial use. They can be used within the U.S. or for
> non-commercial international activities without restriction. The
> non-U.S. data cannot be redistributed for commercial purposes.
> Re-distribution of these data by others must provide this same
> notification.” [WMO Resolution 40. NOAA
> Policy](http://www.wmo.int/pages/about/Resolution40.html)

## R System Information

    ## ─ Session info ───────────────────────────────────────────────────────────────
    ##  setting  value                       
    ##  version  R version 3.6.3 (2020-02-29)
    ##  os       macOS Catalina 10.15.4      
    ##  system   x86_64, darwin15.6.0        
    ##  ui       X11                         
    ##  language (EN)                        
    ##  collate  en_AU.UTF-8                 
    ##  ctype    en_AU.UTF-8                 
    ##  tz       Australia/Brisbane          
    ##  date     2020-04-17                  
    ## 
    ## ─ Packages ───────────────────────────────────────────────────────────────────
    ##  package     * version    date       lib source                             
    ##  assertthat    0.2.1      2019-03-21 [1] CRAN (R 3.6.0)                     
    ##  base64enc     0.1-3      2015-07-28 [1] CRAN (R 3.6.0)                     
    ##  cli           2.0.2      2020-02-28 [1] CRAN (R 3.6.0)                     
    ##  clisymbols    1.2.0      2017-05-21 [1] CRAN (R 3.6.0)                     
    ##  countrycode * 1.1.1      2020-02-08 [1] CRAN (R 3.6.0)                     
    ##  crayon        1.3.4      2017-09-16 [1] CRAN (R 3.6.0)                     
    ##  curl          4.3        2019-12-02 [1] CRAN (R 3.6.0)                     
    ##  data.table  * 1.12.8     2019-12-09 [1] CRAN (R 3.6.0)                     
    ##  digest        0.6.25     2020-02-23 [1] CRAN (R 3.6.0)                     
    ##  dplyr         0.8.5      2020-03-07 [1] CRAN (R 3.6.0)                     
    ##  ellipsis      0.3.0      2019-09-20 [1] CRAN (R 3.6.0)                     
    ##  evaluate      0.14       2019-05-28 [1] CRAN (R 3.6.0)                     
    ##  fansi         0.4.1      2020-01-08 [1] CRAN (R 3.6.0)                     
    ##  glue          1.4.0.9000 2020-04-16 [1] Github (tidyverse/glue@a3375fe)    
    ##  htmltools     0.4.0      2019-10-04 [1] CRAN (R 3.6.0)                     
    ##  jsonlite      1.6.1      2020-02-02 [1] CRAN (R 3.6.0)                     
    ##  knitr         1.28       2020-02-06 [1] CRAN (R 3.6.0)                     
    ##  lifecycle     0.2.0      2020-03-06 [1] CRAN (R 3.6.0)                     
    ##  magrittr      1.5        2014-11-22 [1] CRAN (R 3.6.0)                     
    ##  pillar        1.4.3      2019-12-20 [1] CRAN (R 3.6.0)                     
    ##  pkgconfig     2.0.3      2019-09-22 [1] CRAN (R 3.6.0)                     
    ##  prompt        1.0.0      2020-03-13 [1] Github (gaborcsardi/prompt@b332c42)
    ##  purrr         0.3.3      2019-10-18 [1] CRAN (R 3.6.0)                     
    ##  R6            2.4.1      2019-11-12 [1] CRAN (R 3.6.0)                     
    ##  Rcpp          1.0.4.7    2020-04-16 [1] Github (RcppCore/Rcpp@a80908f)     
    ##  repr          1.1.0      2020-01-28 [1] CRAN (R 3.6.0)                     
    ##  rlang         0.4.5      2020-03-01 [1] CRAN (R 3.6.0)                     
    ##  rmarkdown     2.1        2020-01-20 [1] CRAN (R 3.6.0)                     
    ##  rstudioapi    0.11       2020-02-07 [1] CRAN (R 3.6.0)                     
    ##  sessioninfo * 1.1.1      2018-11-05 [1] CRAN (R 3.6.0)                     
    ##  skimr       * 2.1.1      2020-04-16 [1] CRAN (R 3.6.3)                     
    ##  stringi       1.4.6      2020-02-17 [1] CRAN (R 3.6.0)                     
    ##  stringr       1.4.0      2019-02-10 [1] CRAN (R 3.6.0)                     
    ##  tibble        3.0.0      2020-03-30 [1] CRAN (R 3.6.2)                     
    ##  tidyselect    1.0.0      2020-01-27 [1] CRAN (R 3.6.0)                     
    ##  vctrs         0.2.4      2020-03-10 [1] CRAN (R 3.6.0)                     
    ##  withr         2.1.2      2018-03-15 [1] CRAN (R 3.6.0)                     
    ##  xfun          0.13       2020-04-13 [1] CRAN (R 3.6.2)                     
    ##  yaml          2.2.1      2020-02-01 [1] CRAN (R 3.6.0)                     
    ## 
    ## [1] /Library/Frameworks/R.framework/Versions/3.6/Resources/library
