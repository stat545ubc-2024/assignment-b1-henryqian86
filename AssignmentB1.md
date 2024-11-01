Stat545B_B1
================
Xihan Qian
2024-11-01

## Exercise 1

``` r
#' Summarize Numeric Data by Group
#'
#' Groups a data frame by specified columns and applies user-defined summary functions to numeric columns.
#'
#' @param data A data frame containing the data to summarize. Named `data` to
#' emphasize the primary input and maintain clarity across different functions.
#' @param group_cols A character vector of column names to group by. Named 
#' `group_cols` to clearly indicate that these columns define the groups for summarization.
#' @param summary_funs A named list of summary functions to apply to each numeric 
#' column (e.g., `list(mean = mean, median = median)`). Named `summary_funs` to emphasize that this parameter expects
#' functions that perform summaries.
#' @param na.rm A logical value indicating whether to remove NA values before applying the summary functions. Defaults to 
#' TRUE. Named `na.rm` to match common usage in R for specifying NA handling.
#'
#' @return A tibble with one row per group and one column per variable-summary combination.
#'

library(dplyr)
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
summarize_by_group <- function(data, group_cols, summary_funs = list(mean = mean, sd = sd), na.rm = TRUE) {
  # Check if input is a data frame
  if (!is.data.frame(data)) {
    stop("Input 'data' must be a data frame.")
  }
  
  # Ensure group columns exist in the data
  missing_cols <- setdiff(group_cols, names(data))
  if (length(missing_cols) > 0) {
    stop("Group columns not found in data: ", paste(missing_cols, collapse = ", "))
  }
  
  # Check if summary_funs is a named list of functions
  if (!is.list(summary_funs) || is.null(names(summary_funs)) || any(names(summary_funs) == "")) {
    stop("summary_funs must be a named list of functions")
  }
  
  # Exclude logical columns from the summary
  numeric_cols <- select(data, where(is.numeric))
  
  # Define a helper to apply na.rm in each summary function
  summary_funs_with_na_rm <- lapply(summary_funs, function(f) {
    function(x) f(x, na.rm = na.rm)
  })
  
  # Group the data and summarize with user-defined functions that include na.rm
  summarized_data <- data %>%
    group_by(across(all_of(group_cols))) %>%
    summarize(
      across(
        all_of(names(numeric_cols)),
        summary_funs_with_na_rm,
        .names = "{.col}_{.fn}"
      ),
      .groups = "drop"
    )
  
  return(summarized_data)
}
```

## Exercise 3

Below are some examples demonstrating the function, with some deliberate
error messages being displayed.

### Example 1

``` r
# Sample data
data <- tibble(
  group = c("A", "A", "A", "B", "B", "B", "C", "C", "C"),
  value1 = c(10, 20, 15, 30, NA, 25, 50, 60, NA),
  value2 = c(5, 3, 7, 8, 2, 4, 4, 7, 6)
)

# Define custom summary functions
summary_functions <- list(mean = mean, median = median)

# Use summarize_by_group function
summarize_by_group(data, group_cols = "group", summary_funs = summary_functions, na.rm = TRUE)
```

    ## # A tibble: 3 × 5
    ##   group value1_mean value1_median value2_mean value2_median
    ##   <chr>       <dbl>         <dbl>       <dbl>         <dbl>
    ## 1 A            15            15          5                5
    ## 2 B            27.5          27.5        4.67             4
    ## 3 C            55            55          5.67             6

Here, I use a simple data frame with one grouping column, group, and two
numeric columns, value1 and value2. The data frame has three groups (A,
B, and C) and includes some missing (NA) values in value1. Then I can
calculate the mean and median for value1 and value2 in each group (A, B,
and C). The na.rm = TRUE argument ensures that NA values are ignored in
the calculations.

### Example 2

``` r
summary_functions <- list(sd = sd, min = min, max = max)

# with a new set of summary functions(sd, min and max)
summarize_by_group(data, group_cols = "group", summary_funs = summary_functions, na.rm = TRUE)
```

    ## # A tibble: 3 × 7
    ##   group value1_sd value1_min value1_max value2_sd value2_min value2_max
    ##   <chr>     <dbl>      <dbl>      <dbl>     <dbl>      <dbl>      <dbl>
    ## 1 A          5            10         20      2             3          7
    ## 2 B          3.54         25         30      3.06          2          8
    ## 3 C          7.07         50         60      1.53          4          7

