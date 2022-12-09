
# [***PROJECT***](/data/html/index-2015-01-04.html)

## SETUP

Set working directory to the path of the file you want to read or adjust
the path in the function

Use the ff library. Normal read.csv doesn't work with a file this big

```{r}
library(tidyverse) 
library(dplyr) 
library(ff) 
library(rjson)
```

### Journeys

Read CSV file containing all the 78million journeys. Use "Import Dataset
from text, readr" in the Environment tab. (if importing this way, the
next step is redundant as the backup file is already complete with the
date column and time in the right format)

Extract the date and change the format to ISO so we won't have issues in
the future when using other programs

```{r}
trips<- trips %>% mutate(Date = as.Date(Start.Date, format="%d/%m/%Y")) %>% mutate( Date = format(Date, "%Y/%m/%d"))
```

### Stations names and coordinates

Imports the coordinates of the stations from the json file

```{r}
address <- "/home/elias/Desktop/data/Merged/coordinates.json" 
coordinatesRaw <- fromJSON(file = address)
```

Go into each item in the json file, and extracts the value of the given

column and transforms it into a value instead of a list

```{r}
stationid <- unlist(lapply(coordinatesRaw, [[, 2)) 
stationid <- gsub("^BikePoints_", "", stationid)
station <- unlist(lapply(coordinatesRaw, [[, 4)) 
lat <- unlist(lapply(coordinatesRaw, [[, 9)) 
lon <- unlist(lapply(coordinatesRaw, [[, 10))
```

Create the data frame from the values extracted in the previous step

```{r}
coordinates <- data.frame(station, stationid, lat, lon)
coordinates <- data.frame(station, stationid, lat, lon)
```

## CLEANING

The data needs to be cleaned. As TFL has updated the list of docking
stations in their publicly available data, some of the decommissioned
stations are not included in the list. To solve this a bit of detective
work and Google street view historic data came in very handy.

Also worth noting is that included in the data set are some test
stations along with pop ups, which are close to impossible to locate
accurately without an insane amount of time / luck. Since those stations
only add up to less than 200 total journeys, they have not been
included.

By using the code below, I found the NA where a journey didn't have a
matching start or end docking station id. Knowing this, I used Google
Street view and use the historic imagery to see where the missing
station is or used to be. Most of the time the docking stations had been
decommissioned and removed from the currently available list.

```{r}
groupedStart <- trips %>% group_by(StartStation.Id, StartStation.Name) %>% summarise(Weight = n(), .groups = 'drop') 
groupedStart <- groupedStart %>% rename(stationid = StartStation.Id) mergedStart <- merge(coordinatesfixed, groupedStart, all = TRUE)

groupedEnd <- trips %>% group_by(EndStation.Id, EndStation.Name) %>% summarise(Weight = n(), .groups = 'drop') 
groupedEnd <- groupedEnd %>% rename(stationid = EndStation.Id) mergedEnd <- merge(coordinatesfixed, groupedEnd, all = TRUE)


```

