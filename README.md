Models for Exploratory Data Analysis - Clustering
================
Nov-2019

This analysis applies two common clustering techniques, k-means
clustering and hierarchical clustering, to data on house prices in
England and Wales. It is intended to demonstrate that fitting simple
models to a dataset can be a highly effective way of quickly identifying
trends and generating ideas for further analysis.

More information on this approach can be found here:
<https://r4ds.had.co.nz/model-building.html>

## How have house prices in England and Wales changed since 2008?

Data on median property prices in England and Wales is published
quarterly by the Office for National Statistics, using data from the
Land Registry. It is available here:
<https://www.ons.gov.uk/peoplepopulationandcommunity/housing/datasets/medianhousepricefornationalandsubnationalgeographiesquarterlyrollingyearhpssadataset09>

The Excel document contains house prices for different administrative
geographies of England and Wales, and different types of property. Here,
we’ll look at median prices for all property types in the c. 350 local
authorities in England and Wales.

We’ll start by reading in the data and do some basic cleaning.

``` r
library(tidyverse)
library(readxl)
library(lubridate)

options(scipen = 9999)

house_prices <- read_xls("hpssadataset9medianpricepaidforadministrativegeographies.xls",
         sheet = "2a", skip = 6) %>% 
  # Converting the data from 'wide' to long
  gather(key = "date", value = "median_price", -1, -2, -3, -4) %>% 
  # Converting date to a date format using the myd() function from the lubridate package
  mutate(date = str_replace(date, "Year ending ", "") %>% myd(truncated = 1)) %>% 
  # Looking at data since 2008
  filter(date >= dmy("01-01-2008")) %>% 
  janitor::clean_names() 

# Clean data 
house_prices 
```

    ## # A tibble: 15,660 x 6
    ##    region_country_… region_country_… local_authority… local_authority…
    ##    <chr>            <chr>            <chr>            <chr>           
    ##  1 E12000001        North East       E06000047        County Durham   
    ##  2 E12000001        North East       E06000005        Darlington      
    ##  3 E12000001        North East       E06000001        Hartlepool      
    ##  4 E12000001        North East       E06000002        Middlesbrough   
    ##  5 E12000001        North East       E06000057        Northumberland  
    ##  6 E12000001        North East       E06000003        Redcar and Clev…
    ##  7 E12000001        North East       E06000004        Stockton-on-Tees
    ##  8 E12000001        North East       E08000037        Gateshead       
    ##  9 E12000001        North East       E08000021        Newcastle upon …
    ## 10 E12000001        North East       E08000022        North Tyneside  
    ## # … with 15,650 more rows, and 2 more variables: date <date>,
    ## #   median_price <dbl>

## Visualising changes to house prices

Let’s quickly graph the changes to house prices. First, we’ll look at
absolute values.

``` r
house_prices %>% 
  ggplot(aes(date, median_price)) + 
  geom_line(aes(group = local_authority_name), alpha = 0.2) + 
  geom_smooth(se = FALSE) +
  labs(x = "", y = "Median house price", title = "Median house prices in local authorities since 2008")
```

![](README_files/figure-gfm/unnamed-chunk-2-1.png)<!-- -->

``` r
# Using a logarithmic scale 
house_prices %>% 
  ggplot(aes(date, median_price)) + 
  geom_line(aes(group = local_authority_name), alpha = 0.2) + 
  geom_smooth(se = FALSE) + 
  scale_y_log10() +
  labs(x = "", y = "Median house price (log scale)", 
       title = "Median House Prices in local authorities since 2008")
```

![](README_files/figure-gfm/unnamed-chunk-2-2.png)<!-- -->

The range of house prices across England and Wales is very large, so
we’ll look at relative change. We’ll create an index that looks at
prices relative to their level in March 2008. We’ll look at one
visualisation for all of the locations, and then faceted by region.

``` r
house_prices <- house_prices %>% 
  group_by(local_authority_name) %>% 
  mutate(median_price_index = (median_price/median_price[date == min(date)])*100) %>% 
  ungroup()

house_prices %>% 
  ggplot(aes(date, median_price_index)) + 
  geom_line(aes(group = local_authority_name), alpha =  0.2) + 
  geom_smooth(se = FALSE) + 
  labs(x = "", y = "Median house index (Mar 2008 = 100)", 
       title = "Median house prices in local authorities since 2008")
```

