Processing: vacant buildings to be demolished
---------------------------------------------

This notebook reads in data on vacant buildings as of February 2019 and
October 2019 and processes it into cleaned data files for analysis. It
also adds saves GIS shapefiles for mapping.

For the analysis code, see `02_analysis.Rmd`.

### Load R libraries

``` r
library('tidyverse')
library('readxl')
library('sf')
```

### Load data

Data on vacant buildings as of February 7, 2019 was downloaded from Open
Baltimore for a [previous
story](https://github.com/baltimore-sun-data/vacants-demolish-analysis/).
Data on vacant buildings as of October 10, 2019 was provided by the
Baltimore City Department of Housing & Community Development. An updated
list of the status of vacants scheduled for demolition was also provided
by the Baltimore City Department of Housing & Community Development.

The parcel boundaries are provided on the City of Baltimore’s [Open GIS
site](http://gis-baltimore.opendata.arcgis.com/datasets/b41551f53345445fa05b554cd77b3732_0).
Note that these files are not included in the GitHub repo as they were
too large to upload.

``` r
feb <- read_csv('input/vacants_feb7_2019.csv')
oct <- read_excel('input/vacant list as of 101019.xls')
parcel <- st_read('input/Real_Property/Real_Property.shp')
```

    ## Reading layer `Real_Property' from data source `/Users/czhang/Documents/vacants_compare/share/input/Real_Property/Real_Property.shp' using driver `ESRI Shapefile'
    ## Simple feature collection with 237372 features and 81 fields
    ## geometry type:  MULTIPOLYGON
    ## dimension:      XY
    ## bbox:           xmin: 1393948 ymin: 557750.4 xmax: 1445503 ymax: 621405
    ## epsg (SRID):    2248
    ## proj4string:    +proj=lcc +lat_1=39.45 +lat_2=38.3 +lat_0=37.66666666666666 +lon_0=-77 +x_0=399999.9998983998 +y_0=0 +ellps=GRS80 +towgs84=0,0,0,0,0,0,0 +units=us-ft +no_defs

### Clean neighborhood fields and fix addresses

The neighborhood fields in the `feb` and `oct` dataframes are not
spelled the same. In order to make comparisons easier, let’s truncate
these fields to the first 10 characters. Also add an address field in
the `oct` dataframe.

``` r
feb <- feb %>% mutate(neighborhood = str_sub(tolower(neighborhood), 1, 15))
oct <- oct %>% mutate(Neighborhood = str_sub(tolower(Neighborhood), 1, 15))

# harmonize addresses for Oct.
oct <- oct %>% mutate(Address = case_when(is.na(Direction) & !is.na(`Street Attr`) ~ paste(`House Num`, `Street Name`, `Street Attr`),
                                          is.na(Direction) & is.na(`Street Attr`) ~ paste(`House Num`, `Street Name`),
                                          !is.na(Direction) & !is.na(`Street Attr`) ~ paste(`House Num`, Direction, `Street Name`, `Street Attr`),
                                          !is.na(Direction) & is.na(`Street Attr`) ~ paste(`House Num`, Direction, `Street Name`)),
                      Neighborhood = ifelse(Neighborhood == 'booth-boyd', 'boyd-booth', Neighborhood))
```

A couple of addresses in the `oct` dataframe are grouped addresses: 2463
Druid Hill Avenue and 2465 Druid Hill Avenue –&gt; 2463 Druid Hill
Avenue; 532 N Chester Street and 534 N Chester Street –&gt; 532 N
Chester Street. Let’s fix these.

``` r
oct <- oct %>% mutate(Address = case_when(Address == '2465 DRUID HILL AVE' ~ '2463 DRUID HILL AVE',
                                          Address == '534 N CHESTER ST' ~ '532 N CHESTER ST',
                                          TRUE ~ Address))
```

A few of the rows in the `feb` dataframe are duplicated by `block` and
`lot` because the coordinates provided are not exactly the same by
`block` and `lot`. Let’s fix these by keeping the first in each case

``` r
feb.fixed <- feb %>% group_by(block, lot, buildingaddress) %>% mutate(n=rank(location)) %>% filter(n!=2) %>% select(-n) %>% ungroup()
```

Merge the `feb` and `oct` dataframes in order to compare the vacant
buildings in February to those in October. Note we’ll remove duplicate
addresses with `distinct()` since each block and lot is unique.

### Merging

``` r
merged <- merge(feb.fixed %>% select(block, lot, neighborhood, buildingaddress, location) %>% 
                  distinct(), 
                oct %>% select(Block, Lot, Neighborhood, Address, Zip5) %>% 
                  distinct(), 
                by.x = c('block', 'lot'), 
                by.y = c('Block', 'Lot'), all = T)
```

### Add fields to describe the Feb - Oct change

Create the following columns: - `vacant_in_feb`: 1 if the building was
vacant in Feb, 0 otherwise - `vacant_in_oct`: 1 if the building was
vacant in Oct, 0 otherwise - `chg`: change between Feb and Oct (-1 if
removed from list; 1 if added to list; 0 if no change) -
`stayed_vacant`: 1 if the building stayed on the vacants list between
Feb and Oct, 0 otherwise - `became_vacant`: 1 if the building was newly
on the vacants list in Oct, 0 otherwise - `no_longer_vacant`: 1 if the
building was no longer on the vacant in Oct, 0 otherwise

``` r
merged.clean <- merged %>% mutate(nbrhd = ifelse(is.na(neighborhood), 
                                                 Neighborhood, neighborhood),
                                  address = ifelse(is.na(buildingaddress),
                                                  Address, buildingaddress),
                                  vacant_in_feb = ifelse(!is.na(buildingaddress), 1, 0),
                                  vacant_in_oct = ifelse(!is.na(Address), 1, 0),
                                  chg = vacant_in_oct - vacant_in_feb,
                                  stayed_vacant = ifelse(chg == 0, 1, 0),
                                  no_longer_vacant = ifelse(chg == -1, 1, 0),
                                  became_vacant = ifelse(chg == 1, 1, 0),
                                  blk = ifelse(is.na(block), Block, block),
                                  lt = ifelse(is.na(lot), Lot, lot)) %>%
  select(nbrhd, address, blk, lt, vacant_in_feb, vacant_in_oct, chg, 
         stayed_vacant, no_longer_vacant, became_vacant, location, Zip5) # Zip5 field is empty for non-October vacants; location field is empty for non-February vacants
```

Fill in neighborhoods if `nbrhd` column is blank (since the Housing
Department didn’t provide neighborhoods for all the buildings in the
October vacants list, there are a couple)

``` r
merged.clean <- merged.clean %>% merge(parcel %>% as.data.frame() %>% select(BLOCK, LOT, NEIGHBOR) %>% 
                                       distinct(), 
                                       by.x = c('blk', 'lt'),
                                       by.y = c('BLOCK', 'LOT'), 
                                       all.x = T) %>% mutate(nbrhd = ifelse(is.na(nbrhd), 
                                                                            str_sub(tolower(NEIGHBOR), 1, 15), 
                                                                            nbrhd)) %>% 
  select(-NEIGHBOR)

merged.clean <-
  merged.clean %>% extract(location, c('lat', 'lon'), '\\((.*), (.*)\\)', convert = TRUE) # add lat lon coordinates as separate columns
```

### Merging with parcels data for mapping

Merge with the parcels data to get the geometry of the buildings for
mapping purposes. Note the number of rows in the `merged.clean.parcels`
dataframe is slightly higher than in the `merged.clean` dataframe as
there are some addresses (e.g., 201 W LEXINGTON ST) with more than one
unit.

``` r
merged.clean.parcels <- merge(parcel %>% select(BLOCK, LOT, BLOCKLOT, ShapeSTAre, NEIGHBOR, NO_IMPRV),
                              merged.clean, 
                              by.x = c('BLOCK', 'LOT'),
                              by.y = c('blk', 'lt'),
                              all.y = T)

print(nrow(merged.clean.parcels))
```

    ## [1] 18096

``` r
print(nrow(merged.clean))
```

    ## [1] 18084

Some of the addresses cannot be found in the `parcel` dataframe. For the
ones that don’t have coordinates, export into a CSV and use
[`geocodeio`](https://www.geocod.io/) to get approximate coordinates for
mapping purposes.

``` r
merged.clean.parcels %>% filter(is.na(BLOCKLOT)) %>% summarise(stayed_vacant = sum(stayed_vacant),
                                                               no_longer_vacant = sum(no_longer_vacant),
                                                               became_vacant = sum(became_vacant))
```

    ## Simple feature collection with 1 feature and 3 fields (with 1 geometry empty)
    ## geometry type:  GEOMETRYCOLLECTION
    ## dimension:      XY
    ## bbox:           xmin: NA ymin: NA xmax: NA ymax: NA
    ## epsg (SRID):    2248
    ## proj4string:    +proj=lcc +lat_1=39.45 +lat_2=38.3 +lat_0=37.66666666666666 +lon_0=-77 +x_0=399999.9998983998 +y_0=0 +ellps=GRS80 +towgs84=0,0,0,0,0,0,0 +units=us-ft +no_defs
    ##   stayed_vacant no_longer_vacant became_vacant                 geometry
    ## 1            16               10            22 GEOMETRYCOLLECTION EMPTY

``` r
merged.clean.parcels %>% filter(is.na(BLOCKLOT) & is.na(lat)) %>% write_csv('input/no_coordinates_addresses.csv')
```

Open the geocoded dataframe for no coordinates and fill in for the
addresses in `merged.clean.parcels` with no location data.

``` r
geocoded_coordinates <- read_csv('input/no_coordinates_addresses_geocoded.csv')

merged.clean.parcels.2 <- merged.clean.parcels %>% merge(geocoded_coordinates %>% select(address, Latitude, Longitude), 
                                                         by = c('address'), all.x = T) %>% 
  mutate(latitude = ifelse(is.na(lat), Latitude, lat), 
         longitude = ifelse(is.na(lon), Longitude, lon)) %>% select(-Latitude, -Longitude, -lat, -lon)
```

Reproject the data to EPSG:4326 crs and calculate the coordinates for
the renaming properties from September with no location data by
computing the centroid of each parcel.

``` r
merged.clean.parcels.2.reproj <- st_transform(merged.clean.parcels.2, 4326)
merged.clean.parcels.2.reproj.centroids <- cbind(merged.clean.parcels.2.reproj, st_coordinates(st_centroid(merged.clean.parcels.2.reproj))) 
```

    ## Warning in st_centroid.sf(merged.clean.parcels.2.reproj): st_centroid
    ## assumes attributes are constant over geometries of x

    ## Warning in st_centroid.sfc(st_geometry(x), of_largest_polygon =
    ## of_largest_polygon): st_centroid does not give correct centroids for
    ## longitude/latitude data

``` r
merged.clean.parcels.2.reproj.centroids <- merged.clean.parcels.2.reproj.centroids %>% mutate(latitude = ifelse(is.na(latitude), Y, latitude),
                                                                                              longitude = ifelse(is.na(longitude), X, longitude)) %>% select(-X, -Y, -Zip5, -BLOCKLOT)
```

Output the data. For mapping purposes we will output
`merged.clean.parcels.2.reproj.centroids`. For analysis purposes, we
will output `merged.clean`.

``` r
st_write(merged.clean.parcels.2.reproj.centroids, 
         'output/vacants_spatial_oct.shp') # for mapping in Carto. See: https://baltsun.carto.com/builder/13bf9fdf-9978-4d22-b327-45118565bd0b/embed
```

    ## Warning in abbreviate_shapefile_names(obj): Field names abbreviated for
    ## ESRI Shapefile driver

    ## Writing layer `vacants_spatial_oct' to data source `output/vacants_spatial_oct.shp' using driver `ESRI Shapefile'
    ## Writing 18096 features with 15 fields and geometry type Unknown (any).

``` r
write_csv(merged.clean, 'output/vacants_feb_oct.csv')
```
