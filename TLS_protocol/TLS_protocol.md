# A *Working* Protocol for Processing Lidar Scans

Jeffery Cannon

# Introduction

This is a *working* protocol for processing terrestrial lidar scans that we collect.
By *working* protocol I mean that all aspects of the process can be improved! 
We can all find ways to amend and improve it. **Please speak up** if you have
suggestions on how to make the protocol more user-friendly, more efficient, or more accurate!
And also let us know if you run into issues.

# Collecting Terrestrial Lidar Scans

## Planning a scan [in progress]

* scan spacing, "lassoing", other considerations
* software for selecting points

## Collecting Scans [in progress]

* scanner settings, naming projects, ensuring things are all stitch-able

## Stitching scans using RiScan [in progress]

* Overview
* Step-by-step

* Is there a way to judge particularly bad scans Tanner?
* Is there a way to stitch tile and export Tanner?

## Storing/Saving/Backing up [in progress]

* Google Drive

# Pre-processing scans

### Background

The terrestrial lidar scans we collect have a few issues we need to deal with.

First, scans can include lidar point returns up to 400 m, but only the
first ~60 m of returns is usable. After that, returns are too sparse. Further,
small errors in alignment (like a slight tilt) are magnified at larger distances

<img src=img/raw_scan.PNG width=400 alt="Raw lidar scan showing rapid decay of return density with distance from scanner"></img>

Second, lidar scans are taken from a fixed point and a single sscan can only see
one side of an object. Multiple scans must be combined to capture all sides
of objects.

<img src=img/underside_raw_scan.PNG width=400 alt="View beneath a tree from a single scan showing C shaped pattern of returns from one side of the tree"></img>

Third, lidar scans are huge. A small area (0.16 ha or 40 X 40 m area) typically
comprises nearly 1GB of data. Proccessing this takes a lot of computer memory, so we must
break the data into tiles (or `lidR::chunks`) to process a bit at a time.

Forth, when dealing with lidar data there are often artefacts to deal with including
noise points introduced during collection, and misregistration errors that can
occur. 

Each of these issues can be dealt with by pre-processing the scans with the following
steps.

1. **Merging Scans**: COmbining all co-located scans to see all angles of objects
1. **Tiling scans**: Chopping large areas into smaller 'tiles' that to use less memory
during processing
1. **De-noising scans**: removing scanner artefacts and stray returns from smoke or humidity
1. **Repairing registration errors**: Identifying any scans that are misaligned can lead
 to duplicate or 'phantom' trees that need to be removed.

## Install all software

We'll mostly be using `R` Statistical software for processing lidar data. The
 functions in `R` come from small bits of code (or packages) contributed
by other `R` users. You will need to download and install these packages once on each computer you use for processing data. We will be downloading and installing some "unauthorized"
`R` packages, and for that you'll need to have `Rtools` installed on your computer

### Install `Rtools`

