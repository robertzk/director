# Director [![Build Status](https://travis-ci.org/syberia/director.svg?branch=master)](https://travis-ci.org/syberia/director) [![Coverage Status](https://coveralls.io/repos/syberia/director/badge.png)](https://coveralls.io/r/syberia/director) [![Documentation](https://img.shields.io/badge/rocco--docs-%E2%9C%93-blue.svg)](http://syberia.github.io/director/)

Due to its interactive nature, R tends to produce project structures
that are a collection of loosely organized scripts. With the exception
of [package development](http://r-pkgs.had.co.nz/r.html), there is no
consensus in the R community as to best practices for project structure.
Packages are well suited for abstract functions and features. However,
they are poor for coordinating code that is very domain-specific.
For example, the specific munging for a fixed data set belongs in an R script
and not within a package. 

Director aims to partially solve the problem of project structure in R
by re-defining R scripts as "resources" and allowing the developer to focus
only on what is important: this makes code more modular and re-usable.

# Example

Suppose we wish to be able to read data in, perform some munging, and
save it back to its source. One way to do this is as follows.

```r
data <- read.csv('/some/file')
# Do some munging
write.csv(data, '/some/file')
```

This works, but there are some problems with it. We have tied
the data-set to a specific source: a CSV file. What if we had hundreds
of such scripts and wanted to migrate them to read and write from a database
or an [S3 location](http://aws.amazon.com/s3)? We would have to update
`read.csv` in every file. The same applies if we want to add `stringsAsFactors = FALSE`
to each `read.csv` call.

An alternative approach is to create a directory `adapters` that takes the
following convention: any file in that directory must be of the following form.

```r
read <- function(key) {
  # Implement this.
}

write <- function(value, key) {
  # Implement this.
}
```

For example, we could implement CSV reading and writing as:

```r
# ./adapters/csv.R
read <- function(key) {
  read.csv(file.path("/some/path", paste0(key, ".csv")))
}

write <- function(value, key) {
  write.csv(value, file.path("/some/path", paste0(key, ".csv")))
}
```

Our original file could now become

```r
# ./our_script.R
function(adapter) {
  data <- adapter$read("file1")
  # Munge the data.
  adapter$write(data, "file1")
}
```

But what is `adapter`? We now use director to create a special `adapter`
object that will know how to read and write in the same format given a key. 

```r
# ./run.R

# Initialize a "director" object. It is responsible for determining
# what to do with files in the adapters directory (and eventually more).
library(director)
d <- director(".")

# We'll explain this later.
d$register_parser("/adapters", function(input) {
  structure(list(read = input$read, write = input$write), class = 'adapter')
})

adapter  <- d$resource("adapters/file")$value() # we will explain $value() later
resource <- d$resource("our_script.R")$value()
resource(adapter) # This will run our script and save the data.
```

Our scripts no longer depend on CSV reading. If we moved all our CSV
files to RDS files, we would only have to modify what adapter
gets passed to each script.

Now imagine we have hundreds of such scripts. We can put them in
`scripts` and use director to find all scripts.

```r
scripts <- d$find(base = "scripts") # Find all .R files in the scripts directory
# [1] "scripts/script1" "scripts/script2" ...

lapply(scripts, function(script) {
  script <- d$resource(script)$value()
  script(adapter)
})
```

The above executes all scripts in the `scripts` directory using an adapter of 
our choice. 


# Installation

Director is in active development and not yet available on CRAN (as of May 18, 2017).
To install the latest development build directly from Github,
run the following in the R console.

```r
if (!require(devtools)) install.packages('devtools')
devtools::install_github('syberia/director'); library(director)
```

# Creating a director

```r
d <- director('~/your/R/projects/directory')
```

You can now look for all files in your project using `d$find()`.

### License

This project is licensed under the MIT License:

Copyright (c) 2014-2017 Syberia, Avant, Robert Krzyzanowski

Permission is hereby granted, free of charge, to any person obtaining
a copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including
without limitation the rights to use, copy, modify, merge, publish,
distribute, sublicense, and/or sell copies of the Software, and to
permit persons to whom the Software is furnished to do so, subject to
the following conditions:

The above copyright notice and this permission notice shall be included
in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

### Authors

This package was created by Robert Krzyzanowski, rob@syberia.io in late 2014.


