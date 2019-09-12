---
layout: postx
title:  "Zeek Dictionary Comparison: Chained vs Clustered"
date:   2019-09-14 17:02:14 -0700
categories: algo
description: "The post measures space and time in zeek (zeek.org) with fixed number of SMB package capture files."
---
## Dictionary Stats

|dictionaries <br>of max entries|10K.pcap|100K.pcap| 1M.pcap|%|
|-----|--------|---------|--------|------|
|0    |  37,991|  318,045| 424,433|55.09%|
|1    |  15,576|  128,566| 303,966|39.45%|
|2    |     201|    1,896|  31,444| 4.08%|
|3    |      26|       30|   8,227| 1.07%|
|4    |      17|       18|     775| 0.10%|
|5    |      14|       13|     957| 0.12%|
|6    |      12|       22|     319| 0.04%|
|7    |       6|        6|      27| 0.00%|
|8+   |      90|      105|     315| 0.04%| 
|total|  53,933|  448,701| 770,463|100.0%|  

## Overall memory 

|malloc'd memory (M)     |10K.pcap|100K.pcap|1M.pcap|
|------------------------|--------|---------|-------|
|zeek with chained dict  |     144|      816|  1,672|  
|zeek with clustered dict|     135|      749|  1,499|
|less memory(M)          |       9|       67|    173|
|less memory(%)          |   6.25%|    8.21%| 10.35%|

## Overall processing time

|processing time (s)     |10K.pcap|100K.pcap|1M.pcap|
|------------------------|--------|---------|-------|
|zeek with chained dict  |     .25|     2.13|  13.23|  
|zeek with clustered dict|     .24|     2.02|  12.41|
|less time (M)           |     .01|      .11|    .82|
|less time (%)           |   4.00%|    5.16%|  6.20%|

## Dictionary memory

|malloc'd memory (M)     |10K.pcap|100K.pcap|1M.pcap|
|------------------------|--------|---------|-------|
|zeek with chained dict  |      13|      104|    252|  
|zeek with clustered dict|       4|       32|     62|
|fraction of memory(%)   |     30%|      31%|    25%|

## Dictionary processing time

Use Tcp Connection dictionary as the test base.

### Successful Lookup

|entries                   | 100| 1K | 10K |100K|  1M |
|--------------------------|----|----|-----|----|-----|
|chained dict (microsec)   |    |    |     |    |     |
|clustered dict (microsec) |    |    |     |    |     |
|fraction of time(%)       |   %|   %|    %|    |     |

### Failed Lookup

|entries                   | 100| 1K | 10K |100K|  1M |
|--------------------------|----|----|-----|----|-----|
|chained dict (microsec)   |    |    |     |    |     |
|clustered dict (microsec) |    |    |     |    |     |
|fraction of time(%)       |   %|   %|    %|    |     |


## Top 10 largest dictionaries

## Top 10 longest distance dictionaries


