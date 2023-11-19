---
title: "Chapter 6"
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
## • Search for functions across packages at https://www.tidymodels.org/find/
```

```r
data(ames)
ames <- ames %>% mutate(Sale_Price = log10(Sale_Price))

set.seed(502)
ames_split <- initial_split(ames, prop = 0.80, strata = Sale_Price)
ames_train <- training(ames_split)
ames_test  <-  testing(ames_split)
```

# Chapter 6 - Fitting Models with parsnip

<https://www.tmwr.org/models>

## Create a Model 

### Linear Regression Models

-   *"Ordinary linear regression* uses the traditional method of least squares to solve for the model parameters."


```r
#use the stats package 
#model <- lm(formula, data, ...) #syntax 
```

-   *"Regularized linear regression* adds a penalty to the least squares method to encourage simplicity by removing predictors and/or shrinking their coefficients towards zero. This can be executed using Bayesian or non-Bayesian techniques."


```r
#use the rstanarm package for Bayesian approach 
#model <- stan_glm(formula, data, family = "gaussian", ...) #syntax 
#... = arguments for the prior distributions of the parameters and details about the numerical aspects of the model 
```


```r
#for non-Bayesian approach to regularized regression use glmnet model
#model <- glmnet(x = matrix, y = vector, family = "gaussian", ...) #syntax 

#In this case, the predictor data must already be formatted into a numeric matrix; there is only an x/y method and no formula method.
```

Problem with the above options is that their interfaces are heterogeneous. They require different formatting of the data and have different syntax for thier arguments.

### Specifying the details of a model (via tidymodels methodlogy):

1.  Specify the type of model based on its mathematical structure (Ex: linear regression, random forest, etc...)

2.  Specify the engine for fitting the model; i.e. what software package should be used? (Ex: Stan or glmnet). Parsnip provides consistent interfaces for using those engines for modeling.

3.  Declare the mode of the model, when required; i.e. the type of prediction outcome (Ex: regression, classification)

Make the above specifications before referencing the data.


```r
library(tidymodels)
tidymodels_prefer()

linear_reg() %>% set_engine("lm")
```

```
## Linear Regression Model Specification (regression)
## 
## Computational engine: lm
```

```r
linear_reg() %>% set_engine("glmnet") 
```

```
## Linear Regression Model Specification (regression)
## 
## Computational engine: glmnet
```

```r
linear_reg() %>% set_engine("stan")
```

```
## Linear Regression Model Specification (regression)
## 
## Computational engine: stan
```

### Perform model estimation 

Use the `fit()` function (to use a formula) or the `fit_xy()` function (when your data are already pre-processed).

#### With parsnip, you can ignore the interface of the underlying model. 

-   You can use a formula even if the modeling package only has an x/y interface.

The `translate()` function can provide details on how **parsnip** converts the user\'s code to the package\'s syntax


```r
linear_reg() %>% set_engine("lm") %>% translate()
```

```
## Linear Regression Model Specification (regression)
## 
## Computational engine: lm 
## 
## Model fit template:
## stats::lm(formula = missing_arg(), data = missing_arg(), weights = missing_arg())
```

```r
linear_reg(penalty = 1) %>% set_engine("glmnet") %>% translate()
```

```
## Linear Regression Model Specification (regression)
## 
## Main Arguments:
##   penalty = 1
## 
## Computational engine: glmnet 
## 
## Model fit template:
## glmnet::glmnet(x = missing_arg(), y = missing_arg(), weights = missing_arg(), 
##     family = "gaussian")
```

```r
linear_reg() %>% set_engine("stan") %>% translate()
```

```
## Linear Regression Model Specification (regression)
## 
## Computational engine: stan 
## 
## Model fit template:
## rstanarm::stan_glm(formula = missing_arg(), data = missing_arg(), 
##     weights = missing_arg(), family = stats::gaussian, refresh = 0)
```

```r
#"missing_arg()" = placeholder for the data that hasn't been provided yet 
```

**Example Code:** Predict the sale price of houses as a function of only long and lat


```r
lm_model <- 
  linear_reg() %>% 
  set_engine("lm")

lm_form_fit <- 
  lm_model %>% 
  # Recall that Sale_Price has been pre-logged
  fit(Sale_Price ~ Longitude + Latitude, data = ames_train)

lm_xy_fit <- 
  lm_model %>% 
  fit_xy(
    x = ames_train %>% select(Longitude, Latitude),
    y = ames_train %>% pull(Sale_Price)
  )

lm_form_fit
```

```
## parsnip model object
## 
## 
## Call:
## stats::lm(formula = Sale_Price ~ Longitude + Latitude, data = data)
## 
## Coefficients:
## (Intercept)    Longitude     Latitude  
##    -302.974       -2.075        2.710
```

```r
lm_xy_fit
```

```
## parsnip model object
## 
## 
## Call:
## stats::lm(formula = ..y ~ ., data = data)
## 
## Coefficients:
## (Intercept)    Longitude     Latitude  
##    -302.974       -2.075        2.710
```

```r
#no matter how you code it, you get the same results 
```

#### Parsnip provides consistency in the model arguments 

-   Uses common argument names w/in and b/t packages

-   Tries to use names that the people viewing the results would understand

To map the parsnip names to the original names use a combo of help for the model and the translate( ) function.


```r
rand_forest(trees = 1000, min_n = 5) %>% 
  set_engine("ranger") %>% 
  set_mode("regression") %>% 
  translate()