This example demonstrates using different summary functions (standard
deviation, minimum, and maximum) on the same data frame.

### Example 3

``` r
# Attempt to group by a non-existent column
summarize_by_group(data, group_cols = "category", summary_funs = summary_functions, na.rm = TRUE)
```

    ## Error in summarize_by_group(data, group_cols = "category", summary_funs = summary_functions, : Group columns not found in data: category

Since category is not a column in data, this will trigger an error
message: “Group columns not found in data.” This helps prevent
accidental errors when specifying grouping columns.

### Example 4

``` r
# Attempt to use a vector instead of a data frame
data_vector <- c(1, 2, 3, 4, 5)

# This should cause an error
summarize_by_group(data_vector, group_cols = "group", summary_funs = list(mean = mean), na.rm = TRUE)
```

    ## Error in summarize_by_group(data_vector, group_cols = "group", summary_funs = list(mean = mean), : Input 'data' must be a data frame.

The function expects data to be a data frame. Passing a vector instead
will produce an error: “Input ‘data’ must be a data frame.” This check
enforces correct input types.

## Exercise 4

``` r
library(testthat)
```

    ## 
    ## Attaching package: 'testthat'

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     matches

``` r
library(dplyr)

# Define the test suite for summarize_by_group
test_that("summarize_by_group function works as expected", {
  
  # Test 1: Basic functionality with no NAs (3 values per group)
  data_no_na <- tibble(
    group = c("A", "A", "A", "B", "B", "B", "C", "C", "C"),
    value1 = c(10, 20, 15, 30, 40, 25, 50, 60, 55),
    value2 = c(5, 3, 7, 8, 2, 4, 4, 7, 6)
  )
  
  summary_functions <- list(mean = mean, sd = sd)
  result <- summarize_by_group(data_no_na, group_cols = "group", summary_funs = summary_functions, na.rm = TRUE)
  
  # Expect correct structure and no NAs in output
  expect_s3_class(result, "tbl_df") # Check that result is a tibble
  expect_true(all(!is.na(result))) # Check that there are no NA values in the output
  
  # Test 2: Functionality with NAs, ensuring na.rm = TRUE works as expected
  data_with_na <- tibble(
    group = c("A", "A", "A", "B", "B", "B", "C", "C", "C"),
    value1 = c(10, 20, NA, 30, NA, 25, 50, 60, NA),
    value2 = c(5, 3, 7, 8, 2, NA, 4, 7, 6)
  )
  
  result_with_na <- summarize_by_group(data_with_na, group_cols = "group", summary_funs = summary_functions, na.rm = TRUE)
  
  # Expect correct calculations even with NA values
  expect_true(!any(is.na(result_with_na$value1_mean)))
  expect_true(!any(is.na(result_with_na$value2_sd))) 
  
  # Test 3: Handling of missing group column (should produce an error)
  expect_error(
    summarize_by_group(data_no_na, group_cols = "nonexistent_column", summary_funs = summary_functions, na.rm = TRUE),
    "Group columns not found in data"
  )
  
  # Test 4: Edge case with an empty data frame with correct column names
  empty_data <- tibble(group = character(0), value1 = numeric(0), value2 = numeric(0))
  result_empty <- summarize_by_group(empty_data, group_cols = "group", summary_funs = summary_functions, na.rm = TRUE)
  
  # Expect the output to be an empty tibble with appropriate column names
  expect_s3_class(result_empty, "tbl_df")
  expect_equal(nrow(result_empty), 0)
  expect_setequal(names(result_empty), c("group", "value1_mean", "value1_sd", "value2_mean", "value2_sd"))
  
  # Test 5: Invalid type for summary_funs (not a list), should produce an error
  expect_error(
    summarize_by_group(data_no_na, group_cols = "group", summary_funs = c("mean", "sd"), na.rm = TRUE),
    "summary_funs must be a named list of functions"
  )
})
```

    ## Test passed 🎉

The tests provided for the summarize_by_group() function are to cover a
range of non-redundant inputs.

Test 1 checks the basic functionality of the function using data with no
missing values. This test ensures that, under standard conditions, the
function performs correctly and produces expected results.

Test 2 examines how the function handles NA values, using the na.rm =
TRUE argument.

Test 3 addresses input validation by intentionally providing a
non-existent group column.

Test 4 checks that the function does not fail or produce errors and
instead returns a tibble with zero rows but correctly named columns.

Test 5 assesses the function’s ability to validate the format of
summary_funs.
