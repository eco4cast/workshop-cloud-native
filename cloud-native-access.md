cloud native access
================
2022-05-05

# Synoposis

In this tutorial we see how to use cloud-native access protocols
`s3_bucket()` + `open_dataset()`

``` r
library(tidyverse)
library(arrow)

Sys.setenv(CURLOPT_ACCEPTTIMEOUT_MS=5000) # Jetstream is laggy
```

All big commercial cloud providers (e.g. AWS, Google, Azure) provide
object stores. Amazon’s AWS S3 platform is among the most widely used,
and hosts a [large registry of open
datasets](https://registry.opendata.aws/), including archives of NOAA
global circulation models. A wide range of software tools have been
built around the AWS S3 interface, which has led to the adoption of
S3-compatible interfaces by other cloud providers (Google) and open
source platforms (CEPH, MINIO). While not an official community standard
like POSIX, this makes S3 something of a de-facto standard.

EFI uses the [Jetstream2](https://jetstream-cloud.org/) Cloud’s Object
Store, maintained by NSF XSEDE program. Unlike commercial providers,
Jetstream2 uses an open source system called
[CEPH](https://docs.ceph.com/en/quincy/) (developed by RedHat), which
supports the S3 interface. Consequently, we have to override the default
“endpoint” (which assumes it will be talking to AWS), to tell it to talk
to our server instead.  
The official Jetstream endpoint for the S3 service is
`js2.jetstream-cloud.org:8001`, however, EFI uses a simple reverse-proxy
server to provide a custom endpoint to Jetstream2,
`data.ecoforecast.org`.

Cloud-based object stores use credentials to control who has permission
to read and write data to the store. EFI sets most of it’s buckets to
public-download, so the data can be read without any credentials.
Writing data requires appropriate credentials, with the exception of the
submissions bucket, which permits anonymous uploads.

Here we’ll explore a few popular libraries that support such
‘cloud-native’ data access. First, let’s take a look at the [Apache
Arrow](https://arrow.apache.org/) framework. Note that though these
examples use R, {arrow} clients are available for most major languages,
including python, C++, javascript, Go, Rust, etc.

To access data on the object store, we establish a connection to a
desired bucket on the server. (Note that bucket names are global to S3
object stores, and thus must be unique.) The most important EFI buckets
are:

-   `scores` Contains scores of every forecast submitted, along with
    summary statistics of the forecasts.
-   `forecasts` Raw forecast files submitted to the challenge and
    validated by the submissions-checker
-   `targets` Processed NEON data into the format of variables used in
    the challenge
-   `drivers` Additional possible driver data, such as NOAA GEFS
    forecasts downscaled to NEON sites.

Here we make a connection to the the `scores` bucket. Note the endpoint
override. The `anonymous=TRUE` is optional, but make sure you do not
have any AWS credentials set up in environmental variables or in your
`~/.aws` directory. These probably correspond to your account on other
S3 platforms and will not be compatible with Jetstream2. See section on
troubleshooting below.

``` r
s3 <- s3_bucket("scores", endpoint_override = "js2.jetstream-cloud.org:8001", anonymous=TRUE)
```

This `s3` connection object has methods that let us interact with the
data over this connection. For instance, we can list contents in this
directory (see `?arrow::S3FileSystem` or [online
vignette](https://arrow.apache.org/docs/r/articles/fs.html) for
details.)

``` r
s3$ls()
```

    ## [1] "parquet"

Here we see only one directory (folder). Let’s look inside that:

``` r
s3$ls("parquet")
```

    ## [1] "parquet/aquatics"          "parquet/beetles"          
    ## [3] "parquet/phenology"         "parquet/terrestrial_30min"
    ## [5] "parquet/terrestrial_daily" "parquet/ticks"

``` r
s3$ls(recursive=TRUE) |> head() # get all sub-folders but show only first few
```

    ## [1] "parquet"                                                  
    ## [2] "parquet/aquatics"                                         
    ## [3] "parquet/aquatics/2020"                                    
    ## [4] "parquet/aquatics/2020/aquatics-2020-09-01-EFInull.parquet"
    ## [5] "parquet/aquatics/2020/aquatics-2020-10-01-EFInull.parquet"
    ## [6] "parquet/aquatics/2020/aquatics-2020-11-01-EFInull.parquet"

Note that `ls()` returns the complete path (`parquet/aquatics`, not
`aquatics`) to each element.

While we could read in individual files, `arrow` is particularly
powerful when all these files share the same *schema* (exactly the same
column names & data types). In such cases, we can establish a connection
the data across all files at once, *without* reading all that data in.
Observe how the files are organized into sub-folders by `target_id`
(aquatics, phenology, etc) and year. We can tell `arrow` about this
“partitioning”, allowing fast data filtering that need not read many of
those files.

``` r
path <- s3$path("parquet")
ds <- open_dataset(path, partitioning = c("target_id", "year"))
```

Note that opening the dataset requires a `path()` to the remote object,
which we get with the `$path()` method, not the `$ls()` method (the
latter merely returns text strings that don’t contain the actual
connection details.)

**Pro tip**\*: We could in fact have short-cut this by including the
path in our original call to `s3_bucket()`, which is happy to point at
any sub-directory of the bucket as well:

``` r
s3 <- s3_bucket("scores/parquet", endpoint_override = "data.ecoforecast.org")
ds <- open_dataset(s3, partitioning = c("target_id", "year"))
```

Or if we don’t need stuff like `$ls()` to list bucket contents, we can
use the URI-notation for a one-line access:

``` r
ds <- open_dataset("s3://scores/parquet?endpoint_override=data.ecoforecast.org", partitioning = c("target_id", "year"))
```

In this example, the individual files in this data set are in the
parquet format, but this approach may also be use The resulting `ds`
object is a remote connection to a data.frame-like object. A concept
here is that this is a *lazy* connection - it hasn’t read in any data
into R yet. If we print the object, it shows us the schema: column names
and types, following optional and required columns from the EFI metadata
standard:

``` r
ds
```

    ## FileSystemDataset with 2656 Parquet files
    ## target_id: string
    ## model_id: string
    ## pub_time: string
    ## site_id: string
    ## x: bool
    ## y: bool
    ## z: bool
    ## time: timestamp[us, tz=UTC]
    ## variable: string
    ## mean: double
    ## sd: double
    ## observed: double
    ## crps: double
    ## logs: double
    ## quantile02.5: double
    ## quantile10: double
    ## quantile90: double
    ## quantile97.5: double
    ## interval: double
    ## start_time: timestamp[us, tz=UTC]
    ## horizon: double
    ## year: int32
    ## 
    ## See $metadata for additional Schema metadata

Note we have a FileSystemDataset object made up of lots of individual
Parquet files! In addition to the standard fields in an EFI forecast,
(`time`, `site_id`, `variable`, etc) this data includes two Strictly
Proper Scoring metrics: `crps` and `logs`. We also have the
corresponding `observed` value (when avialable) that was used to compute
these scores. While many EFI forecasts are submitted as *ensembles*, the
scoring files summarize that uncertainty in terms of mean, standard
deviation, and quantiles. (Note that the scoring is still based directly
on ensemble methods, and not merely on these summary statistics.)

Viewing the schema instead of the top of the data frame may seem strange
at first, but actually is quite handy. This helps us think about what
queries we want first. Being *lazy* about reading lets our code be
faster and smarter – if we filter the data, it can skip reading in any
fields that would have been left out. All `select()` and most common
`filter()` operations will work with this remote table:

``` r
pheno <-
  ds |> 
  filter(target_id == "phenology", 
         year == 2022,
         start_time > lubridate::as_datetime("2022-04-01")) |>
  collect()

pheno
```

    ## # A tibble: 129,996 × 22
    ##    target_id model_id     pub_time site_id x     y     z     time               
    ##  * <chr>     <chr>        <chr>    <chr>   <lgl> <lgl> <lgl> <dttm>             
    ##  1 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-02 00:00:00
    ##  2 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-02 00:00:00
    ##  3 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-03 00:00:00
    ##  4 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-03 00:00:00
    ##  5 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-04 00:00:00
    ##  6 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-04 00:00:00
    ##  7 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-05 00:00:00
    ##  8 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-05 00:00:00
    ##  9 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-06 00:00:00
    ## 10 phenology PEG_FUSION_0 2022-04… BART    NA    NA    NA    2022-04-06 00:00:00
    ## # … with 129,986 more rows, and 14 more variables: variable <chr>, mean <dbl>,
    ## #   sd <dbl>, observed <dbl>, crps <dbl>, logs <dbl>, quantile02.5 <dbl>,
    ## #   quantile10 <dbl>, quantile90 <dbl>, quantile97.5 <dbl>, interval <dbl>,
    ## #   start_time <dttm>, horizon <dbl>, year <int>

Note that some more complex filters, such as those which use regex
patterns (see `?grepl`) or other R functions inside the call will not
work. Arrow is also stricter about some conventions – e.g. experienced
users might note we had to explicitly convert our date string to a
datetime object in order to use a logical `>` comparison; while dplyr
does this automatically with in-memory data.frames. After our filter, we
use the function `collect()` to signal we are ready to read our data
into R. This lets `arrow` know it can be lazy no longer, so expect this
step to be slow try and defer this operation as long as possible. In
addition to `select()` and `filter()`, arrow is now capable of executing
a wide range of group_by() and summarize() operations.

## Troubleshooting S3 Connections

Before running an `s3_bucket()` call, it can be helpful to unset the
following environmental variables.

``` r
Sys.setenv("AWS_EC2_METADATA_DISABLED"="TRUE")
Sys.unsetenv("AWS_ACCESS_KEY_ID")
Sys.unsetenv("AWS_SECRET_ACCESS_KEY")
Sys.unsetenv("AWS_DEFAULT_REGION")
Sys.unsetenv("AWS_S3_ENDPOINT")
```
