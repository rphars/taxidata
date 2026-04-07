# taxidata
NYC taxi datasets for Customer Models

# Customer Models: Insights in the NYC Taxi Market (2026)
This repository contains datasets for the Customer Models group assignment. These datasets consist of:
* A dataset equal to the raw taxi data for February 2026 from https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page, but including an NTA code (useful to connect data to other NY open data)
* Contains additional data on demographics, housing, and socio-economics (joinable using NTA codes)
* Contains a file on licensed businesses at the NTA level
* The text below also contains instructions on joining weather data.

---

## The Dataset
The primary file is taxi_data.parquet. However, this file contains >3m rows. We therefore recommend you use taxi_data_100k.parquet instead, which contains a random sample of 100.000 taxi rides. You may want to use a much smaller sample initially to speed up code development and model estimation.

## Data Dictionaries
* For the primary file, find the dictionary here: https://www.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf
* For the econ/demo/socio/housing dictionary, download data_dict_overall.xlsx from this repository.
* For the business licenses dictionary, see https://data.cityofnewyork.us/Business/Issued-Licenses/w7w3-xahh/about_data
* Weather data is added in the code below, it needs no files from this repository

## Loading the Data in R

To load the Parquet data directly from this repository, use the following code.

```r
# Required libraries
# install.packages(c("arrow", "dplyr","jsonlite","lubridate"))
library(arrow)
library(dplyr)
library(jsonlite)
library(lubridate)

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

#Note, you probably want to filter columns first to only select those you want before you combine with the rest
demdata_subset <- demdata %>%
  #add columns where desired
  select(GeoID,Pop_1E)


#Files can be joined together using the NTA id. 
#We do this twice, because there's two locations: pickup and dropoff.

#the rename_with makes sure that all columns added for pickup are prefaced by PU, and DO for dropoff
taxi_data<- taxi_data%>%
  left_join(
    demdata_subset %>% rename_with(~ paste0("PU_", .), -GeoID), 
    by = c("PU_NTA_Code" = "GeoID")
  ) 

taxi_data<-taxi_data%>%
  left_join(
    demdata_subset %>% rename_with(~ paste0("DO_", .), -GeoID), 
    by = c("DO_NTA_Code" = "GeoID")
  )


#Download weather data with
api_url <- "https://archive-api.open-meteo.com/v1/archive?latitude=40.7831&longitude=-73.9712&start_date=2026-02-01&end_date=2026-02-28&hourly=temperature_2m,relative_humidity_2m,apparent_temperature,precipitation,snowfall,snow_depth,pressure_msl,cloud_cover,wind_speed_10m,sunshine_duration,is_day&timezone=America%2FNew_York"
weather_raw <- fromJSON(api_url)

weather_clean <- as.data.frame(weather_raw$hourly) %>%
  mutate(
    datetime = ymd_hm(time, tz = "America/New_York"),
    # Convert sunshine seconds to minutes
    sunshine_minutes = sunshine_duration / 60
  ) %>%
  # Drop the raw string time and raw sunshine seconds
  select(-time, -sunshine_duration)
# weather data can be joined based on date and hour. Achieving this is left up to you.
```

## Loading the Data in Python
Run this in your terminal (not your Python script) to ensure you have the necessary libraries:

```bash
### 1. Install Required Packages
pip install pandas fastparquet requests
```
Then add this to your python script and run.
```Python
import requests
import pandas as pd
import fastparquet
# GitHub URL for the raw data
gh_url = "https://raw.githubusercontent.com/rphars/taxidata/main/"

# 1. Download and Read the 100k Taxi Data file.
# We explicitly use 'fastparquet' as the engine for speed and reliability
taxi_data = pd.read_parquet(f"{gh_url}taxi_data_100k.parquet", engine='fastparquet')

# 2. Load the Census/Economic Data
# low_memory=False prevents pandas from guessing data types on massive census files.
demdata = pd.read_csv(f"{gh_url}demdata_simpl.csv", sep=";", low_memory=False)
socdata = pd.read_csv(f"{gh_url}socdata_simpl.csv", sep=";", low_memory=False)
housingdata = pd.read_csv(f"{gh_url}housingdata_simpl.csv", sep=";", low_memory=False)
econdata = pd.read_csv(f"{gh_url}econdata_simpl.csv", sep=";", low_memory=False)

# The standard comma-separated company data file
company_data = pd.read_csv(f"{gh_url}Issued_Licenses_20260401.csv", low_memory=False)

# Joining the Data (Pickup and Dropoff)

# Filter demographic columns to only select those you want to merge. You can add columns and -of course- the other datasets here.
demdata_subset = demdata[['GeoID', 'Pop_1E']].copy()

# Join for Pickup (PU)
# Create a dictionary to rename all columns (except GeoID) with a "PU_" prefix
pu_rename = {col: f"PU_{col}" for col in demdata_subset.columns if col != 'GeoID'}
demdata_pu = demdata_subset.rename(columns=pu_rename)

# Perform the left join and drop the redundant RHS key
taxi_data = pd.merge(
    taxi_data, 
    demdata_pu, 
    how="left", 
    left_on="PU_NTA_Code", 
    right_on="GeoID"
).drop(columns=['GeoID'])

# 2. Join for Dropoff (DO)
# Create a dictionary to rename all columns (except GeoID) with a "DO_" prefix
do_rename = {col: f"DO_{col}" for col in demdata_subset.columns if col != 'GeoID'}
demdata_do = demdata_subset.rename(columns=do_rename)

# Perform the left join and drop the redundant RHS key
taxi_data = pd.merge(
    taxi_data, 
    demdata_do, 
    how="left", 
    left_on="DO_NTA_Code", 
    right_on="GeoID"
).drop(columns=['GeoID'])

# Positron only
# Open the dataframe in the Data Viewer
%view taxi_data
```
