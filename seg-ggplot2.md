SEG - ggplot2
================
Martin Frigaard

# Motivation

This document creates the `ggplot2` heatmap in the [SEG
application](https://www.diabetestechnology.org/seg/). In earlier
versions of the application, the heatmap didn’t look quite like the DTS
folks wanted, so I decided to write up how I changed the heatmap using a
pre-made .png image.

The goal is to make the heatmaps above look something like this:

``` r
# fs::dir_ls("image")
knitr::include_graphics("image/SEG_n2083.png")
```

![](image/SEG_n2083.png)<!-- -->

What we ended up with was this:

``` r
# fs::dir_ls("image")
knitr::include_graphics("image/2018-12-20-heat_map_1.0.png")
```

![](image/2018-12-20-heat_map_1.0.png)<!-- -->

Now we have a blank background image.

``` r
# fs::dir_ls("image")
knitr::include_graphics("image/BackgroundComplete.png")
```

![](image/BackgroundComplete.png)<!-- -->

In order to do this, I need to load a few data sets from Github.

## Download and read in the data

``` r
# 5 - HEAT MAP DATA INPUTS ============= ----
# 5.0 upload AppRiskPairData.csv from github  ---- ---- ---- ----
github_root <- "https://raw.githubusercontent.com/"
app_riskpair_repo <- "mjfrigaard/SEG_shiny/master/Data/AppRiskPairData.csv"
# download to data repo
if (!file.exists("data/")) {
  dir.create("data/")
}
utils::download.file(url = paste0(github_root, app_riskpair_repo), 
                     destfile = "data/Riskpairdata.csv")
# sample measure data
samp_meas_data_rep <- "mjfrigaard/SEG_shiny/master/Data/FullSampleData.csv"
utils::download.file(url = paste0(github_root, samp_meas_data_rep), 
                     destfile = "data/Sampledata.csv")
# Read in the RiskPairData & SampMeasData
RiskPairData <- readr::read_csv(file = "data/Riskpairdata.csv")
SampMeasData <- readr::read_csv(file = "data/Sampledata.csv")
```

## The `RiskPairData`

This data has columns and risk pairs for both `REF` and `BGM`, and the
`RiskFactor` variable for each pair of `REF` and `BGM` data.

``` r
RiskPairData %>% dplyr::glimpse(78)
```

    ## Observations: 361,201
    ## Variables: 5
    ## $ RiskPairID <dbl> 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16,...
    ## $ REF        <dbl> 0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, ...
    ## $ BGM        <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, ...
    ## $ RiskFactor <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, ...
    ## $ abs_risk   <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, ...

Below you can see a sample of the `REF`, `BGM`, `RiskFactor`, and
`abs_risk` variables.

``` r
RiskPairData %>% 
  dplyr::sample_n(size = 10) %>% 
  dplyr::select(REF, BGM, RiskFactor, abs_risk)
```

    ## # A tibble: 10 x 4
    ##      REF   BGM RiskFactor abs_risk
    ##    <dbl> <dbl>      <dbl>    <dbl>
    ##  1   233   585     -1.33     1.33 
    ##  2   344   375     -0.204    0.204
    ##  3   563   189      1.31     1.31 
    ##  4   258   431     -0.888    0.888
    ##  5   459   319      0.517    0.517
    ##  6   139   429     -2.48     2.48 
    ##  7   180    75      1.40     1.40 
    ##  8    61   589     -3.50     3.50 
    ##  9    89   546     -2.96     2.96 
    ## 10     5   122     -2.56     2.56

## The `SampMeasData`.

This data set mimics a blood glucose monitor, with only `BGM` and `REF`
values.

``` r
SampMeasData %>% dplyr::glimpse(78)
```

    ## Observations: 7,857
    ## Variables: 2
    ## $ BGM <dbl> 121, 212, 161, 191, 189, 293, 130, 147, 83, 132, 146, 249, 11...
    ## $ REF <dbl> 127, 223, 166, 205, 210, 296, 142, 148, 81, 131, 155, 254, 12...

Define the `mmolConvFactor`.

``` r
# 5.1 mmol conversion factor ---- ---- ---- ---- ---- ---- ----
mmolConvFactor <- 18.01806
```

## Create `rgb2hex` function and set colors

Add the `rgb2hex` function and choose the red -\> green colors for
scale.

``` r
# 5.2 rgb2hex function ---- ---- ---- ---- ---- ---- ---- ---- ---- ----
# This is the RGB to Hex number function for R
rgb2hex <- function(r, g, b) rgb(r, g, b, maxColorValue = 255)

# 5.3 risk factor colors ---- ---- ---- ---- ---- ---- ---- ---- ----
# These are the values for the colors in the heatmap.
abs_risk_0.0000_color <- rgb2hex(0, 165, 0)
# abs_risk_0.0000_color
abs_risk_0.4375_color <- rgb2hex(0, 255, 0)
# abs_risk_0.4375_color
abs_risk_1.0625_color <- rgb2hex(255, 255, 0)
# abs_risk_1.0625_color
abs_risk_2.7500_color <- rgb2hex(255, 0, 0)
# abs_risk_2.7500_color
abs_risk_4.0000_color <- rgb2hex(128, 0, 0)
# abs_risk_4.0000_color
riskfactor_colors <- c(
  abs_risk_0.0000_color,
  abs_risk_0.4375_color,
  abs_risk_1.0625_color,
  abs_risk_2.7500_color,
  abs_risk_4.0000_color
)
```

## Base layer + color gradient

The base layer color gradient are built below. The base data frame only
exists to give the x/y coordinates at this
stage.

``` r
# 5.4 create base_data data frame ---- ---- ---- ---- ---- ---- ---- ----
base_data <- data.frame(
  x_coordinate = 0,
  y_coordinate = 0,
  color_gradient = c(0:4)
)
# 5.5 base layer ---- ---- ---- ---- ---- ---- ---- ----
base_layer <- ggplot() +
  geom_point(
    data = base_data, # defines data frame
    aes(
      x = x_coordinate,
      y = y_coordinate,
      fill = color_gradient
    )
  ) # + # uses x, y, color_gradient
base_layer
```

![](seg-ggplot2_files/figure-gfm/base_data-create-1.png)<!-- -->

## Add the risk layer

This level is where we add the risk layer with the `RiskPairData`. First
note the relationship between the values for `RiskFactor` and
`REF`/`BGM` without the base layer in the plot.

If we look at `RiskFactor` as a function of `seg_val`, we see the
following.

``` r
RiskPairData %>% 
  tidyr::gather(key = "seg_key", 
                value = "seg_val", 
                c(REF, BGM)) %>% 
  ggplot(aes(x = seg_val, y = RiskFactor, group = seg_key)) + 
    geom_point(aes(color = seg_key), alpha = 1/8, size = 0.5)
```

![](seg-ggplot2_files/figure-gfm/RiskFactor-REF-BGM-1.png)<!-- -->

Now we will plot the `abs_risk` as a function of `seg_val` values.

``` r
RiskPairData %>% 
  tidyr::gather(key = "seg_key", 
                value = "seg_val", 
                c(REF, BGM)) %>% 
  ggplot2::ggplot(aes(x = seg_val, y = abs_risk, group = seg_key)) + 
    ggplot2::geom_point(aes(color = seg_key), 
                        alpha = 1/8, size = 0.5)
```

![](seg-ggplot2_files/figure-gfm/abs_risk-REF-BGM-1.png)<!-- -->

``` r
RiskPairData %>% 
  ggplot2::ggplot(aes(x = REF, 
                      y = BGM,
                      color = abs_risk)) + 
  ggplot2::geom_point()
```

![](seg-ggplot2_files/figure-gfm/risk-layer-no-base_layer-1.png)<!-- -->

This sets the `abs_risk` as the color gradient, vs. manually setting it
with the `base_layer` below.

``` r
# 5.6 risk pair data layer  ---- ---- ---- ---- ---- ---- ---- ----
# RiskPairData %>% glimpse(78)
risk_layer <- base_layer +
  geom_point(
    data = RiskPairData, # new data set
    aes(
      x = REF, # additional aesthetics from new data set
      y = BGM,
      color = abs_risk
    ),
    show.legend = FALSE
  )
risk_layer
```

![](seg-ggplot2_files/figure-gfm/risk_layer_gradient-create-1.png)<!-- -->

Note the less than smooth transitions between the x and y coordinates.

## The Gaussian smoothed image

The goal is to get something more like this:

``` r
# fs::dir_ls("image")
knitr::include_graphics(path = "image/BackgroundComplete.png")
```

![](image/BackgroundComplete.png)<!-- -->

But without these as a backdrop, the heatmap image is rendered with the
following
appearance.

``` r
# 5.7 add fill gradient  ---- ---- ---- ---- ---- ---- ---- ---- ---- ----
risk_level_color_gradient <- risk_layer +
  ggplot2::scale_fill_gradientn( # scale_*_gradientn creats a n-color gradient
    values = scales::rescale(c(
      0, # darkgreen
      0.4375, # green
      1.0625, # yellow
      2.75, # red
      4.0 # brown
    )), 
    limits = c(0, 4),
    colors = riskfactor_colors,
    guide = guide_colorbar(
      ticks = FALSE,
      barheight = unit(100, "mm")
    ),
    breaks = c(
      0.25,
      1,
      2,
      3,
      3.75
    ),
    labels = c(
      "none",
      "slight",
      "moderate",
      "high",
      "extreme"
    ),
    name = "risk level"
  )
risk_level_color_gradient
```

![](seg-ggplot2_files/figure-gfm/risk_level_color_gradient-1.png)<!-- -->

``` r
# 5.8 add color gradient  ---- ---- ---- ---- ---- ---- ---- ---- ---- ----
# Add the new color scales to the scale_y_continuous()
heatmap_plot <- risk_level_color_gradient +
  ggplot2::scale_color_gradientn(
    colors = riskfactor_colors, # these are defined above with rgb2hex function
    guide = "none",
    limits = c(0, 4),
    values = scales::rescale(c(
      0, # darkgreen
      0.4375, # green
      1.0625, # yellow
      2.7500, # red
      4.0000
    ))
  ) +
  ggplot2::scale_y_continuous(
    limits = c(0, 600),
    sec.axis =
      sec_axis(~. / mmolConvFactor,
        name = "measured blood glucose (mmol/L)"
      ),
    name = "measured blood glucose (mg/dL)"
  ) +
  scale_x_continuous(
    limits = c(0, 600),
    sec.axis =
      sec_axis(~. / mmolConvFactor,
        name = "reference blood glucose (mmol/L)"
      ),
    name = "reference blood glucose (mg/dL)"
  )
heatmap_plot
```

![](seg-ggplot2_files/figure-gfm/build-heatmap_plot-1.png)<!-- -->

``` r
# 5.11 add SampMeasData to heatmap_plot ---- ---- ---- ---- ---- ---- ----
# sample measure data import ---- 
# Add the data to heatmap_plot
heat_map_1.0 <- heatmap_plot +
  geom_point(
    data = SampMeasData, # introduce sample data frame
    aes(
      x = REF,
      y = BGM
    ),
    shape = 21,
    fill = "white",
    size = 1.2,
    stroke = 1
  )
heat_map_1.0
```

![](seg-ggplot2_files/figure-gfm/heat_map_1.0-1.png)<!-- -->

## Convert Gaussian image to ggplot2 background

This can be done in two ways. The first comes from the `magik`
package.

``` r
library(magick)
```

``` r
# export heat_map_1.0 -----------------------------------------------------
ggplot2::ggsave(filename = 
                  base::paste0("Image/", 
                               base::noquote(lubridate::today()),
                               "-heat_map_1.0.png"), 
                device = "png")
```

## Smoothing the heatmap

### Option 1

This comes from this
[stackoverflow](https://stackoverflow.com/questions/48497419/smooth-heatmap-how-to-create-them)
question.

``` r
library(RColorBrewer)
# Color:
rf <- colorRampPalette(rev(RColorBrewer::brewer.pal(11,'Spectral')))
r <- rf(32)

# Data:
x <- rnorm(mean = 1.5, 5000)
y <- rnorm(mean = 1.6, 5000)
df <- data.frame(x,y)

#Plot:
p <- ggplot(df, aes(x,y))
h3 <- p + stat_bin_hex(bins = 25) + scale_fill_gradientn(colours = r)
h3
```

![](seg-ggplot2_files/figure-gfm/RColorBrewer-1.png)<!-- -->

The answer I am interested in comes from
\[@Axeman\](<https://stackoverflow.com/users/4341440/axeman>)

> Use `p + stat_density_2d(aes(fill = ..density..), geom = 'raster',
> contour = FALSE, n = 500) + scale_fill_gradientn(colours=r)`, as
> \[@MrSmithGoesToWashington\](<https://stackoverflow.com/users/9088378/mrsmithgoestowashington>)
> suggested. (I would preferably use a `viridis` color scale to avoid
> the banding though.) Also, that plot is not `h3` since it doesn’t have
> hex bins.

These changes produce this graph.

``` r
p + stat_density_2d(aes(fill = ..density..), 
                    geom = 'raster', contour = FALSE, n = 500) + 
                          scale_fill_gradientn(colours = r)
```

![](seg-ggplot2_files/figure-gfm/alterntive-stat_density_2d-1.png)<!-- -->

That’s a beauty\!

To adapt the `risk_layer_gradient` to this graph, I need to see what the
underlying data were for this graph.

``` r
df %>% 
  skimr::skim_to_wide() %>% 
  dplyr::select(variable, 
                mean,
                sd, 
                p50,
                hist)
```

    ## # A tibble: 2 x 5
    ##   variable mean  sd    p50    hist    
    ##   <chr>    <chr> <chr> <chr>  <chr>   
    ## 1 x        1.53  1.01  1.52   ▁▁▂▇▇▅▁▁
    ## 2 y        1.61  0.99  "1.6 " ▁▁▅▇▇▂▁▁

This image has `5000` pixels per square inch.

## The `stat_density_2d` geom

This is covered in depth on the `ggplot2`
[website](https://ggplot2.tidyverse.org/reference/geom_density_2d.html).

``` r
geom_density_2d(mapping = NULL, data = NULL, stat = "density2d",
  position = "identity", ..., lineend = "butt", linejoin = "round",
  linemitre = 10, na.rm = FALSE, show.legend = NA,
  inherit.aes = TRUE)
```

``` r
stat_density_2d(mapping = NULL, data = NULL, geom = "density_2d",
  position = "identity", ..., contour = TRUE, n = 100, h = NULL,
  na.rm = FALSE, show.legend = NA, inherit.aes = TRUE)
```

The first example uses the `faithful` data set and starts by creating a
basic scatter plot. The variables in the scatter are summarized below.

``` r
faithful %>% 
  dplyr::select(eruptions, waiting) %>% 
  skimr::skim_to_wide() %>% 
  dplyr::select(variable,
                n,
                mean,
                sd,
                hist)
```

    ## # A tibble: 2 x 5
    ##   variable  n     mean    sd      hist    
    ##   <chr>     <chr> <chr>   <chr>   <chr>   
    ## 1 eruptions 272   " 3.49" " 1.14" ▇▃▁▁▂▅▇▃
    ## 2 waiting   272   "70.9 " 13.59   ▂▅▃▂▅▇▆▂

The plot for `eruptions` vs. `waiting` is
below.

``` r
point_plot <- ggplot2::ggplot(data = faithful, aes(x = eruptions, y = waiting)) +
           geom_point() + 
                xlim(0.5, 6) +
                    ylim(40, 110)
point_plot
```

![](seg-ggplot2_files/figure-gfm/point_plot-1.png)<!-- -->

Then if I add the `geom_density_2d` to the scatter plot, I see the graph
below.

``` r
point_plot + 
  geom_density_2d()
```

![](seg-ggplot2_files/figure-gfm/point-geom_density_2d-1.png)<!-- -->

By specifying the

``` r
point_plot + 
  stat_density_2d(aes(fill = stat(level)), geom = "polygon")
```

![](seg-ggplot2_files/figure-gfm/add-geom-polygon-1.png)<!-- -->

Get a small sample of the `diamonds` data set.

``` r
set.seed(4393)
DiamondsSmall <- diamonds[base::sample(base::nrow(diamonds), 1000), ]
DiamondsSmall %>% glimpse(78)
```

    ## Observations: 1,000
    ## Variables: 10
    ## $ carat   <dbl> 1.01, 0.90, 0.23, 1.14, 0.31, 0.81, 0.30, 0.32, 1.11, 0.7...
    ## $ cut     <ord> Very Good, Premium, Ideal, Ideal, Premium, Very Good, Pre...
    ## $ color   <ord> H, I, G, G, H, E, G, H, E, D, G, F, G, I, E, H, E, E, D, ...
    ## $ clarity <ord> VVS2, VS2, SI1, VS2, VVS2, SI2, IF, VS1, VS2, VVS2, VS2, ...
    ## $ depth   <dbl> 63.3, 61.8, 62.1, 62.0, 61.4, 63.7, 62.9, 58.9, 58.5, 62....
    ## $ table   <dbl> 57, 59, 53, 55, 58, 57, 58, 63, 58, 60, 57, 55, 57, 56, 5...
    ## $ price   <int> 6097, 3398, 364, 7371, 625, 2735, 956, 624, 7480, 3799, 8...
    ## $ x       <dbl> 6.39, 6.15, 3.93, 6.68, 4.32, 5.86, 4.25, 4.46, 6.77, 5.6...
    ## $ y       <dbl> 6.35, 6.21, 3.96, 6.77, 4.35, 5.94, 4.21, 4.40, 6.84, 5.7...
    ## $ z       <dbl> 4.03, 3.82, 2.45, 4.17, 2.66, 3.76, 2.66, 2.61, 3.98, 3.5...

``` r
base_diamonds_plot <- ggplot2::ggplot(data = DiamondsSmall, aes(x = x, y = y))
base_diamonds_plot
```

![](seg-ggplot2_files/figure-gfm/base_diamonds_plot-1.png)<!-- -->

If you map an aesthetic to a categorical variable, you will get a set of
contours for each value of that variable.

``` r
base_diamonds_plot + 
  ggplot2::geom_density_2d(aes(color = cut))
```

![](seg-ggplot2_files/figure-gfm/color-cut-1.png)<!-- -->

Similarly, if you apply faceting to the plot, contours will be drawn for
each facet, but the levels will calculated across all facets

``` r
base_diamonds_plot + 
  ggplot2::stat_density_2d(aes(fill = stat(level)), geom = "polygon") + 
  ggplot2::scale_fill_viridis_c()
```

![](seg-ggplot2_files/figure-gfm/unnamed-chunk-3-1.png)<!-- -->

``` r
base_diamonds_plot + 
  stat_density_2d(aes(fill = stat(level)), geom = "polygon") +
  facet_grid(. ~ cut) + 
  scale_fill_viridis_c()
```

![](seg-ggplot2_files/figure-gfm/nlevel-vs-level-1.png)<!-- -->

``` r
base_diamonds_plot + 
  stat_density_2d(aes(fill = stat(nlevel)), geom = "polygon") +
  facet_grid(. ~ cut) + 
  scale_fill_viridis_c()
```

![](seg-ggplot2_files/figure-gfm/nlevel-vs-level-2.png)<!-- -->

``` r
base_diamonds_plot + 
  stat_density_2d(geom = "raster", 
                  aes(fill = stat(..density..)), 
                  contour = FALSE)
```

![](seg-ggplot2_files/figure-gfm/raster-image-1.png)<!-- -->

``` r
risk_base_plot <- ggplot2::ggplot(data = RiskPairData, aes(
      x = REF, # additional aesthetics from new data set
      y = BGM,
      color = abs_risk))
risk_base_plot
```

![](seg-ggplot2_files/figure-gfm/risk_base_plot-1.png)<!-- -->

This can be converted with the `raster` with `stat_density_2d`

``` r
risk_base_plot + 
  stat_density_2d(geom = "raster", 
                  aes(fill = stat(..density..)), 
                  contour = FALSE)
```

![](seg-ggplot2_files/figure-gfm/risk_base_plot-stat_density_2d-1.png)<!-- -->