![](README_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

``` r
house_prices %>% 
  ggplot(aes(date, median_price_index)) + 
  geom_line(aes(group = local_authority_name), alpha =  0.2) + 
  geom_smooth(se = FALSE) + 
  facet_wrap(~region_country_name) +
  labs(x = "", y = "Median house index (Mar 2008 = 100)", 
       title = "Median house prices in local authorities since 2008")
```

![](README_files/figure-gfm/unnamed-chunk-3-2.png)<!-- -->

# K-means clustering

We’ll first use k-means clustering to identify areas which have
experienced similar patterns in their house price growth. I won’t
explain how k-means clustering works, as there are countless articles on
this around the Web already.

``` r
# Preparing the data 
house_prices_kmeans <- house_prices %>% 
  select(local_authority_name, date, median_price_index) %>% 
  spread(date, value = median_price_index) %>% 
  as.data.frame()

hp_rownames <- pull(house_prices_kmeans, local_authority_name)

house_prices_kmeans <- house_prices_kmeans %>% select(-local_authority_name)

rownames(house_prices_kmeans) <- hp_rownames

# Identifying the best number for 'k' 
set.seed(42)

total_withiness <- map_dbl(1:10,  function(k){
  model <- kmeans(x = house_prices_kmeans, centers = k)
  model$tot.withinss
})

elbow_df <- data.frame(
  k = 1:10,
  total_withiness = total_withiness
)

ggplot(elbow_df, aes(x = k, y = total_withiness)) +
  geom_line() +
  scale_x_continuous(breaks = 1:10)
```

![](README_files/figure-gfm/unnamed-chunk-4-1.png)<!-- -->

We’ll go with 4 clusters

``` r
model <- kmeans(house_prices_kmeans, centers = 4)

library(broom)

model_results <- augment(model, house_prices_kmeans) %>% 
  select("local_authority_name" = 1, "cluster" = .cluster) 

house_prices <- inner_join(house_prices, model_results, by = "local_authority_name")
```

### Visualising the k-means results

Visualising the house price changes per cluster shows that the k-means
clustering has identified areas which have experienced different
trajectories of house price growth.

``` r
house_prices %>% 
  ggplot(aes(date, median_price_index)) + 
  geom_line(aes(group = local_authority_name, col = cluster)) +
  modelr::geom_ref_line(h = 100) + 
  facet_wrap(~cluster)
```

![](README_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

Overlaying the clustering onto a map, we can see that the group with the
largest increases in house prices (group 2) are located in London (with
the one exception of Cambridge). The group with the second largest
increases in house prices (group 4) are located around London and the
South East.

In general, the further a location is from London, the less house prices
have increased since 2008. It’s worth noting that locations closer to
London (and those in London) also had higher house prices to start with.
In these locations, prices started higher than they were elsewhere in
England and Wales and have also grown faster.

``` r
library(sf)
library(leaflet)

england_wales_map <- read_sf("Local_Authority_Districts_December_2017_Full_Clipped_Boundaries_in_Great_Britain", 
        "Local_Authority_Districts_December_2017_Full_Clipped_Boundaries_in_Great_Britain") %>% 
  select(lad17cd) %>% 
  st_simplify(dTolerance = 100) %>% 
  st_transform(crs = 4326) 

england_wales_map <- england_wales_map %>% 
  inner_join(distinct(house_prices, local_authority_code, local_authority_name, cluster), 
             by = c("lad17cd" = "local_authority_code")) 

fact_pal <- colorFactor("viridis", england_wales_map$cluster)

leaflet(england_wales_map) %>% 
  addPolygons(color = "white", weight = 1, fillColor = ~fact_pal(cluster), fillOpacity = 1, 
              popup = ~local_authority_name) %>% 
  addLegend(pal = fact_pal, values = ~cluster) 
```

![](README_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

# Hierarchical clustering

We’ll now use hierarchical clustering to spot trends in the house prices
data. While this method of clustering is different to k-means, the
results should be comparable to the k-means clustering.

``` r
# Preparing the data 
house_prices_hclust <- house_prices %>% 
  select(local_authority_name, date, median_price_index) %>% 
  spread(date, value = median_price_index) %>% 
  as.data.frame()

rownames(house_prices_hclust) <- pull(house_prices_hclust, local_authority_name)
house_prices_hclust <- house_prices_hclust %>% select(-local_authority_name)

# Performing the clustering and joining with our house prices data 
dist_matrix <- dist(house_prices_hclust, method = "euclidean")

house_prices_hclust <- hclust(dist_matrix, method = "complete")
plot(house_prices_hclust)
```

![](README_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

We’ll go with four clusters again

``` r
hclust_out <- cutree(house_prices_hclust, k = 4) %>% 
  tidy() %>% 
  select("local_authority_name" = 1, "hclust" = 2)
```

    ## Warning: 'tidy.numeric' is deprecated.
    ## See help("Deprecated")

``` r
house_prices <- house_prices %>% inner_join(hclust_out)
```

### Visualising the hierarchical clustering results

``` r
house_prices %>% 
  ggplot(aes(date, median_price_index)) + 
  geom_line(aes(group = local_authority_name, color = factor(hclust))) +
  geom_smooth() + 
  modelr::geom_ref_line(h = 100) +
  facet_wrap(~hclust)
```

![](README_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

The hierarchical clustering has identify the same trend as the k-means
clustering - areas in and around London have experienced the largest
increases in house prices since 2008. It’s interesting that both methods
of clustering identified that Cambridge has experienced a ‘London-like’
pattern in its house price increases since 2008.

``` r
england_wales_map <- england_wales_map %>% 
  inner_join(distinct(house_prices, local_authority_code, hclust), 
             by = c("lad17cd" = "local_authority_code")) 

fact_pal <- colorFactor("viridis", england_wales_map$hclust)

leaflet(england_wales_map) %>% 
  addPolygons(color = "white", weight = 1, fillColor = ~fact_pal(hclust), fillOpacity = 1, 
              popup = ~local_authority_name) %>% 
  addLegend(pal = fact_pal, values = ~hclust) 
```

![](README_files/figure-gfm/unnamed-chunk-11-1.png)<!-- -->
