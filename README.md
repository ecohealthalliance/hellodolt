Hello, Dolt!
================
Noam Ross
9/1/2021

``` r
library(DBI)
library(RMariaDB)
library(duckdb)
library(sys)
library(nycflights13)
library(withr)
library(arrow)
library(dplyr)
library(dbplyr)
library(bench)
library(dbx)
library(waldo)
library(vroom)
knitr::opts_chunk$set(error = FALSE)
```

# Setting up a Dolt Database

Install dolt from source. The latest source version is needed

``` bash
git clone git@github.com:dolthub/dolt.git
cd dolt/go && go install ./cmd/dolt ./cmd/git-dolt ./cmd/git-dolt-smudge
```

Check current version (shell)

``` r
exec_wait("~/go/bin/dolt", "version")
```

    ## dolt version 0.28.2

    ## [1] 0

<!-- Kill stuff from previous runs -->

. ## Start a server and connect to it

``` r
dir.create("doltdb", showWarnings = FALSE)
with_dir("doltdb", exec_wait("~/go/bin/dolt", "init"))
```

    ## Successfully initialized dolt data repository.

    ## [1] 0

``` r
dolt_server_pid <- with_dir("doltdb", sys::exec_background(
  "~/go/bin/dolt", c("sql-server", "--port 3333", "--host 127.0.0.1",
                     "--user user", "--password pwd", "--l error")
))
Sys.sleep(1) # Wait for server to fully start up before connecting
```

``` r
dolt_conn <- dbConnect(RMariaDB::MariaDB(), host = "127.0.0.1", port = 3333,
                       username = "user", password = "pwd", dbname = "doltdb")
```

# A Grossly Unfair Performance Comparison

Comparing Dolt to DuckDB and using arrow to write Parquet files. In our
use case, we might use the latter options and use S3 versioning to track
those binary files.

## Writing to the database

Dolt

``` r
flights <- nycflights13::flights
dim(flights)
```

    ## [1] 336776     19

``` r
print(object.size(flights), units = "Mb")
```

    ## 38.8 Mb

``` r
# Using dbx::dbInsert here rather than dbWriteTable because it is much faster
# to generate large INSERT statements than iterating through a parameterized
# statement as dbBind does.
dbCreateTable(dolt_conn, "flights", flights)
dolt_write_time <- system.time(
  dbxInsert(dolt_conn, "flights", flights)
)
dolt_write_time
```

    ##    user  system elapsed 
    ##   6.830   0.163  62.205

DuckDB

``` r
duck_conn <- dbConnect(duckdb::duckdb(), dbdir = "duckdb")
duck_write_time <- system.time(
  dbWriteTable(duck_conn, "flights", flights, overwrite = TRUE)
)
duck_write_time
```

    ##    user  system elapsed 
    ##   0.293   0.150   0.454

Parquet

``` r
pq_write_time <- system.time(
  arrow::write_parquet(flights, "flights.parquet")
)
pq_write_time
```

    ##    user  system elapsed 
    ##   0.251   0.010   0.244

This isn’t really a “fair” comparison, given the different goals of the
projects, but the write time for Dolt is 100 time that of DuckDB, and
300 of writing to Parquet via arrow.

## Read comparisons and dbplyr testing

``` r
dolt_read_time <- system.time(
  fl_dolt <- tbl(dolt_conn, "flights") |> 
    collect()
)
dolt_read_time
```

    ##    user  system elapsed 
    ##   0.999   2.151   5.336

``` r
duck_read_time <- system.time(
  fl_duck <- tbl(duck_conn, "flights") |> 
    collect()
)
duck_read_time
```

    ##    user  system elapsed 
    ##   0.073   0.028   0.101

``` r
pq_read_time <- system.time(
  fl_parquet  <- read_parquet("flights.parquet")
)
pq_read_time
```

    ##    user  system elapsed 
    ##   0.178   0.049   0.112

For reading the whole table, Dolt read time is 50 times that of DuckDB,
and 600 times that of reading parquet.

Now let’s compare querying and reading a tiny value

``` r
query_benchmark <- bench::mark(
  dolt_query = tbl(dolt_conn, "flights") |> 
    filter(carrier == "MQ", dest == "DTW", month == 5, arr_delay < -10, day == 30 ) |> 
    select(flight, tailnum) |> 
    collect(),
  
  duck_query = tbl(duck_conn, "flights") |> 
    filter(carrier == "MQ", dest == "DTW", month == 5, arr_delay < -10, day == 30 ) |> 
    select(flight, tailnum) |> 
    collect(),
  
  pq_query = arrow::open_dataset("flights.parquet") |> 
    filter(carrier == "MQ", dest == "DTW", month == 5, arr_delay < -10, day == 30 ) |> 
    select(flight, tailnum) |> 
    collect(),
  min_iterations = 10,
  check = FALSE # data are returned in different order, so don't check they are identical
)
query_benchmark
```

    ## # A tibble: 3 × 6
    ##   expression      min   median `itr/sec` mem_alloc `gc/sec`
    ##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    ## 1 dolt_query    1.79s    1.83s     0.547    2.35MB    0.137
    ## 2 duck_query  37.39ms  37.89ms    26.0    144.71KB   11.1  
    ## 3 pq_query    68.88ms  74.05ms    13.1      9.68MB    3.29

<!-- Wrap up -->
