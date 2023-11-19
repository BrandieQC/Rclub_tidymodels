---
title: "Chapter 5"
author: "Brandie Quarles"
date: "2023-11-18"
output: 
  html_document: 
    keep_md: yes
---



## Load the Data


```r
library(tidymodels)
```

```
## ── Attaching packages ────────────────────────────────────── tidymodels 1.1.1 ──
```

```
## ✔ broom        1.0.5     ✔ recipes      1.0.8
## ✔ dials        1.2.0     ✔ rsample      1.2.0
## ✔ dplyr        1.1.3     ✔ tibble       3.2.1
## ✔ ggplot2      3.4.3     ✔ tidyr        1.3.0
## ✔ infer        1.0.5     ✔ tune         1.1.2
## ✔ modeldata    1.2.0     ✔ workflows    1.1.3
## ✔ parsnip      1.1.1     ✔ workflowsets 1.0.1
## ✔ purrr        1.0.2     ✔ yardstick    1.2.0
```

```
## ── Conflicts ───────────────────────────────────────── tidymodels_conflicts() ──
## ✖ purrr::discard() masks scales::discard()
## ✖ dplyr::filter()  masks stats::filter()
## ✖ dplyr::lag()     masks stats::lag()
## ✖ recipes::step()  masks stats::step()
## • Use tidymodels_prefer() to resolve common conflicts.
```

```r
data(ames)
ames <- ames %>% mutate(Sale_Price = log10(Sale_Price))
```

# Chapter 5 - Spending the Data

<https://www.tmwr.org/splitting>

Steps to creating a useful model:

1.  parameter estimation

2.  model selection and tuning

3.  performance assessment

There's only a finite pool of data to split across these tasks. So, how do you do it?

-   If lots of data: allocate specific subsets of data for different tasks (ex: determine which predictors are abundant before parameter estimation)

-   If data is limited: some overlap in how and when data is allocated --\> need a methodology for data spending

## Common Methods for Data Splitting

### Split the data into a training set and test set

-   Training set - used to develop and optimize the model (usually the majority of the data)

-   Test set - held in reserve until one or two models are chosen. Used to determine the efficacy of the model. ONLY USE THIS ONCE, or it becomes part of the modeling process. Also, make sure that it resembles any new data that would be given to the model.

#### How do you split the data?

**Random sampling -** use the rsample package


```r
library(tidymodels)
tidymodels_prefer()

# Set the random number stream using `set.seed()` so that the results can be reproduced later. 
set.seed(501)

# Save the split information for an 80/20 split of the data
ames_split <- initial_split(ames, prop = 0.80)
ames_split #only contains the partitioning info
```

```
## <Training/Testing/Total>
## <2344/586/2930>
```

Get the actual data sets


```r
ames_train <- training(ames_split)
ames_test  <-  testing(ames_split)

dim(ames_train) #matches the length of the training set from the previous code chunk
```

```
## [1] 2344   74
```

**Stratified sampling** - the training/test split is conducted separately for each class and then those subsamples are combined

-   Can be useful when there is a class imbalance (one class occurs much less frequently than another). In that case, random sampling may allocate the rare class disproportionately to either the training or test set.

-   "For regression problems, the outcome data can be artificially binned into quartiles and then stratified sampling can be conducted four separate times. This is an effective method for keeping the distributions of the outcome similar between the training and test set."

Use strata within rsample to split the Ames data within each of the 4 quartiles of the sales price distribution.


```r
set.seed(502)
ames_split <- initial_split(ames, prop = 0.80, strata = Sale_Price)
ames_train <- training(ames_split)
ames_test  <-  testing(ames_split)

dim(ames_train)
```

```
## [1] 2342   74
```

Note: you can only use one column for stratification.

**Time series data** - common to use the most recent data as the test set.

-   "The **rsample** package contains a function called `initial_time_split()` that is very similar to `initial_split()`. Instead of using random sampling, the `prop` argument denotes what proportion of the first part of the data should be used as the training set; the function assumes that the data have been pre-sorted in an appropriate order."

**You should only avoid a test set when the data are really small.**

### Validation Sets

Used to get a rough estimate of how well the model performs prior to the test set.


```r
set.seed(52)
# To put 60% into training, 20% in validation, and 20% in testing:
ames_val_split <- initial_validation_split(ames, prop = c(0.6, 0.2))
ames_val_split
```

```
## <Training/Validation/Testing/Total>
## <1758/586/586/2930>
```

```r
ames_train <- training(ames_val_split)
ames_test <- testing(ames_val_split)
ames_val <- validation(ames_val_split)
```

This will be revisited in Chapter 10.

### Multilevel data

"Data splitting should occur at the independent experimental unit level of the data."

This is especially important when there are multiple rows per experimental unit. Ex: longitudinal data, repeated measures, different measures on the same plant, etc...
