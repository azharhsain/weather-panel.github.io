# Working with shapefiles

Your first step should be to view your shapefile in a software system
like QGIS or ArcGIS.
QGIS is a free and open-source desktop geographic information system application that supports viewing, editing and analysis of geospatial data. ArcGIS is proprietary software for working with maps and geographic data.

These systems also allow you to perform many kinds of spatial
analysis. Some of the most commonly useful are calculating Zonal
Statistics, such as the average of a gridded dataset across each
shapefile region, and Merging units, and translating points to
polygons with Voronoi Polygons.

Both R and Python support working with spatial data. See the following examples:
 
````{tabbed} R
To read shape files you could use a package like `maptools`,  `rgdal`, `sf`, or `PBSmapping`.

```{code-block} R
library(maptools)
shapefile <- readShapePoly("/my_shapefile.shp")

# or

library(PBSmapping)
shapefile <- importShapefile("/my_shapefile.shp")
```
````

````{tabbed} Python

Shapefiles can be opened with Python packages in a few different ways:

```{code-block} python
import fiona
shape = fiona.open("my_shapefile.shp")
print shape.schema
{'geometry': 'LineString', \ 
'properties': OrderedDict([(u'FID', 'float:11')])}

# or

import shapefile
shape = shapefile.Reader("my_shapefile.shp")

# or

import geopandas as gpd
shapefile = gpd.read_file("my_shapefile.shp")
print(shapefile)
```
````


## Constructing averages within spatial units

Now, you probably have a gridded spatiotemporal dataset of historical
weather and economic output data specific to shapefile regions.  The
next step is construct the weather measures that will correspond to
each of your economic data observations.  To do this, you will need to
construct a weighted average of the weather in each region for each
timestep.

In some cases, there are tools available that will help you do
this. If you are using area weighting (i.e., no weighting grid) and
your grid is fine enough that every region fully contains at least one
cell, one tool you can use is
[regionmask](http://www.matteodefelice.name/post/aggregating-gridded-data/).

If your situation is more complicated, or if you just want to know how
to do it yourself, it is important to set up the mathematical process
efficiently since this can be a computationally expensive step.

The regional averaging process is equivalent to a matrix
transformation: $$w_\text{region} = A w_\text{gridded}$$

where $w_\text{region}$ is a vector of weather values across
regions, in a given timestep; and $w_\text{gridded}$ is a vector of
weather values across grid cells.  Suppose there are $N$ regions and
$R C$ grid cells, then the transformation matrix $A$ will be $N x
R C$. The $A$ matrix does not change over time, so once you
calculate it, the process for generating each time step is faster.

Below, we sketch out two approaches to generating this matrix, but a
few comments are common no matter how you generate it.

1. The sum of entries across each row should be 1. Missing values can
   cause reasonable-looking calculations to produce rows sums that are
   less than one, so make sure you check.
   
2. This matrix is huge, but it is very sparse: most entries
   are 0. Make sure to use a sparse matrix implementation (e.g.,
   `sparseMatrix` in R, `scipy.sparse` in python, `sparse` in Matlab).
   
3. The $w_\text{gridded}$ data starts as a matrix, but here we use
   it as a vector.  It is easy (in any language) to convert a matrix
   to a vector with all of its values, but you need to be careful
   about the order of the entries, and order the columns of $A$ the
   same way.
   
   In R, `as.vector` will convert from a matrix to a vector, with each
   *column* being listed in full before moving on to the next column.
   
   In python, `numpy.flatten` will convert a numpy matrix to a vector,
   with each *row* being listed in full before moving on to the next
   row.
   
   In Matlab, indexing the grid with `(:)` will convert from an array
   to a vector, with each *column* being listed in full before moving
   on to the next column.

### Version 1. Using grid cell centers

The easiest way to generate weather for each region in a shapefile is
to generate a collection of points at the center of each grid
cell. This approach can be used without generating an $A$ matrix,
but the matrix method improves efficiency.

As an example, in R, you generate these points like so:
```R
longitudes <- seq(longitude0, longitude1, gridwidth)
latitudes <- seq(latitude0, latitude1, gridwidth)
pts <- expand.grid(x=longitudes, y=latitudes)
```

Now, you can iterate through each region, and get a list of all of the
points within each region. Here's how you would do that with the
`PBSmapping` library in R:
```R
events <- data.frame(EID=1:nrow(pts), X=pts$x, Y=pts$y)
events <- as.EventData(events, projection=attributes(polys)$projection)
eids <- findPolys(events, polys, maxRows=6e5)
```

Then you can use the cells that have been found (which, if you've set
it up right, will be in the same order as the columns of $A$) to
fill in the entries of your transformation matrix.

If your regions are not much bigger than the grid cells, you may get
regions that do not contain any cell centers. In this case, you need
to find whichever grid cell is closest. For example, in R, using
`PBSmapping`:
```R
centroid <- calcCentroid(polys, rollup=1)
dists <- sqrt((pts$x - centroid$X)^2 + (pts$y - centroid$Y)^2)
closest <- which.min(dists)[1]
```

### Version 2. Allowing for partial grid cells

Just using the grid cell centers can result in a poor representation
of the weather that overlaps each region, particularly when the
regions are of a similar size to the grid cells. In this case, you
need to determine how much each grid cell overlaps with each region.

There are different ways for doing this, but one is to use QGIS.
Within QGIS, you can create a shapefile with a rectangle for each grid
cell. Then intersect those with the region shapefile, producing a
separate polygon for each region-by-grid cell combination. Then have
QGIS compute the area of each of those regions: these will give you
the portion of grid cells to use.

## Matching names

It is often necessary to match names within two datasets with geographical unit observations. For example, a country’s statistics ministry may report values by administrative unit, but to find out the actual spatial extent of those units, you may need to use the GADM shapefiles.

Matching observations by name can be annoyingly time-consuming. These problems even exist at the level of countries, where, for example, North Korea is regularly listed as “Democratic People's Republic of Korea”, “Korea, North”, and “Korea, Dem. Rep.”; and information is indiscriminately reported for isolated regions or sovereign states (Guadeloupe’s data may or may not be included in France). Reporting units may not correspond to standard administrative units at all, and you will need to aggregate or disaggregate regions to match between datasets.

```{tip}
Here are some suggestions for dealing with political geography:

1. Try to perform all merging on abbreviation codes rather than names. At the level of countries, use [ISO alpha-3 codes](https://www.nationsonline.org/oneworld/country_code_list.htm) if possible.

2. Use string matching. However, in this case, you will need to inspect all of the matches to make sure that they are correct.

3. Construct “translation functions” for each dataset, which map the regional names in that dataset to a canonical list of region names. For example, choose the names in one dataset as a canonical list, and name the matching functions as `<dataset>2canonical` and `canonical2<dataset2>`.
```