This creates a new data frame with all the missing stations that I could
find, along with their coordinates and Id (in some cases the id is new,
since TFL reused the id of decommissioned docks, which has made this
extra difficult.

```{r}
df1 <- data.frame(station = c("Bourne Street, Belgravia","Great Marlborough St, Soho","Chelsea Green, Chelsea","Colombo Street, Southwark","Cumberland Gate, Hyde Park","Kennington Lane Tesco, Vauxhall","Black Prince Road, Vauxhall","Central House, Whitechapel","Lord's, St. John's Wood","Speakers' Corner 1, Hyde Park","Speakers' Corner 2, Hyde Park","Harrington Square 2, Camden Town","Christian Street, Whitechapel","Westfield Eastern Access Road, Shepherd's Bush","Rodney Street, Angel","Westfield Library Corner, Shepherd's Bush","Grant Road Central, Clapham Junction","Grant Road West, Clapham Junction","Michael Road, Walham Green","Hammersmith Town Hall, Hammersmith","Rossmore Road, Marylebone","Arundel Street","Wansey Street, Walworth", "Thornfield House, Poplar", "Thessaly Road North, Nine Elms", "Exhibition Road Museums 1, South Kensington", "Exhibition Road Museums 2, South Kensington"), stationid = c(137,159,220,240,304,305,310,33,363,407,406,425,489,555,59,591,672,1000,731,753,760,1002,79, 567, 1001, 432, 482), lat = c(51.490856,51.514667,51.490714,51.505342,51.512151,51.486824,51.490994,51.515231,51.528900,51.512301,51.512232,51.533403,51.512950,51.509159,51.531825,51.506093,51.464655,51.464416,51.477292,51.492736,51.525194,51.511650,51.491056, 51.509913,51.477273, 51.496471, 51.495948), lon = c(-0.153256,-0.137694,-0.166221,-0.105596,-0.162146,-0.115659,-0.117122,-0.070133,-0.171592,-0.159925,-0.160529,-0.139258,-0.064150,-0.219645,-0.114333,-0.224303,-0.173719,-0.174521,-0.189155,-0.234055,-0.164032,-0.113885,-0.097291, -0.023606, -0.138781, -0.173757, -0.173693))
```

Here I remove the stations that are in the list of docks but have no
registered journeys.

```{r}
coordinatesfixed <- rbind(coordinates, df1) 
coordinatesfixed <- coordinatesfixed %>% filter(station != "Burgess Park Albany Road, Walworth") coordinatesfixed <- coordinatesfixed %>% filter(station != "South Bermondsey Station, Bermondsey") coordinatesfixed <- coordinatesfixed %>% filter(station != "Brandon Street, Walworth") coordinatesfixed <- coordinatesfixed %>% filter(station != "The Blue, Bermondsey")
```

Remove the journeys from pop up, test and workshop stations; as well as
change the Id of docking stations that have been decommissioned and
their id reused for a new stations.

```{r}
trips$StartStation.Name <- gsub(" , ", ", ", trips$StartStation.Name) 
trips$EndStation.Name <- gsub(" , ", ", ", trips$EndStation.Name) 
trips <- trips %>% filter(StartStation.Id != 241) 
trips <- trips %>% filter(EndStation.Id != 241) 
trips <- trips %>% filter(StartStation.Name != "LSP1") 
trips <- trips %>% filter(StartStation.Name != "LSP2") 
trips <- trips %>% filter(EndStation.Name != "LSP1") 
trips <- trips %>% filter(EndStation.Name != "LSP2") 
trips <- trips %>% filter(StartStation.Name != "Pop Up Dock 1") 
trips <- trips %>% filter(StartStation.Name != "Pop Up Dock 2") 
trips <- trips %>% filter(EndStation.Name != "Pop Up Dock 1") 
trips <- trips %>% filter(EndStation.Name != "Pop Up Dock 2") 
trips <- trips %>% filter(StartStation.Id != 346) 
trips <- trips %>% filter(EndStation.Id != 346) 
trips <- trips %>% filter(StartStation.Name != "PENTON STREET COMMS TEST TERMINAL _ CONTACT MATT McNULTY") 
trips <- trips %>% filter(EndStation.Name != "PENTON STREET COMMS TEST TERMINAL _ CONTACT MATT McNULTY") 
trips <- trips %>% filter(EndStation.Name != "Electrical Workshop PS") 
trips <- trips %>% filter(EndStation.Name != "Mechanical Workshop Clapham") 
trips$StartStation.Id[trips$StartStation.Id == 205] <- 206 
trips$EndStation.Id[trips$EndStation.Id == 205] <- 206 
trips$StartStation.Id[trips$StartStation.Id == 482 & trips$StartStation.Name == "Thornfield House, Poplar"] <- 567 
trips$EndStation.Id[trips$EndStation.Id == 482 & trips$EndStation.Name == "Thornfield House, Poplar"] <- 567 
trips$StartStation.Id[trips$StartStation.Id == 659 & trips$StartStation.Name == "Grant Road West, Clapham Junction"] <- 1000 
trips$EndStation.Id[trips$EndStation.Id == 659 & trips$EndStation.Name == "Grant Road West, Clapham Junction"] <- 1000 
trips$StartStation.Id[trips$StartStation.Name == "Thessaly Road North, Wandsworth Road"] <- 1001
trips$EndStation.Id[trips$EndStation.Name == "Thessaly Road North, Wandsworth Road"] <- 1001
trips$StartStation.Id[trips$StartStation.Name == "Thessaly Road North, Nine Elms"] <- 1001
trips$EndStation.Id[trips$EndStation.Name == "Thessaly Road North, Nine Elms" ] <-1001
trips$StartStation.Id[trips$StartStation.Name == "Arundel Street, Temple"] <- 1002 
trips$EndStation.Id[trips$EndStation.Name == "Arundel Street, Temple"] <- 1002
```

Save the now clean data

```{r}
write.csv(trips, "/media/elias/3EE6-C041/Clean Data/tripsClean.csv", row.names = FALSE) 
write.csv(coordinatesfixed, "/home/elias/Desktop/coordinatesClean.csv", row.names = FALSE)


```

## ANALYSIS

Using the clean data, renamed trips instead of tripsClean or
cooordinatesClean for ease of use.

Group by date and station giving you the amount of trips that took place
on that date from that station

```{r}
grouped <- trips %>% group_by(Date, StartStation.Id) %>% summarise(Total=n(), .groups = 'drop')
```

Rename column to be able to merge

```{r}
grouped <- grouped %>% rename( station = StartStation.Name)
```

Total trips grouped by day

```{r}
daytotals <- trips %>% group_by(Date) %>% summarise(dayTotal=n(), .groups = 'drop')
```

### Save files

Save files to CSV format to be used in other programs

```{r}
write.csv(coordinates, "/home/elias/Desktop/coord.csv", row.names = FALSE) 
write.csv(grouped, "/home/elias/Desktop/data.csv", row.names = FALSE) 
write.csv(daytotals, "/home/elias/Desktop/daytotals.csv", row.names = FALSE)
```

## (OPTIONAL)

Get the hour in a separate column

```{r}
trips <- trips %>% mutate(Time = as.POSIXct(Start.Date, format="%d/%m/%Y %H:%M")) %>% mutate( Time = format(Time, "%H:%M"))
```

Backup trips

```{r}
address <- "/home/elias/Desktop/tripsbackup.csv" 
write.csv(trips, address , row.names = FALSE)
```

## GEPHY

```{r}
edges <- subset(tripsbackup, select = c(Date, StartStation.Id, EndStation.Id)) 
edges <- edges %>% rename(Source = StartStation.Id, Target = EndStation.Id)
```

### For non interactive graph with all the journeys

```{r}
edges100filter <- edges[edges$Weight > 100,] 
# Write the dataframe to a csv file with the row names so we create a column that'll be used as an id
write.csv(edges100filter,"/home/elias/Desktop/edgesid.csv", row.names = TRUE) 
# Remove edges
id <- subset(edgesid, select = -c(X)) write.csv(edgesid,"/home/elias/Desktop/edgesid.csv", row.names = TRUE) 
# Remove the dataframe called edgesid, as we have a backup with the colum id 
rm(edgesid) 
# Import the csv file called edgesid to get the file with id 
edgesid <- edgesid %>% rename(id = X) 
# set up cut-off values 
breaks <- c(100,148,196,244,292,340,388,150000) 
# specify interval/bin labels 
tags <- c("-3","-2","-1","0","1","2","3") 
# bucketing values into bins 
group_tags <- cut(edges100filter$Weight, breaks=breaks, include.lowest=TRUE, right=FALSE, labels=tags)
# inspect bins 
summary(group_tags)

groups <- as.data.frame(group_tags)

write.csv(group_tags,"/home/elias/Desktop/tags.csv", row.names = TRUE)

tags <- tags %>% rename(id = X, group = x)

newEdges <- merge(edgesid, tags, all.x = TRUE)

newEdges <- subset(newEdges, select = -c(id))

write.csv(newEdges,"/home/elias/Desktop/newEdges.csv", row.names = FALSE)
```

### For interactive graph with journeys by date

Split the dataset into individual files containing only the trips made
in a day. Creates a dictionary of dataframes

```{r}
splitdata <- split(edges, edges$Date)
```

Loop changing the format of the date to be used as a name for the file.
Use sprintf is the same as a F-string in python

```{r}
for (x in splitdata){ name <- gsub('/','-', unique(x[c("Date")])) write.csv(x, sprintf("/home/elias/Desktop/edges/%s.csv", name), row.names = FALSE) }
```

## (OPTIONAL)

### Useful commands

Get all the distinct dates in the dataset

```{r}
date <- unique(edges[c("Date")])
```

create vector with dates

```{r}
for (dates in date){ print(dates) }
```

### Joins

Left Join

```{r}
newDataframe <- merge(Df1, Df2, all.x = TRUE)
```

Right Join

```{r}
newDataframe <- merge(Df1, Df2, all.y = TRUE)
```

Inner Join

```{r}
newDataframe <- merge(Df1, Df2,)
```

Outer Join

```{r}
newDataframe <- merge(Df1, Df2, all = TRUE)
```

If no matching column names use by.x = "name of column in Df1" and by.y
= "name of column in Df2"

Merge equivalent dataframes (adding rows)

```{r}
rbind(Dataframe, data_to_add)
```

Data to be added can be a vector or a dataframe. If it is a vector, it
has to be just one row, which means it has to be as long as the number
of columns in the dataframe.

Remove rows from a dataframe

```{r}
updated_myData <- subset(myData, condition != conditionvalue)
```
