
<!-- README.md is generated from README.Rmd. Please edit that file -->

# DBIlog

<!-- badges: start -->

[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)
[![CRAN
status](https://www.r-pkg.org/badges/version/DBIlog)](https://cran.r-project.org/package=DBIlog)
<!-- badges: end -->

The goal of DBIlog is to implement logging for arbitrary DBI backends,
similarly to Perl’s [DBI::Log](https://metacpan.org/pod/DBI::Log). This
is useful for troubleshooting and auditing codes that access a database.
The initial use case for this package is to help debugging DBItest
tests.

## Installation

You can install the released version of DBIlog from
[CRAN](https://CRAN.R-project.org) with:

``` r
install.packages("DBIlog")
```

Install the development version from GitHub using

``` r
# install.packages("devtools")
devtools::install_github("r-dbi/DBIlog")
```

## Example

The `LoggingDBI` driver wraps arbitrary drivers:

``` r
library(DBIlog)
drv <- LoggingDBI(RSQLite::SQLite())
#> drv1 <- RSQLite::SQLite()
```

All calls to DBI methods are logged, by default to the console. Logging
can be redirected to a file, optionally all outputs may be logged as
well.

``` r
conn <- dbConnect(drv, file = ":memory:")
#> conn1 <- dbConnect(drv1, file = ":memory:")
dbWriteTable(conn, "iris", iris[1:3, ])
#> dbWriteTable(conn1, name = "iris", value = structure(list(Sepal.Length = c(5.1, 4.9, 
#> 4.7), Sepal.Width = c(3.5, 3, 3.2), Petal.Length = c(1.4, 1.4, 1.3), Petal.Width = c(0.2, 
#> 0.2, 0.2), Species = structure(c(1L, 1L, 1L), .Label = c("setosa", "versicolor", 
#> "virginica"), class = "factor")), row.names = c(NA, 3L), class = "data.frame"), overwrite = FALSE, 
#>     append = FALSE)
data <- dbGetQuery(conn, "SELECT * FROM iris")
#> dbGetQuery(conn1, "SELECT * FROM iris")
#> ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
#> ## 1          5.1         3.5          1.4         0.2  setosa
#> ## 2          4.9         3.0          1.4         0.2  setosa
#> ## 3          4.7         3.2          1.3         0.2  setosa
dbDisconnect(conn)
#> dbDisconnect(conn1)

data
#>   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
#> 1          5.1         3.5          1.4         0.2  setosa
#> 2          4.9         3.0          1.4         0.2  setosa
#> 3          4.7         3.2          1.3         0.2  setosa
```

The log is runnable R code\! Run it in a fresh session to repeat the
operations, step by step or in an otherwise controlled fashion. For
example, use a collecting logger to output all calls and results after
the fact.

``` r
log_obj <- make_collect_log_obj()
collecting_logger <- logger(log_obj)

drv <- LoggingDBI(RSQLite::SQLite(), logger = collecting_logger)
conn <- dbConnect(drv, file = ":memory:")
dbWriteTable(conn, "iris", iris[1:3, ])
data <- dbGetQuery(conn, "SELECT * FROM iris")
dbDisconnect(conn)

log_obj$retrieve()
#> drv1 <- RSQLite::SQLite()
#> conn1 <- dbConnect(drv1, file = ":memory:")
#> dbWriteTable(conn1, name = "iris", value = structure(list(Sepal.Length = c(5.1, 4.9, 
#> 4.7), Sepal.Width = c(3.5, 3, 3.2), Petal.Length = c(1.4, 1.4, 1.3), Petal.Width = c(0.2, 
#> 0.2, 0.2), Species = structure(c(1L, 1L, 1L), .Label = c("setosa", "versicolor", 
#> "virginica"), class = "factor")), row.names = c(NA, 3L), class = "data.frame"), overwrite = FALSE, 
#>     append = FALSE)
#> dbGetQuery(conn1, "SELECT * FROM iris")
#> ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
#> ## 1          5.1         3.5          1.4         0.2  setosa
#> ## 2          4.9         3.0          1.4         0.2  setosa
#> ## 3          4.7         3.2          1.3         0.2  setosa
#> dbDisconnect(conn1)

ev <- evaluate::evaluate(log_obj$retrieve())
cat(unlist(ev, use.names = FALSE), sep = "\n")
#> drv1 <- RSQLite::SQLite()
#> 
#> conn1 <- dbConnect(drv1, file = ":memory:")
#> 
#> dbWriteTable(conn1, name = "iris", value = structure(list(Sepal.Length = c(5.1, 4.9, 
#> 4.7), Sepal.Width = c(3.5, 3, 3.2), Petal.Length = c(1.4, 1.4, 1.3), Petal.Width = c(0.2, 
#> 0.2, 0.2), Species = structure(c(1L, 1L, 1L), .Label = c("setosa", "versicolor", 
#> "virginica"), class = "factor")), row.names = c(NA, 3L), class = "data.frame"), overwrite = FALSE, 
#>     append = FALSE)
#> 
#> dbGetQuery(conn1, "SELECT * FROM iris")
#> 
#>   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
#> 1          5.1         3.5          1.4         0.2  setosa
#> 2          4.9         3.0          1.4         0.2  setosa
#> 3          4.7         3.2          1.3         0.2  setosa
#> 
#> ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
#> 
#> ## 1          5.1         3.5          1.4         0.2  setosa
#> 
#> ## 2          4.9         3.0          1.4         0.2  setosa
#> 
#> ## 3          4.7         3.2          1.3         0.2  setosa
#> 
#> dbDisconnect(conn1)
```

DBIlog is smart about DBI objects created or returned, and will assign a
new variable name to each new object. Cleared results or closed
connections are not removed automatically. Call `dbBegin()` and
`dbCommit()` or `dbRollback()` (or `dbWithTransaction()`) on the driver
object to define a scope for the autogenerated variable names.

``` r
dbBegin(drv)
#> Error in (function (classes, fdef, mtable) : unable to find an inherited method for function 'dbBegin' for signature '"LoggingDBIDriver"'

conn <- dbConnect(drv, file = ":memory:")
dbDisconnect(conn)

dbCommit(drv)
#> Error in (function (classes, fdef, mtable) : unable to find an inherited method for function 'dbCommit' for signature '"LoggingDBIDriver"'

dbBegin(drv)
#> Error in (function (classes, fdef, mtable) : unable to find an inherited method for function 'dbBegin' for signature '"LoggingDBIDriver"'

conn <- dbConnect(drv, file = ":memory:")
dbDisconnect(conn)

dbCommit(drv)
#> Error in (function (classes, fdef, mtable) : unable to find an inherited method for function 'dbCommit' for signature '"LoggingDBIDriver"'
```
