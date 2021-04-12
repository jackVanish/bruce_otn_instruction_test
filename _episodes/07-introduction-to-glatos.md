---
title: Introduction to glatos Data Processing Package
teaching: 30
exercises: 0
questions:
    - "How do I load my data into glatos?"
    - "How do I filter out false detections?"
    - "How can I consolidate my detections into detection events?"
    - "How do I summarize my data?"
---

The glatos package is a powerful toolkit that provides a wide range of functionality for loading,
processing, and visualizing your data. With it, you can gain valuable insights
with quick and easy commands that condense high volumes of base R into straightforward
functions, with enough versatility to meet a variety of needs.

First, we must set our working directory and import the relevant library.

~~~
## Set your working directory ####

setwd("./data")
library(glatos)
library(tidyverse)
library(VTrack)
library(utils)
~~~
{: .language-r}

Your code may not be in the 'code/glatos' folder, so use the appropriate file path for
your data.


Next, we will create paths to our detections and receiver files. GLATOS can
function with both GLATOS and OTN Node-formatted data, but the functions are different
for each. Both, however, provide a marked performance boost over base R, and Both
ensure that the resulting data set will be compatible with the rest of the glatos
framework.

First we will combine all our data extracts into one file before glatos can read them in.

~~~
format <- cols( # Heres a col spec to use when reading in the files
  .default = col_character(),
  datelastmodified = col_date(format = ""),
  bottom_depth = col_double(),
  receiver_depth = col_double(),
  sensorname = col_character(),
  sensorraw = col_character(),
  sensorvalue = col_character(),
  sensorunit = col_character(),
  datecollected = col_datetime(format = ""),
  longitude = col_double(),
  latitude = col_double(),
  yearcollected = col_double(),
  monthcollected = col_double(),
  daycollected = col_double(),
  julianday = col_double(),
  timeofday = col_double(),
  datereleasedtagger = col_logical(),
  datereleasedpublic = col_logical()
)
detections <- tibble()
for (detfile in list.files('.', full.names = TRUE, pattern = "proj.*\\.zip")) {
  print(detfile)
  tmp_dets <- read_csv(detfile, col_types = format)
  detections <- bind_rows(detections, tmp_dets)
}
write_csv(detections, 'all_dets.csv', append = FALSE)
~~~
{:.language-r}


With our new file in hand, we'll want to use the read_otn_detections function
to load our data into a dataframe. In this case, our data is formatted in the ACT
style- if it were GLATOS formatted, we would want to use read_glatos_detections()
instead.

Remember: you can always check a function's documentation by typing a question
mark, followed by the name of the function.
~~~
## GLATOS help files are helpful!! ####
?read_otn_detections

# Save our detections file data into a dataframe called detections
detections <- read_otn_detections(det_file=det_file_name)
~~~
{: .language-r}


Remember that we can use head() to inspect a few lines of our data to ensure it was loaded properly.

~~~
# View first 2 rows of output
head(detections, 2)
~~~
{: .language-r}

With our data loaded, we next want to apply a false filtering algorithm to reduce
the number of false detections in our dataset. glatos uses the Pincock algorithm
to filter probable false detections based on the time lag between detections- tightly
clustered detections are weighted as more likely to be true, while detections spaced
out temporally will be marked as false.

~~~
## Filtering False Detections ####
## ?glatos::false_detections

# write the filtered data (no rows deleted, just a filter column added)
# to a new det_filtered object
detections_filtered <- false_detections(detections, tf=3600, show_plot=TRUE)
head(detections_filtered)
nrow(detections_filtered)
~~~
{: .language-r}

The false_detections function will add a new column to your dataframe, 'passed_filter'.
This contains a boolean value that will tell you whether or not that record passed the
false detection filter. That information may be useful on its own merits; for now,
we will just use it to filter out the false detections.

~~~
# Filter based on the column if you're happy with it.

detections_filtered <- detections_filtered[detections_filtered$passed_filter == 1,]
nrow(detections_filtered) # Smaller than before
~~~
{: .language-r}

With our data properly filtered, we can begin investigating it and developing some
insights. glatos provides a range of tools for summarizing our data so that we can
better see what our receivers are telling us.

We can begin with a summary by animal, which will group our data by the unique animals we've
detected.

~~~
# Summarize Detections ####
#?summarize_detections
#summarize_detections(detections_filtered)

# By animal ====
sum_animal <- summarize_detections(detections_filtered, location_col = 'station', summ_type='animal')

sum_animal
~~~
{: .language-r}

We can also summarize by location, grouping our data by distinct locations.

~~~
# By location ====

sum_location <- summarize_detections(detections_filtered, location_col = 'station', summ_type='location')

head(sum_location)
~~~
{: .language-r}

If you had some other location-like column you'd prefer to group by, you can specify that. For example, we will create a new column and use that as the location.

~~~
# You can make your own column and use that as the location_col
# For example we will create a uniq_station column for if you have duplicate station names across projects
detections_filtered_special <- detections_filtered %>% 
  mutate(station_uniq = paste(glatos_array, station, sep=':'))

sum_location_special <- summarize_detections(detections_filtered_special, location_col = 'station_uniq', summ_type='location')

head(sum_location_special)
~~~
{: .language-r}

Finally, we can summarize by both dimensions.
~~~
# By both dimensions
sum_animal_location <- summarize_detections(det = detections_filtered,
                                            location_col = 'station',
                                            summ_type='both')

head(sum_animal_location)
~~~
{: .language-r}

Summarising by both dimensions will create a row for each station and each animal pair, let's filter out the station where the animal wasn't detected.
~~~
# Filter out stations where the animal was NOT detected.
sum_animal_location <- sum_animal_location %>% filter(num_dets > 0)

sum_animal_location
~~~
{: .language-r}

One other method- we can summarize by a subset of our animals as well. If we only want
to see summary data for a fixed set of animals, we can pass an array containing the animal_ids
that we want to see summarized.

~~~
# create a custom vector of Animal IDs to pass to the summary function
# look only for these ids when doing your summary
tagged_fish <- c('PROJ58-1218508-2015-10-13', 'PROJ58-1218510-2015-10-13')

sum_animal_custom <- summarize_detections(det=detections_filtered,
                                          animals=tagged_fish,  # Supply the vector to the function
                                          location_col = 'station',
                                          summ_type='animal')

sum_animal_custom
~~~
{: .language-r}

Alright, we can summarize our data. Let's move on and see if we can make our dataset
more amenable to plotting by reducing it from detections to detection events.

Detection Events differ from detections in that they condense a lot of temporally and
spatially clustered detections for a single animal into a single detection event. This is
a powerful and useful way to clean up the data, and makes it easier to present and
clearer to read. Fortunately, glatos lets us do this easily.

~~~
# Reduce Detections to Detection Events ####

# ?glatos::detection_events
# arrival and departure time instead of multiple detection rows
# you specify how long an animal must be absent before starting a fresh event

events <- detection_events(detections_filtered,
                           location_col = 'station', 
                           time_sep=3600)

head(events)
~~~
{: .language-r}

We can also keep the full extent of our detections, but add a group column so that we can see how they
would have been condensed.

~~~
# keep detections, but add a 'group' column for each event group
detections_w_events <- detection_events(detections_filtered,
                                        location_col = 'station',
                                        time_sep=3600, condense=FALSE)
~~~
{: .language-r}

With our filtered data in hand, let's move on to some visualization.
