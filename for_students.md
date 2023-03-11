# Learning from mistakes
## Overview 
Learning and struggling is okay. It took me sometime to develop an efficient strategy and I managed to learn a lot through my mistakes. There was also a lot of by-product which might be useful later on so I want to leave it here 
## The fixation on table schema
The csv. files provided have the same structure and can be united into one table using the following SQL query.

``` CREATE TABLE trip_data.year_trip_data AS 
 SELECT * 
 FROM (
   SELECT * FROM `trip_data.FEB_2022`
   UNION ALL
   SELECT * FROM `trip_data.MAR_2022`
   UNION ALL
   ...
 )
```
At that time I wanted to add columns ride_length and month but BigQuery does not let you alter external tables which I realised later.
When I think back about it. I know that I could have just used SELECT statement to achieve what I wanted but the realisation of that didn't come then and as a result I took a lot of  additional steps to prepare and process data. So I turned to Sheets to alter the csv Files.

## The Sheets Phase 
Some files were easy to work with as they were. What I quickly realised is that Sheets are unable to open files largere than 100Mb it was also something that I didn't know prior. Since I already started altering the default schema and do initial cleaning in Sheets I decided to find a way. Since the csv. files were too big why not split them in two? And this is where R comes in:
```
JUN_2022 <- read.csv("C:/Users/.../.../.../Cyclistic/202206-divvy-tripdata.csv")
JUN_2022_1 <- head(JUN_2022,nrow(JUN_2022)/2)
JUN_2022_2 <- tail(JUN_2022,nrow(JUN_2022)/2)
write.csv(JUN_2022_1,'C:/.../.../Documents/JUN_2022_1.csv', row.names=FALSE)
write.csv(JUN_2022_2,'C:/.../.../Documents/JUN_2022_2.csv', row.names=FALSE) 
```

And for the files with odd number of rows.

``` 
SEP_2022 <- read.csv("C:/Users/.../.../.../Cyclistic/202209-divvy-publictripdata.csv")
half_len_SEP <- nrow(SEP_2022)%/%2
half_len_1_SEP <- half_len_SEP+1
SEP_2022_1 <- SEP_2022[1:half_len_SEP,]
SEP_2022_2 <- SEP_2022[half_len_1_SEP:nrow(SEP_2022),]
write.csv(SEP_2022_1,'C:/.../.../Documents/SEP_2022_1.csv', row.names=FALSE)
write.csv(SEP_2022_2,'C:/.../.../Documents/SEP_2022_2.csv', row.names=FALSE) 
```

Now we have appropriately sized csv. files to use in spreadsheet software. Later on I added the columns I wanted in each and every one of them, trimmed the whitespace and cleaned the data. What happened afterwards is that the coordinates in some cases were missing and R would fill the empty cells with the string N/A and it would change the datatype of the whole column to STRING. As a result it was no longer possible to UNION ALL anymore.
Now we had to find a way to reweite the files omitting any 'N/A' instances
```
library("dplyr")

S_PATH <- "C:\\...\\...\\...\\F-CLEAN\\Cleaned\\" 
list_filenames <- list.files(path=S_PATH) 
total_file <- read.csv(paste(S_PATH,list_filenames[1],sep=''))
for (i in list_filenames[2:length(list_filenames)]){
  temp_filename <- paste(S_PATH,i,sep="")                                 
  temp_file <- read.csv(temp_filename)
  print(temp_filename)
  total_file <- rbind(total_file,temp_fil
  rm(temp_file)
}

write.csv(total_file,paste(S_PATH,"total.csv",sep=""), row.names=FALSE)
map_data <- total_file %>%
  group_by(start_lat, start_lng) %>%
  dplyr::add_count(start_lat, start_lng, sort = TRUE) %>% 
  as.data.frame()
View(map_data)

for (i in list_filenames){  
  temp_filename <- paste(S_PATH,i,sep="")                                 
  temp_file <- read.csv(temp_filename)                                     
  if (nrow(temp_file[!complete.cases(temp_file),]) > 0) { 
    print(paste(nrow(temp_file[!complete.cases(temp_file),])," ", i))
    temp_file <- na.omit(temp_file)
  }
  write.csv(temp_file,paste(S_PATH,"Cleaned\\",i,sep=""), row.names=FALSE)
  rm(temp_file)
}
```
All this work wouldn't be necessary if Just used the SELECT AS statement in SQL in the first place :)
to be continued 
