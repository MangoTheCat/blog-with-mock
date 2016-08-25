---
title: "Testing without the internet using mock functions"
author: "Karina Marks, Gábor Csárdi"
output: html_document
---

## Testing

At Mango we validate and test packages for our ValidR customers.
We create unit tests for different requirements
of functions. These tests must run on a customers machine, where internet
access could be restricted; hence, all tests 
must work - and pass - on any OS and without an internet connection. This
creates a problem when testing packages whose main functionality is to
connect to a server for tasks such as; web scraping or web API. The way we
do this is by using the method of
[mocking](https://en.wikipedia.org/wiki/Mock_object).


## Mocking

Mocking allows you to replace parts of your system under test with mock
objects and make assertions about how they have been used. The `testthat`
package supports mocking via the `with_mock()` function. `with_mock()` allows us to
temporarily replace some R functions. In our case we replace the functions
that connect to the internet with mock functions that only *pretend* to do
so.

### Simple Example - a system error

Suppose we use the `system()` function to call out to the operating
system and start some utility. This is a somewhat fragile operation, it
might fail if the utility is not available, so we make sure that we 
handle errors properly. This is somewhat tricky in the case of `system()` as 
it does not signal an R error if the sytem function fails, it just 
returns the exit status of the system shell, which is typically
non-zero on an error.

To test that errors are handled properly, we would need to create an
environment where the system function indeed fails. While this is not
impossible, it is much easier to just pretend that it fails using
`with_mock()`:


```r
## Package or script code
ext <- function() {
  ## This usually works, but probably not always
  status <- system("sleep 5")
  if (status != 0) { stop("sleeping failed") }
}
## Test code
library(testthat)
with_mock(
  `base::system` = function(...) { return(127) },
  expect_error(ext(), "sleeping failed")
)
```

We want to test that `ext()` behaves well if a system error happens,
and throws a proper R error. We use `with_mock()` to temporarily change
`system()`, and pretend that it has failed. `with_mock()` takes two kinds
of arguments: named and unnamed. The named arguments are the mock functions,
the names define the functions to mock. Unnamed arguments are expressions
to evaluate in the mocked environment.

## Using `with_mock` to avoid internet

### Example - `GET()`

Consider the function `GET()` from the `httr` package,
which will perform an HTTP GET request:


```r
library(httr)
response <- GET("http://httpbin.org/get")
```

```
## Tracing curl::curl_fetch_memory(url, handle = handle) on exit
```

Under the hood `GET()` uses the `curl_fetch_memory()` function from the
`curl` package, built on top of `libcurl`. The job of the `GET()` function is
to call `curl` with the right arguments, and then process its response
properly. So this is what we need to test. (Testing `curl` itself is
another mocking story.)

If we would like to test that `GET()` works correctly, without any internet
connection, the steps to do this are;

* Trace `curl_fetch_memory()` to see the input it receives and the
  output it generates.
* Call `GET()` for the scenario we want to test, and record the input
  and output of `curl_fetch_memory()`, so that we can use it to pretend an
  internet connection later in the real test case. Study the recorded
  input and output to see it can be reused, and update it as needed.
* Write the test case using `with_mock()` on `curl_fetch_memory()` to
  replace it with a mock function that checks if the input received from
  `GET()` is correct and provide the recorded output.

For brevity, let's assume that we only use the output of
`curl_fetch_memory()` now. The input can be handled similarly.
We trace it and record its return value:


```r
library(curl)
cfm_output <- NULL
trace(
  curl_fetch_memory,
  exit = function() { cfm_output <<- returnValue() }
)
response <- GET("http://httpbin.org/get")
```

The return value of `curl_fetch_memory()` is a list that contains the HTTP
status code, headers, some timing information, and of course the content
itself:


```r
names(cfm_output)
```

```
## [1] "url"         "status_code" "headers"     "modified"    "times"      
## [6] "content"
```

We save this into a file now, and put the file into the `testthat`
directory where the unit tests run.


```r
save(cfm_output, file = "cfm_output.rda")
```

Now that we have the response stored locally we can use it to unit test
`GET()` without connecting to the internet.


```r
test_that("GET works as it should", {
  response <- with_mock(
    `curl::curl_fetch_memory` = function(...) {
	    load("cfm_output.rda")
      cfm_output
    },
    GET("http://httpbin.org/get")
  )
  expect_silent(stop_for_status(response))
  expect_equal(headers(response)$`content-type`, "application/json")
  # ... more tests for the response
})
```

We have successfully tested `GET()` without having to connect to a server.

## Primitive Functions

The only problem with using this method is that you may sometimes wish to
mock a primitive function, but `with_mock` does not allow you to do this.

### Example - `BROWSE()`

This function is again from the `httr` package and relies on the function
`browseURL()` from the base package `utils` to connect to the internet, so we
will need to mock this function. This function only runs if the R session
is interactive, i.e. `isTrue(interactive())`, which probably holds when
*writing* the unit tests, but not when *running* them. 
It seems straightforward to mock `interactive()` to return `TRUE`, but it
is a primitive function, so `with_mock()` cannot deal with it:

```r
interactive
with_mock(
  `interactive` = function(...) TRUE,
  interactive()
)
```

```
## Error in FUN(X[[i]], ...): old_fun must be a function
```

```
## Error in FUN(X[[i]], ...): old_fun must be a function
```

To work around this we must mock the function manually:


```r
test_that("Interactive is always TRUE", {

  interactive_original <- base::interactive
  on.exit(
    {
      assign("interactive", interactive_original, envir = baseenv())
      lockBinding("interactive", baseenv())
    },
    add = TRUE
  )
  unlockBinding('interactive', baseenv())
  assign('interactive', function() TRUE, envir = baseenv())

  expect_true(interactive())
})
```

* We have saved the original version of `interactive()`, so that we can restore it.
* `unlockBinding()` has allowed us to replace the function in the `base`
  package with our version.
* `on.exit()` makes sure that after the test has run, this function is
  changed back, and  `lockBinding()` seals the `base` package again.

After this, mocking `browseURL()` is already easy, and we leave it
as an exercise to the reader.

Note that this method is not allowed for tests in CRAN packages,
because calling `unlockBinding()` is forbidden there. If you are
writing tests for your own R package, then a workaround is to define
your own non-primitive `is_interactive()` function and use (and mock) 
that in your package:


```r
is_interactive <- function() interactive()
with_mock(
  `is_interactive` = function() TRUE,
  is_interactive()
)
```

```
## [1] TRUE
```

Links

* [This blog post on GitHub](https://github.com/MangoTheCat/blog-with-mock)
* [The `testthat` R package.](https://github.com/hadley/testthat#readme)
* [The `httr` R package.](https://github.com/hadley/httr#readme)
* [The `curl` R package.](https://github.com/jeroenooms/curl#readme)
* [ValidR](http://www.mango-solutions.com/wp/products-services/products/validr/)