```

```
## Random Forest Model Specification (regression)
## 
## Main Arguments:
##   trees = 1000
##   min_n = 5
## 
## Computational engine: ranger 
## 
## Model fit template:
## ranger::ranger(x = missing_arg(), y = missing_arg(), weights = missing_arg(), 
##     num.trees = 1000, min.node.size = min_rows(~5, x), num.threads = 1, 
##     verbose = FALSE, seed = sample.int(10^5, 1))
```

There are 2 catgs of model arguments:

-   Main arguments - commonly used and available across engines

-   Engine arguments - spec. to a particular engine or used more rarely

Can specific engine-specific arguments in set_engine().


```r
rand_forest(trees = 1000, min_n = 5) %>% 
  set_engine("ranger", verbose = TRUE) %>% #print out more info about the fit 
  set_mode("regression") 
```

```
## Random Forest Model Specification (regression)
## 
## Main Arguments:
##   trees = 1000
##   min_n = 5
## 
## Engine-Specific Arguments:
##   verbose = TRUE
## 
## Computational engine: ranger
```

## Use the Model Results

The parsnip model object stores several things

-   The fitted model can be found in an element called fit - use the extract_fit_engine() function:


```r
lm_form_fit %>% extract_fit_engine()
```

```
## 
## Call:
## stats::lm(formula = Sale_Price ~ Longitude + Latitude, data = data)
## 
## Coefficients:
## (Intercept)    Longitude     Latitude  
##    -302.974       -2.075        2.710
```

You can then print or plot that object:


```r
lm_form_fit %>% extract_fit_engine() %>% vcov()
```

```
##             (Intercept)     Longitude      Latitude
## (Intercept)  207.311311  1.5746587743 -1.4239709610
## Longitude      1.574659  0.0165462548 -0.0005999802
## Latitude      -1.423971 -0.0005999802  0.0325397353
```

-   **Disclaimer: "**Never pass the `fit` element of a **parsnip** model to a model prediction function, i.e., use `predict(lm_form_fit)` but *do not* use `predict(lm_form_fit$fit)`. If the data were preprocessed in any way, incorrect predictions will be generated (sometimes, without errors). The underlying model\'s prediction function has no idea if any transformations have been made to the data prior to running the model.:

-   You can also save the summary() results


```r
model_res <- 
  lm_form_fit %>% 
  extract_fit_engine() %>% 
  summary()

# The model coefficient table is accessible via the `coef` method.
param_est <- coef(model_res)
class(param_est)
```

```
## [1] "matrix" "array"
```

```r
param_est
```

```
##                Estimate Std. Error   t value     Pr(>|t|)
## (Intercept) -302.973554 14.3983093 -21.04230 3.640103e-90
## Longitude     -2.074862  0.1286322 -16.13019 1.395257e-55
## Latitude       2.709654  0.1803877  15.02128 9.289500e-49
```

If you want to create a table or other visualization from the parameter values, you would need to convert the above result from a matrix to a data frame.

-   Use the broom package to convert model objects to tidy structure


```r
tidy(lm_form_fit)
```

```
## # A tibble: 3 × 5
##   term        estimate std.error statistic  p.value
##   <chr>          <dbl>     <dbl>     <dbl>    <dbl>
## 1 (Intercept)  -303.      14.4       -21.0 3.64e-90
## 2 Longitude      -2.07     0.129     -16.1 1.40e-55
## 3 Latitude        2.71     0.180      15.0 9.29e-49
```

```r
#removes info about the type of statistical test
#standardizes the row names 
```

## Make Predictions

Parsnip rules for predictions

1.  Results = tibble
2.  Col names = predictable
3.  Same \# rows in tibble as in the input data set (i.e. if any rows of the new data contain missing values, the output will be padded with missing results for those rows)


```r
ames_test_small <- ames_test %>% slice(1:5)
predict(lm_form_fit, new_data = ames_test_small)
```

```
## # A tibble: 5 × 1
##   .pred
##   <dbl>
## 1  5.22
## 2  5.21
## 3  5.28
## 4  5.27
## 5  5.28
```

```r
#row order same as OG data --> easier to mege with the OG data

ames_test_small %>% 
  select(Sale_Price) %>% 
  bind_cols(predict(lm_form_fit, ames_test_small)) %>% 
  # Add 95% prediction intervals to the results:
  bind_cols(predict(lm_form_fit, ames_test_small, type = "pred_int")) 
```

```
## # A tibble: 5 × 4
##   Sale_Price .pred .pred_lower .pred_upper
##        <dbl> <dbl>       <dbl>       <dbl>
## 1       5.02  5.22        4.91        5.54
## 2       5.39  5.21        4.90        5.53
## 3       5.28  5.28        4.97        5.60
## 4       5.28  5.27        4.96        5.59
## 5       5.28  5.28        4.97        5.60
```

Benefit of standardization = different models can be used with the same syntax


```r
#alternative model 
tree_model <- 
  decision_tree(min_n = 2) %>% 
  set_engine("rpart") %>% 
  set_mode("regression")

tree_fit <- 
  tree_model %>% 
  fit(Sale_Price ~ Longitude + Latitude, data = ames_train)

ames_test_small %>% 
  select(Sale_Price) %>% 
  bind_cols(predict(tree_fit, ames_test_small))
```

```
## # A tibble: 5 × 2
##   Sale_Price .pred
##        <dbl> <dbl>
## 1       5.02  5.15
## 2       5.39  5.15
## 3       5.28  5.32
## 4       5.28  5.32
## 5       5.28  5.32
```

A list of all of the models that can be used with **parsnip** (across different packages that are on CRAN) can be found at <https://www.tidymodels.org/find/>.

## Creating model specifications - made easy


```r
#parsnip_addin()
```

Opens a window w/ a list of possible models for each model mode.