* [**Download and install `Rtools`**](https://cran.r-project.org/bin/windows/Rtools/rtools40.html)
* Be sure to follow the directions at the bottom regarding Putting Rtools on the PATH. This is critical for a proper installation.
* create a text file using notepad that contains the following line: `PATH="${RTOOLS40_HOME}\usr\bin;${PATH}"`
* Save the file as `.Renviron` in your Documents folder.

### Install necessary `R` packages

Once installed you can install the `R` packages you'll need using the following code:

```{r eval = FALSE}
# Run this once on each computer

# main package for dealing with lidar data in R
install.packages('lidR', repos="https://cloud.r-project.org")
# fast software for handling GIS data
install.packages('sf', repos="https://cloud.r-project.org")
# software for installing `unauthorized` R pakcages
install.packages('devtools', repos="https://cloud.r-project.org")

```

The remaining packages are not on CRAN, but hosted by users on github pages.
The `devtools` package allows us to install these directly.

```{r eval=FALSE}
# Forestry specific Lidar functions for TLS
devtools::install_github('tiagodc/TreeLS')
# good tree detection software (our modified version of spanner)
devtools::install_github('jbcannon/spanner')
# functions for our labs  common tasks
devtools::install_github('jbcannon/landecoutils') 

```

## Merging and tiling scans 

### Procedure: Merge and tile large area

To merge, clip, and tile scans from a large plot (> 0.2 ha) use the following R Code

* `ctg` Path to a folder containing all *.las files for a plot to be combined
* `out_dir`
* `bnd` A shapefile of the plot boundary. Note in this example we subset to a specific plot
* `tile_size` side length of tiles that the data is cut into
* `buffer` width of a buffer around the plot boundary `bnd`. This is included
to reduce edge effects (e.g., overhanging crowns). By default `buffer=10` but
is an optional argument
* `max_scan_distance` maximum radius from scanner within which lidar returns are
included. Defaults to `max_scan_distance = 60` which is working for us.
* `scan_locations` this is an optional input of an `sf` object containing the coordinates
for all scans. This is useful for quickly identifying for a particular region,
which scans are within the `max_scan_distance` for inclusion. Scan locations
can be passed from `landecoutils::find_ctg_centroids`. If `NULL`
the scan locations will be computed within the `stitch_TLS...` function. It's 
just provided separately to increase effeciency if the same `ctg` is used more 
than once.

```{r, eval=FALSE}
# load required packages
library(lidR)
library(sf)
library(landecoutils)

#bring up the help file for the main function used...
?stitch_TLS_dir_to_LAS_tiles
?find_ctg_centroids

# Path to folder containing the scans
ctg = readLAScatalog('E:/ecofor_segmentation/raw_scans/Plot_3') 

# shapefile indicating Plot boundary. Here we have to subet Plot 3
bnd = st_read(
  'R:/landscape_ecology/projects/eco_for/overstory_pattern/shp/ecoforplots.shp')
bnd = subset(bnd, PlotID==3) #if needed, subset feature by some attribute to get a single polygon

# Make sure bnd has same crs as ctg
bnd = st_transform(bnd, st_crs(ctg))

# This function finds the "center" of all scans so we can delete points > 60 m away
scan_locations = find_ctg_centroids(ctg)

stitch_TLS_dir_to_LAS_tiles(ctg = ctg,
                            out_dir = 'E:/ecofor_segmentation/tiled_scans/Plot_3',
                            bnd = bnd,
                            tile_size = 30, 
                            buffer = 10, #defaults to 10
                            max_scan_distance = 60, # defaults to 60
                            scan_locations = scan_locations)

```

<img src=img/stitch_TLS_dir_to_LAS_tiles.PNG width=400 alt="Demonstration of stitch_TLS_dir_to_LAS_tiles function showing scan locations
(blue), over study boundary (thick rectange), and tiled grid of study
area."></img>

The function works by automatically setting up a tiled grid over the study boundary
(`bnd`) and `buffer` you provide as specified by `tile_size`. For each tile in the
grid, the function loads and clips any scans overlapping the tile (blue circles)
and merges them together. Depending on the scanner used, some tiles have different
column names and the function rectifies that by choosing data common to all scans.
Each tile is saved in the directory specified by `out_dir` with file name basedon
on the `XY` coordinate of the bottom left corner of each tile.

<img src=img/stitched_tiles_output.png width=600 alt="Example output of stitch_TLS_dir_to_LAS_tiles with 30 m tile size"></img>

### Procedure: Merge small area (no tiling)

For smaller areas (e.g., a fixed 18 m radius plot) you may be able to avoid tiling
the area, and just merge together multiple scans into a single *.las file. For this
use the similar `landecoutils::stitch_TLS_dir_to_LAS`.

Inputs are the same as above for `stitch_TLS_dir_to_LAS_tiles` except that the plot
boundary or region of interest is defined by `roi` rather than `bnd` and there is
no need to specify a `tile_size`. A small `buffer` (usually 10 m) is still specified
to account for edge effects.

```{r eval=FALSE}
library(lidR)
library(sf)
library(landecoutils)

#bring up the help file for the main functions used...
?stitch_TLS_dir_to_LAS
?find_ctg_centroids

# Path to folder containing the scans of a small plot area 
ctg = readLAScatalog('E:/ecofor_segmentation/raw_scans/Plot_11') # Path to folder containing the scans

# Load a shapefile with the area of interest (e.g., ~0.2 ha)
bnd = st_read('R:/landscape_ecology/projects/eco_for/click_mapping_2021/click_mapping_ground_truth/grid-cells.shp')
bnd = subset(bnd, Plot==11 & cell == 90) #subsets bnd to only the Plot and cell of interest.
# Make sure bnd has same crs as ctg
bnd = st_transform(bnd, st_crs(ctg))

scan_locations = find_ctg_centroids(ctg)
stitch_TLS_dir_to_LAS(ctg = ctg,
                      out_las = 'E:/ecofor_segmentation/ground_truth_scans/Plot11_cell90.las', #where to write the .las
                      roi = bnd,
                      buffer = 10,
                      max_scan_distance = 60,
                      scan_locations = scan_locations)
```

## Cleaning up scans

### Noise removal

Some scans will include meaningless returns (noise) that are easily noticeable
when viewed in CloudCompare with an increased point size. These noisy returns come
from smoke, water vapor, or scan detector issues and will need to be removed. Lidar
returns with very low intensity (e.g., $< 4000$) are often noise and can be removed.
We also use a noise filter found in the `lidR::classify_noise()` function. It uses
an isolated voxel filter (`ivf()`) to remove all points that don’t have a certain
number of points (`n`) in the neighboring voxels with size specified by `res`. I
have found that `ivf(res = 0.1, n = 15)` works for most issues encountered to date.

<img src=img/denoise.png width=400 alt="Example of tiled TLS scan with noise to be removed"></img>

#### Procedure: Noise removal on single file

```{r eval=FALSE}
library(lidR)
# Code to do noise removal after merging (especially 2020 TallTimbers scans)
las_fn = 'E:/ecofor_segmentation/ground_truth_scans/Plot11_cell90.las'
new_fn = 'E:/ecofor_segmentation/ground_truth_scans/Plot11_cell90_denoised.las'
las = readLAS(las_fn)
las = filter_poi(las, Intensity < 4000) #18 is the LAS specification for noise
las = classify_noise(las, ivf(res=0.1,n=15)) #This setting seems to work for our TT scans with noise
las = filter_poi(las, Classification !=18) #18 is the LAS specification for noise
writeLAS(las, new_fn) #overwrite the old file with the filtered one.
```

#### Procedure: Noise removal on multiple files using pattern matching

You can remove noise from a single file at a time, or you can use a `for` loop
in `R` to denoise an entire group at a time. Here is an example of de-noising all
files in a folder with “Plot6_” in their filename.  Replace `pattern` to pattern
match your files of interest. **Test out the `ivf` settings on a single file first
as this code overwrites original tiles after denoising.**

```{r eval=FALSE}
library(lidR)
fn = list.files(out_dir, pattern='Plot6_', full.names=TRUE)
fn = grep('.las', fn, value=TRUE)
for(f in fn) {
  cat('working on ', basename(f),'...\n')
  x = lidR::filter_poi(lidR::readLAS(f), Intensity>4000)
  x = lidR::filter_poi(lidR::classify_noise(x, lidR::ivf(res=0.1, n=15)),
                       Classification!=18)
  lidR::writeLAS(x, f, index=TRUE)
}
```

### Misregistration

Because each area of interest is composed of two or more scans from different angles,
sometimes not everything lines up perfectly. If a scan is misaligned it can lead
to trees looking much thicker than they really are, or if more severe, lead to the
duplication of some trees! Dealing with misregistration generally means removing
the scan that is misaligned from the other merged scans. *This may be easier done
at the earlier stage of stitching scans in RiScan*. However, they can also be
removed later using the following procedure and code

<img src=img/misregistration_errors.png width=600 alt="Misregistration due to wind movement. Note that trunk (white error) has no issues but small branches (red arrows) are apparently misaligned"></img>

Note that not all mis-alignments are due to registration. Wind can move plant parts
between scans and make it appear as though the scan has minor misregistation. In
the following screenshot you can see that small branches appear to be misaligned,
but large tree bowls do not. Another clue that this is from wind is that the “phantom”
branches come from two different scans (blue and gold) so it is likely the branch
moved between the scans.

The effect of this kind of misregistration is minimal, as we are generally focused
on larger trees and trunks which don't move in the wind. So you can ignore this type
of registration error.

<img alt="Misregistration due to wind movement. Note that trunk (white error) has no issues but small branches (red arrows) are apparently misaligned" src=img/misregistration_wind.png width=400></img>

#### Procedure: Removing misregistration errors

The basic idea here is to find misregistered portions of scans, then isolate and
remove the points coming from the offending scan. Here we identify bad portions
of scans in CloudCompare, then remove them with R (though you can also remove in 
CC).

In CloudCompare find the areas of the scan that have misalginment. Set the Scalar
Field to `gpstime` which logs the time at which each return was captured. Generally
returns from a single scan are all grouped in within a short time. Adjust the 
display ranges of the `gpstime` attribute until you've identified the returns you
want to remove. In the case below the green points need to be removed. These are
prepresented by 3 peaks in the display range window.

<img alt="Using the gpstime attribute to isolate duplicated and misregistered returnsin CloudCompare" src=img/misregistration_gpstime.png width=600></img>

<img alt="Histogram of  gpstime attribute and area to be filtered out" src=img/gpstime_histogram.png width=400></img>

Now using R, you can load the tile, view a histogram of the `gpstime` attribute,
note the range of times to remove, and filter them out using the following code
as an example. Use the `filter_poi` function and create a gpstime function to 
keep only the appropriate range. Note the the values may not match numerically
because CloudCompare rescales some attributes. So look at the histogram and find
the range that shape that corresponds. Usually the time ranges are in short spikes
representing when the scanner is running and breaks during transportation.

Within the `filter_poi()` function, you'll need to write a logical
statement to remove the correct pixels. `&` is the `AND` operator and `|` is the
`OR` operator. See other example logical operations below

```{r eval=FALSE}
library(lidR)
x = readLAS('E:/ecofor_segmentation/ground_truth_scans/Plot5_cell7.las')
hist(x$gpstime, breaks=1e3)
x = filter_poi(x, gpstime < 400 | gpstime > 700)
writeLAS(x, 'E:/ecofor_segmentation/ground_truth_scans/Plot5_cell7.las', index=TRUE)
```

Examples of logical operators you may use.

* Keep only `gpstime` below 700: `gpstime < 700`
* Keep only `gpstime` above 200: `gpstime > 200`
* Keep `gpstime` outside of the range of 400-700: `gpstime < 400 | gpstime > 700`
* Keep only returns if `gpstime` is outside fo 400-700 or outside of 100-200:
`(gpstime < 100 | gpstime > 200) & (gpstime < 400 | gpstime > 700)`

# Stem mapping

You have made it! Once scans are planned, collected, and pre-processed you have 
clean Lidar tiles that are ready to be processed. The stem mapping portion of the
protocol should be considered the **most tentative**. It will more consistently
change as we try new algorithms, and individual project goals change. But it is 
always useful to have a basic stem map for many of our scans. 

Here is a summary of the main steps used for stem mapping, though it's best to
view the code directly to see the steps.

## Overview

1. **Normalization**: First we load the data and use a cloth surface filter which,
put simply flips a lidar scan upside down and "drapes" a surface over the bottom
to generate an elevation model. The elevation is then subtracted from each point
to flatten out and remove elevation from the scan.

<img src=img/zhang2016_csf.png width=400 alt="Diagram of cloth simulation filter (Figure from Zhang et al. 2016 Remote Sensing)"></img>

2. **Filter**: In order to see through the vegetation, we filter out all low
intensity points. This removes vegetation to reveal the tree boles and also
reduces the point density speeding up later processing.

<img src=img/bole_intensity_filter.png width=600 alt="Example of intensity filter used to remove most vegetation and reveal tree boles"></img>

3. **Cylinder search** Most tree finding algorithms take a horizontal 'slice'
of the data and try to fit cylinders around the stems that it finds. `spanner` does
just that, but it also uses information on vertical alignment of points to help
find trees. No algorithm is perfect. The settings below have done the best at
capturing smaller trees.

<img src=img/tls_slice.png width=600 alt="example of horizontal slice taken from 1 to 2 m to use for cylinder fitting in spanner"></img>

4. **Tree segmentation** Now that we know where the tree are, we need to associate
all of the points that belong to the same tree together. The `spanner` package
does this using graph theory. Trees are "grown" from their location on the ground
and nearby points are successively added until all points are assigned.

<img src=img/tree_seg_spanner.png width=600 alt="Example TLS scan with high sapling density segmented with spanner"></img>

5. **Cleanup** Lastly, we do a few housekeeping steps. Whole or partial trees
near the scan edges may not have been successfully assigned (edge effects) so we
remove them. If the user specifies, we also export individual las files containing
the segmented trees. 

Take look at the code below and see if you can identify where each of these
steps take place. And find a few that I glossed over.

## Procedure: Stem mapping

To stem map TLS tiles, use the following code. This works on small tiles, but
you'll need to use something like catalog apply for large areas.

```{r eval=FALSE}
install_github('tiagodc/TreeLS')
install_github('jbcannon/spanner') #slightly modified version of spanner
install_github('jbcannon/landecoutils')
library(lidR)
library(sf)
library(TreeLS)
library(landecoutils)
library(spanner)

?segment_with_spanner

bnd = sf::st_read('R:/landscape_ecology/projects/eco_for/click_mapping_2021/click_mapping_ground_truth/grid-cells.shp')
bnd = subset(bnd, Plot == 1 & cell == 46)
nCores=4
in_las = 'E:/ecofor_segmentation/ground_truth_scans/Plot1_cell46.las'
out_map = 'C:/Users/jeffery.cannon/OneDrive - Joseph W. Jones Ecological Research Center/Desktop/test.shp'
tree_seg = 'C:/Users/jeffery.cannon/OneDrive - Joseph W. Jones Ecological Research Center/Desktop/testseg.las'
landecoutils::segment_with_spanner(las = in_las, bnd=bnd, stemmap_shp = out_map, tree_seg_las = tree_seg)

```
