# taxidata
NYC taxi datasets for Customer Models

# Customer Models: NYC Taxi & NTA Analytics (Feb 2026)
This repository contains the integrated datasets for the Customer Models course. This dataset:
* Takes the raw taxi data for February 2026 from https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page and adds an NTA code (useful to connect data to other NY open data)
* Contains additional data on demographics, housing, and socio-economics (including NTA codes)
* Contains a file on licensed businesses at the NTA level

---

## The Dataset
The primary file is taxi_data.parquet. However, this file contains >3m rows. We therefore recommend you use taxi_data_100k.parquet instead, which contains a random sample of 100.000 taxi rides.

## Loading the Data in R

To load the Parquet file directly from this repository, use the following code.

```r
# Required libraries
# install.packages(c("arrow", "dplyr"))
library(arrow)
library(dplyr)

#github url
gh_url<-"https://raw.githubusercontent.com/rphars/taxidata/main/"

#Download and Read the 100k file. 
taxi_data<-read_parquet(paste(gh_url,'taxi_data_100k.parquet',sep=""))
#if you absolutely want the 3m rows, load with taxi_data.parquet. We recommend using the 100k file, though.
#note, read.csv2, as files have semicolon as separator rather than comma
demdata<-read.csv2(paste(gh_url,'demdata_simpl.csv',sep=""))
socdata<-read.csv2(paste(gh_url,'socdata_simpl.csv',sep=""))
housingdata<-read.csv2(paste(gh_url,'housingdata_simpl.csv',sep=""))
econdata<-read.csv2(paste(gh_url,'econdata_simpl.csv',sep=""))
#without the csv2, as this file _does_ have comma separated values.
company_data<-read.csv(paste(gh_url,'Issued_Licenses_20260401.csv',sep=""))

