---
layout: post
title: "Get CGM Data"
author: "Andy Choens"
category: "Data Management"
excerpt: Documented R code for querying data from a Nightscout DB.

---

About
=====

[Nighscout](https://nightscout.github.io/) stores all of Karen's CGM
data on a [Mongodb](https://www.mongodb.org) server, controlled by our
family. To use Karen's Nightscout data to better understand her
diabetes, I have to learn how to use Mongo. Mongodb appears to be the
formal name of the product. It is referred to here as Mongo which
appears to be an abbreviation adopted by the community.

I have been using the [Robomongo](http://robomongo.org/) application to
view Mongo JSON data directly in the db server. Like R, it is FOSS and
is available for Linux, Mac and Windows.

Code & Data
===========

The .Rmd file used to create this post is available on
[GitHub](https://github.com/Choens/blood-sugars/blob/master/get-cgm-data.Rmd).
The public data sets are in the [data
sub-folder](https://github.com/Choens/blood-sugars/tree/master/data).
The date on which the data file was built is provided in the CSV file
name.

File names and content is subject to periodic updates. Open a bug or
download one of these datasets for future use if you want to have a
guaranteed stable data-set. We have been using our current rig since
June 21, 2015. We are producing data at a prodigious rate. Data
collected prior to June 2015 is less consistent due to the limitations
with the previous rig and mistakes we made figuring out how to use it. I
do not recommend using data collected prior to June 2015 for analysis.

Goals
=====

1.  *DONE:* Query Karen's Nightscout database.
2.  *DONE:* Import Nightscout data (JSON) and turn it into a R data
    frame.
3.  *DONE:* QA this data frame (number of entries, data completeness,
    etc.).
4.  *DONE:* Export the data frame as a CSV for further analysis.
5.  *IN PROGRESS:* Develop a function to simplify the query process
    against Nightscout databases.

Goals \#1 and \#2 were hard because I am not an experienced Mongo
programmer. Goals \#3 and \#4 were easy, these "goals" are basic R
programming. Goal \#5, a function to simplify querying the Nightscout
data, will be completed at a later date. I will build upon what I
demonstrate here in that process.

R Packages
==========

There are two R packages able to query Mongo:

-   [RMongo](http://cran.r-project.org/web/packages/RMongo/index.html)
    and
-   [rmongodb](http://cran.r-project.org/web/packages/rmongodb/index.html).

I skimmed the documentation for both packages. RMongo looked easier to
use so I tried it first. Unfortunately, I was never able to connect to
our Mongo server running on [mongolab](https://mongolab.com/) using
RMongo. RMongo appears to lack the ability to connect to Mongo using
user names, passwords or custom ports. All three of these re required to
connect to a mongolab server. Based on the documentation, RMongo has a
more R-centric API and it is probably great if you want to connect to a
local Mongo server.

The second package, rmongodb, is complicated but works well. This
package loops over a cursor, an algorithm commonly used in web
development but not in R which tends to avoid loops. Although it feels
unnecessarily complicated, it works well and it is fast. Structural
differences in JSON objects and data structures used in R add additional
complexity to the code below.

Goals 1 & 2: Import & Convert To Data Frame
===========================================

Nearly all of my database experience is with Relational Database
Management Systems (RDBMS) such as Postgres or SQL Server. To succeed
with rmongodb, I had to adopt a different work flow than I am used to.
It isn't really harder, but it quite different compared to what I would
do to query data via RODBC or other RDBMS-oriented package.

Init Chunk
----------

I always start with a chunk called init, to define variables, load
packages, etc. This script has two dependencies, rmongodb and dplyr.

The file, passwords.R is not part of the public repo for security
reasons. Thus, you cannot run this code as easily as some other code for
R. If you have your own Nightscout mongo server running, you can adopt
the code in the
[passwords.example.R](https://github.com/Choens/blood-sugars/blob/master/passwords.example.R)
file to connect to your own database.

    ## passwords.R -----------------------------------------------------------------
    ## Defines the variables I don't want to post to GitHub. (Sorry)
    ## ns is short for Nightscout.
    ## ns_host = URL for the host server (xyz.mongolab.com:123456)
    ## ns_db = Name of the Nightscout database (lade_k_nightscout)
    ## ns_user = Admin User Name (Not admin)
    ## ns_pw = Admin Password (Not Admin)
    source("passwords.R")

    ## Required Packages -----------------------------------------------------------
    library(rmongodb)  ## For importing the data.
    library(dplyr)     ## For QA / data manipulation.
    library(pander)    ## For nice looking tables, etc.

Loading the rmongodb package, version 1.8.0, returns the following
dramatic warning:

> WARNING!

> There are some quite big changes in this version of rmongodb.
> mongo.bson.to.list, mongo.bson.from.list (which are workhorses of many
> other rmongofb high-level functions) are rewritten. Please, TEST IT
> BEFORE PRODUCTION USAGE. Also there are some other important changes,
> please see NEWS file and release notes at
> <https://github.com/mongosoup/rmongodb/releases/>

In spite of this warning, appears to have worked fine. Opening a
connection to Mongo is similar to opening a connection to a RDBMS. Mongo
stores data in an object called a collection, which is sorta-kinda like
a table in a traditional RDBMS. However, unlike a table, the structure
of a collection is not defined prior to use. There are several other
important differences, which you can learn about by reading the
[introduction tutorial](https://www.mongodb.org/about/introduction/)
written by actual mongo experts.

The following code chunk returns a list of all the collections which
exist in the Nighscout database. The "entries" collection is the only
collection we are interested in today. The database name,
lade\_k\_nightscout is prepended to each collection name.

    ## Open a connection to mongo --------------------------------------------------
    con <- mongo.create(host = ns_host,
                        username = ns_user,
                        password = ns_pw,
                        db = ns_db
                       )

    ## Make sure we have a connection ----------------------------------------------
    if(mongo.is.connected(con) == FALSE) stop("Mongo connection has been terminated.")


    ns_collections <- mongo.get.database.collections(con, ns_db)
    pandoc.list(ns_collections)

-   lade\_k\_nightscout.entries
-   lade\_k\_nightscout.devicestatus
-   lade\_k\_nightscout.treatments

<!-- end of list -->
The next code chunk will produce some vectors needed to hold the
Nightscout data before it is turned into a data frame. When importing
data from a RDBMS, it is normal practice to place the imported data
directly into a R data frame. When importing data from Mongo, the data
must first be placed into vectors. This is further complicated by the
fact that records have a different number of fields and we have to
handle the NULL values in R, rather than via the database.

The following query imports all of the data in the enties collection.
Rather than build each vector incrementally, it is faster and more
memory efficient to create vectors large enough to hold all of the data
present in the server. The variable, ns\_count, is used to hold the
number of records in the "entries" collection.

    ## Make sure we still have a connection ----------------------------------------
    if(mongo.is.connected(con) == FALSE) stop("Mongo connection has been terminated.")

    ## Collections Variables -------------------------------------------------------
    ## Yeah, I just hard-coded these. Sue me.
    ns_entries <- "lade_k_nightscout.entries"

    ## Mongo Variables -------------------------------------------------------------
    ## ns_count: Total number of records in entries.
    ## ns_cursor: A cursor variable capable of returning the valie of all fields in
    ##            a single row of the entries collection.
    ##
    ns_count   <- mongo.count(con, ns_entries)
    ns_cursor <- mongo.find(con, ns_entries)

    ## R Vectors to hold Nightscout  data ------------------------------------------
    ## If you don't define the variable type, you tend to get characters.
    device     <- vector("character",ns_count)
    date       <- vector("numeric",ns_count)
    dateString <- vector("character",ns_count)
    sgv        <- vector("integer",ns_count)
    direction  <- vector("character",ns_count)
    type       <- vector("character",ns_count)
    filtered   <- vector("integer",ns_count)
    unfiltered <- vector("integer",ns_count)
    rssi       <- vector("integer",ns_count)
    noise      <- vector("integer",ns_count)
    mbg        <- vector("numeric",ns_count)
    slope      <- vector("numeric",ns_count)
    intercept  <- vector("numeric",ns_count)
    scale      <- vector("numeric",ns_count)

As of 2015-07-28 the "entries" collection contains
`r format(ns_count, big.mark=",")` records. That is a lot of data, about
a single person. The following code chunk imports the records in
'entries' and places the data into the vectors produced above.

The "ns\_cursor" variable is a cursor, an approach which should be
familiar to web-developers. The cursor returns a single record at a
time. The use of a loop feels odd because R programming usually avoids
using loops but this does appear to be the preferred way of getting
importing data from Mongo.

    ## Get the CGM Data, with a LOOP -----------------------------------------------
    ## The examples I found on the Internet always show this loop as a while loop.
    ## Depending on my future import needs, I may need to change this to a for loop.
    ## A new record is added every five minutes, which means it is not impossible for
    ## a new data entry to be produced while this script is running, even though it
    ## takes less that a couple of seconds to run.

    i = 1

    while(mongo.cursor.next(ns_cursor)) {
        
        # Get the values of the current record
        cval = mongo.cursor.value(ns_cursor)

        ## Place the values of the record into the appropriate location in the vectors.
        ## Must catch NULLS for each record or the vectors will have different lengths when we are done.
        device[i] <- if( is.null(mongo.bson.value(cval, "device")) ) NA else mongo.bson.value(cval, "device")
        date[i] <- if( is.null(mongo.bson.value(cval, "date")) ) NA else mongo.bson.value(cval, "date")
        dateString[i] <- if(is.null(mongo.bson.value(cval, "dateString")) ) NA else mongo.bson.value(cval, "dateString")
        sgv[i] <- if( is.null( mongo.bson.value(cval, "sgv") ) ) NA else mongo.bson.value(cval, "sgv")
        direction[i] <- if( is.null( mongo.bson.value(cval, "direction") ) ) NA else mongo.bson.value(cval, "direction")
        type[i] <- if( is.null(mongo.bson.value(cval, "type") ) ) NA else mongo.bson.value(cval, "type")
        filtered[i] <- if( is.null( mongo.bson.value(cval, "filtered") ) ) NA else mongo.bson.value(cval, "filtered")
        unfiltered[i] <- if( is.null( mongo.bson.value(cval, "unfiltered") ) ) NA else mongo.bson.value(cval, "unfiltered")
        rssi[i] <- if( is.null( mongo.bson.value(cval, "rssi") ) ) NA else mongo.bson.value(cval, "rssi")
        noise[i] <- if( is.null( mongo.bson.value(cval, "noise") ) ) NA else mongo.bson.value(cval, "noise")
        mbg[i] <- if( is.null(mongo.bson.value(cval, "mbg"))) NA else mongo.bson.value(cval, "mbg")
        slope[i] <- if( is.null( mongo.bson.value(cval, "slope") ) ) NA else mongo.bson.value(cval, "slope")
        intercept[i] <- if( is.null( mongo.bson.value(cval, "intercept") ) ) NA else mongo.bson.value(cval, "intercept")
        scale[i] <- if( is.null( mongo.bson.value(cval, "scale") ) ) NA else mongo.bson.value(cval, "scale")

        ## Increment the cursor to the next record.
        i = i + 1
    }


    ## Data Clean Up ---------------------------------------------------------------

    ## Fixes the date data.
    ## I'm not sure why I have to divide by 1000. If I don't, this won't work.
    date <- as.POSIXct(date/1000, origin = "1970-01-01 00:00:01")


    ## Builds the data.frame -------------------------------------------------------
    entries <- as.data.frame(list( device = device,
                                   date = date,
                                   dateString = dateString,
                                   sgv = sgv,
                                   direction = direction,
                                   type = type,
                                   filtered = filtered,
                                   unfiltered = unfiltered,
                                   rssi = rssi,
                                   noise = noise,
                                   mbg = mbg,
                                   slope = slope,
                                   intercept = intercept,
                                   scale = scale
                                  )
                             )

Mongo allows each record to have a different number of data elements.
Thus, not all records include a "mbg" element. Thus, NULLS must be
handled by the client. Querying a collection with a rapidly changing
data structure can be a little complicated.

The next code chunk does some very minimal QA on the "entries" data
frame. If the data frame has 0 rows, it will force the script to stop.
Otherwise, it returns a table with some basic meta-data about the
imported data set.

    if(dim(entries)[1] == 0) stop("Entries variable contains no rows.")

    entries %>%
        summarize(
            "N Rows" = n(),
            "N Days" = length(unique( format.POSIXct(.$date, format="%F" ))),
            "First Day" = min( format.POSIXct(.$date, format="%F" )),
            "Last Day" = max( format.POSIXct(.$date, format="%F" ))
            ) %>%
        pander()

<table>
<colgroup>
<col width="12%" />
<col width="12%" />
<col width="16%" />
<col width="16%" />
</colgroup>
<thead>
<tr class="header">
<th align="center">N Rows</th>
<th align="center">N Days</th>
<th align="center">First Day</th>
<th align="center">Last Day</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="center">14125</td>
<td align="center">73</td>
<td align="center">FALSE</td>
<td align="center">2015-07-28</td>
</tr>
</tbody>
</table>

The following code chunk returns the number of entries recorded each day
between June 20 and June 30. Each entry is an independent intersitial
glucose reading recorded by her CGM.

    entries %>%
        filter(date >= "2015-06-20" & date <= "2015-06-30") %>%
        group_by( "Date" = format.POSIXct(.$date, format="%F") ) %>%
        summarize("N Entries" = n() ) %>%
        pander()

<table>
<colgroup>
<col width="15%" />
<col width="15%" />
</colgroup>
<thead>
<tr class="header">
<th align="center">Date</th>
<th align="center">N Entries</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="center">2015-06-21</td>
<td align="center">15</td>
</tr>
<tr class="even">
<td align="center">2015-06-22</td>
<td align="center">279</td>
</tr>
<tr class="odd">
<td align="center">2015-06-23</td>
<td align="center">261</td>
</tr>
<tr class="even">
<td align="center">2015-06-24</td>
<td align="center">275</td>
</tr>
<tr class="odd">
<td align="center">2015-06-25</td>
<td align="center">286</td>
</tr>
<tr class="even">
<td align="center">2015-06-26</td>
<td align="center">237</td>
</tr>
<tr class="odd">
<td align="center">2015-06-27</td>
<td align="center">85</td>
</tr>
<tr class="even">
<td align="center">2015-06-28</td>
<td align="center">130</td>
</tr>
<tr class="odd">
<td align="center">2015-06-29</td>
<td align="center">282</td>
</tr>
</tbody>
</table>

I set up Karen's current rig late on June 21. As a result, Karen's rig
only recorded 15 records on that 'day'. Karen believes she changed her
sensor out on June 27, which is why there are only 85 records on that
day. For some reason the rig was not reading correctly or it wasn't
communicating with the server on the 28th. The other days are fairly
consistent, but the table does demonstrate how the number of entries
recorded does vary a little each day.

The final code chunk exports a date-stamped CSV file. I'll try to add a
new data set periodically to the public data if anyone wants to use it.
Old data sets will remain frozen, for reproducibility purposes, but may
disappear at some point in the future. Don't expect there to be more
than 5 data sets in the data folder.

    ## Saves the data as a CSV file ------------------------------------------------
    ## You are welcome to use the data stored publicly in the data folder.
    file_name <- paste("data/entries-", Sys.Date(), ".csv", sep="")
    write.csv(entries, file_name, row.names = FALSE)

    ## Clean up the session and good-bye.
    rm(list=ls())