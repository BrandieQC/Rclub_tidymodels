---
title: "Chapter 7"
author: "Brandie Quarles"
date: "2023-12-08"
output: 
  html_document: 
    keep_md: yes
---



# Chapter 7 - A Model Workflow

<https://www.tmwr.org/workflows>

**Why a model workflow is important:**

1.  Encourages good methodology. Single point of entry to the estimation components of a data analysis.

2.  Enables better organization of projects.

**Model workflow = broader modeling process**

1.  Pre-processing steps

2.  Model itself

3.  Post-processing activities

## Load the Data


```r
library(tidyverse)
```

```
## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
## ✔ dplyr     1.1.3     ✔ readr     2.1.4
## ✔ forcats   1.0.0     ✔ stringr   1.5.0
## ✔ ggplot2   3.4.3     ✔ tibble    3.2.1
## ✔ lubridate 1.9.2     ✔ tidyr     1.3.0
## ✔ purrr     1.0.2     
## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
## ✖ dplyr::filter() masks stats::filter()
## ✖ dplyr::lag()    masks stats::lag()
## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors
```

```r
library(tidymodels)
```

```
## ── Attaching packages ────────────────────────────────────── tidymodels 1.1.1 ──
## ✔ broom        1.0.5     ✔ rsample      1.2.0
## ✔ dials        1.2.0     ✔ tune         1.1.2
## ✔ infer        1.0.5     ✔ workflows    1.1.3
## ✔ modeldata    1.2.0     ✔ workflowsets 1.0.1
## ✔ parsnip      1.1.1     ✔ yardstick    1.2.0
## ✔ recipes      1.0.8     
## ── Conflicts ───────────────────────────────────────── tidymodels_conflicts() ──
## ✖ scales::discard() masks purrr::discard()
## ✖ dplyr::filter()   masks stats::filter()
## ✖ recipes::fixed()  masks stringr::fixed()
## ✖ dplyr::lag()      masks stats::lag()
## ✖ yardstick::spec() masks readr::spec()
## ✖ recipes::step()   masks stats::step()
## • Learn how to get started at https://www.tidymodels.org/start/
```

```r
data(ames)
ames <- ames %>% mutate(Sale_Price = log10(Sale_Price))

set.seed(502)
ames_split <- initial_split(ames, prop = 0.80, strata = Sale_Price)
ames_train <- training(ames_split)
ames_test  <-  testing(ames_split)
```

## Workflow Package Basics


```r
library(tidymodels)  # Includes the workflows package
tidymodels_prefer()

lm_model <- 
  linear_reg() %>% 
  set_engine("lm")
```

A workflow requires a parsnip model object (made in above code chunk)


```r
lm_wflow <- 
  workflow() %>% 
  add_model(lm_model)

lm_wflow
```

```
## ══ Workflow ════════════════════════════════════════════════════════════════════
## Preprocessor: None
## Model: linear_reg()
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## Linear Regression Model Specification (regression)
## 
## Computational engine: lm
```

```r
#Note: "Preprocessor: None"
```

A standard R formula can be the pre-processor:


```r
lm_wflow <- 
  lm_wflow %>% 
  add_formula(Sale_Price ~ Longitude + Latitude)

lm_wflow
```

```
## ══ Workflow ════════════════════════════════════════════════════════════════════
## Preprocessor: Formula
## Model: linear_reg()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## Sale_Price ~ Longitude + Latitude
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## Linear Regression Model Specification (regression)
## 
## Computational engine: lm
```

Can use a fit() method to create the model:


```r
lm_fit <- fit(lm_wflow, ames_train)
lm_fit
```

```
## ══ Workflow [trained] ══════════════════════════════════════════════════════════
## Preprocessor: Formula
## Model: linear_reg()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## Sale_Price ~ Longitude + Latitude
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## 
## Call:
## stats::lm(formula = ..y ~ ., data = data)
## 
## Coefficients:
## (Intercept)    Longitude     Latitude  
##    -302.974       -2.075        2.710
```

Can predict() on the fitted workflow:


```r
predict(lm_fit, ames_test %>% slice(1:3))
```

```
## # A tibble: 3 × 1
##   .pred
##   <dbl>
## 1  5.22
## 2  5.21
## 3  5.28
```

Both the model and pre-processor can be removed or updated:


```r
lm_fit %>% update_formula(Sale_Price ~ Longitude)
```

```
## ══ Workflow ════════════════════════════════════════════════════════════════════
## Preprocessor: Formula
## Model: linear_reg()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## Sale_Price ~ Longitude
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## Linear Regression Model Specification (regression)
## 
## Computational engine: lm
```

## Adding Raw Variables to the workflow ( )

"There is another interface for passing data to the model, the `add_variables()` function, which uses a **dplyr**-like syntax for choosing variables. The function has two primary arguments: `outcomes` and `predictors`. These use a selection approach similar to the **tidyselect** backend of **tidyverse** packages to capture multiple selectors using `c()`."


```r
lm_wflow <- 
  lm_wflow %>% 
  remove_formula() %>% 
  add_variables(outcome = Sale_Price, predictors = c(Longitude, Latitude))
lm_wflow
```

```
## ══ Workflow ════════════════════════════════════════════════════════════════════
## Preprocessor: Variables
## Model: linear_reg()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## Outcomes: Sale_Price
## Predictors: c(Longitude, Latitude)
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## Linear Regression Model Specification (regression)
## 
## Computational engine: lm
```

```r
#Note: preprocessor: Variables (no longer formula)

#Can also use a more general selector:
#predictors = c(ends_with("tude"))

#Any outcome columns accidentally specified in the predictors argument will be removed. So, you can use:
#predictors = everything()
```

The above specification assembles the data, unaltered into a data frame and passes it to the underlying model function:


```r
fit(lm_wflow, ames_train)
```

```
## ══ Workflow [trained] ══════════════════════════════════════════════════════════
## Preprocessor: Variables
## Model: linear_reg()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## Outcomes: Sale_Price
## Predictors: c(Longitude, Latitude)
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## 
## Call:
## stats::lm(formula = ..y ~ ., data = data)
## 
## Coefficients:
## (Intercept)    Longitude     Latitude  
##    -302.974       -2.075        2.710
```

"If you would like the underlying modeling method to do what it would normally do with the data, `add_variables()` can be a helpful interface. As we will see in Section [7.4.1](https://www.tmwr.org/workflows#special-model-formulas), it also facilitates more complex modeling specifications. However, as we mention in the next section, models such as `glmnet` and `xgboost` expect the user to make indicator variables from factor predictors. In these cases, a recipe or formula interface will typically be a better choice."

-   Example: Would need to use add_formula ( ) with mixed effects models when you want to specify something like (week \| subject) to say that week is a random effect that has different slopes for each value of subject.

## How does a workflow ( ) use the formula?

Since the pre-processing is model dependent, workflows attempts to emulate what the underlying model would do whenever possible. If not possible, the formula processing would not do anything to the columns used in the formula.

### Special Formulas and Inline Functions 

Problem: Standard R methods can't properly process mixed effect model formulas. Also, the formula specifies the data and the statistical attributes of the model.

Workflow solution (use add_variables and add_model):


```r
library(multilevelmod)
data(Orthodont)
```

```
## Warning in data(Orthodont): data set 'Orthodont' not found
```

```r
multilevel_spec <- linear_reg() %>% set_engine("lmer")

multilevel_workflow <- 
  workflow() %>% 
  # Pass the data along as-is: 
  add_variables(outcome = distance, predictors = c(Sex, age, Subject)) %>% 
  add_model(multilevel_spec, 
            # This formula is given to the model
            formula = distance ~ Sex + (age | Subject))

#multilevel_fit <- fit(multilevel_workflow, data = Orthodont)
#multilevel_fit
#> ══ Workflow [trained] ═══════════════════════════════════════════════════════════════
#> Preprocessor: Variables
#> Model: linear_reg()
#> 
#> ── Preprocessor ─────────────────────────────────────────────────────────────────────
#> Outcomes: distance
#> Predictors: c(Sex, age, Subject)
#> 
#> ── Model ────────────────────────────────────────────────────────────────────────────
#> Linear mixed model fit by REML ['lmerMod']
#> Formula: distance ~ Sex + (age | Subject)
#>    Data: data
#> REML criterion at convergence: 471.2
#> Random effects:
#>  Groups   Name        Std.Dev. Corr 
#>  Subject  (Intercept) 7.391         
#>           age         0.694    -0.97
#>  Residual             1.310         
#> Number of obs: 108, groups:  Subject, 27
#> Fixed Effects:
#> (Intercept)    SexFemale  
#>       24.52        -2.15
```

Similar method can be used for survival analysis


```r
library(censored)
```

```
## Loading required package: survival
```

```r
parametric_spec <- survival_reg()

parametric_workflow <- 
  workflow() %>% 
  add_variables(outcome = c(fustat, futime), predictors = c(age, rx)) %>% 
  add_model(parametric_spec, 
            formula = Surv(futime, fustat) ~ age + strata(rx)) #allows the use of strata
#"in survival analysis models, a formula term such as strata(site) would indicate that the column site is a stratification variable. This means it should not be treated as a regular predictor and does not have a corresponding location parameter estimate in the model."
parametric_fit <- fit(parametric_workflow, data = ovarian)
parametric_fit
```

```
## ══ Workflow [trained] ══════════════════════════════════════════════════════════
## Preprocessor: Variables
## Model: survival_reg()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## Outcomes: c(fustat, futime)
## Predictors: c(age, rx)
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## Call:
## survival::survreg(formula = Surv(futime, fustat) ~ age + strata(rx), 
##     data = data, model = TRUE)
## 
## Coefficients:
## (Intercept)         age 
##  12.8734120  -0.1033569 
## 
## Scale:
##      rx=1      rx=2 
## 0.7695509 0.4703602 
## 
## Loglik(model)= -89.4   Loglik(intercept only)= -97.1
## 	Chisq= 15.36 on 1 degrees of freedom, p= 8.88e-05 
## n= 26
```

## Creating Multiple Workflows at Once 

Sometimes, the data require multiple attempts to find an appropriate model.

Examples:

-   Predictive models, when you need to evaluate a variety of model types.

-   Sequential testing of models when you want to compare the full model to models with predictors removed sequentially.

Solution: the **workflowset** package creates combinations of workflow components.

-   Combines a list of pre-processors with a list of model specifications --\> set of workflows.

Example with Ames data (evaluate the different ways that house location is represented):


```r
location <- list(
  longitude = Sale_Price ~ Longitude,
  latitude = Sale_Price ~ Latitude,
  coords = Sale_Price ~ Longitude + Latitude,
  neighborhood = Sale_Price ~ Neighborhood
)
```

Use the workflow_set() function:


```r
library(workflowsets)
location_models <- workflow_set(preproc = location, models = list(lm = lm_model))
location_models #provides a wflow_id to each workflow (combo of preproc and models)
```

```
## # A workflow set/tibble: 4 × 4
##   wflow_id        info             option    result    
##   <chr>           <list>           <list>    <list>    
## 1 longitude_lm    <tibble [1 × 4]> <opts[0]> <list [0]>
## 2 latitude_lm     <tibble [1 × 4]> <opts[0]> <list [0]>
## 3 coords_lm       <tibble [1 × 4]> <opts[0]> <list [0]>
## 4 neighborhood_lm <tibble [1 × 4]> <opts[0]> <list [0]>
```

```r
location_models$info[[1]] #basic info about the first workflow
```

```
## # A tibble: 1 × 4
##   workflow   preproc model      comment
##   <list>     <chr>   <chr>      <chr>  
## 1 <workflow> formula linear_reg ""
```

```r
extract_workflow(location_models, id = "coords_lm") #full details about the workflow of interest 
```

```
## ══ Workflow ════════════════════════════════════════════════════════════════════
## Preprocessor: Formula
## Model: linear_reg()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## Sale_Price ~ Longitude + Latitude
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## Linear Regression Model Specification (regression)
## 
## Computational engine: lm
```

"Workflow sets are mostly designed to work with resampling, which is discussed in Chapter [10](https://www.tmwr.org/resampling#resampling). The columns `option` and `result` must be populated with specific types of objects that result from resampling. We will demonstrate this in more detail in Chapters [11](https://www.tmwr.org/compare#compare) and [15](https://www.tmwr.org/workflow-sets#workflow-sets)."

Create model fits for each formula and save them in a new column called `fit`. We\'ll use basic **dplyr** and **purrr** operations:


```r
location_models <-
   location_models %>%
   mutate(fit = map(info, ~ fit(.x$workflow[[1]], ames_train)))
location_models
```

```
## # A workflow set/tibble: 4 × 5
##   wflow_id        info             option    result     fit       
##   <chr>           <list>           <list>    <list>     <list>    
## 1 longitude_lm    <tibble [1 × 4]> <opts[0]> <list [0]> <workflow>
## 2 latitude_lm     <tibble [1 × 4]> <opts[0]> <list [0]> <workflow>
## 3 coords_lm       <tibble [1 × 4]> <opts[0]> <list [0]> <workflow>
## 4 neighborhood_lm <tibble [1 × 4]> <opts[0]> <list [0]> <workflow>
```

```r
location_models$fit[[1]]
```

```
## ══ Workflow [trained] ══════════════════════════════════════════════════════════
## Preprocessor: Formula
## Model: linear_reg()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## Sale_Price ~ Longitude
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## 
## Call:
## stats::lm(formula = ..y ~ ., data = data)
## 
## Coefficients:
## (Intercept)    Longitude  
##    -184.396       -2.025
```

## Evaluating the Test Set

"Let\'s say that we\'ve concluded our model development and have settled on a final model. There is a convenience function called `last_fit()` that will *fit* the model to the entire training set and *evaluate* it with the testing set."


```r
final_lm_res <- last_fit(lm_wflow, ames_split) #takes a data split as input and not a dataframe 
final_lm_res
```

```
## # Resampling results
## # Manual resampling 
## # A tibble: 1 × 6
##   splits             id               .metrics .notes   .predictions .workflow 
##   <list>             <chr>            <list>   <list>   <list>       <list>    
## 1 <split [2342/588]> train/test split <tibble> <tibble> <tibble>     <workflow>
```

"The `.workflow` column contains the fitted workflow and can be pulled out of the results using:"


```r
fitted_lm_wflow <- extract_workflow(final_lm_res)
fitted_lm_wflow
```

```
## ══ Workflow [trained] ══════════════════════════════════════════════════════════
## Preprocessor: Variables
## Model: linear_reg()
## 
## ── Preprocessor ────────────────────────────────────────────────────────────────
## Outcomes: Sale_Price
## Predictors: c(Longitude, Latitude)
## 
## ── Model ───────────────────────────────────────────────────────────────────────
## 
## Call:
## stats::lm(formula = ..y ~ ., data = data)
## 
## Coefficients:
## (Intercept)    Longitude     Latitude  
##    -302.974       -2.075        2.710
```

Get access to performance metrics and predictions:


```r
collect_metrics(final_lm_res)
```

```
## # A tibble: 2 × 4
##   .metric .estimator .estimate .config             
##   <chr>   <chr>          <dbl> <chr>               
## 1 rmse    standard       0.164 Preprocessor1_Model1
## 2 rsq     standard       0.189 Preprocessor1_Model1
```

```r
collect_predictions(final_lm_res) %>% slice(1:5)
```

```
## # A tibble: 5 × 5
##   id               .pred  .row Sale_Price .config             
##   <chr>            <dbl> <int>      <dbl> <chr>               
## 1 train/test split  5.22     2       5.02 Preprocessor1_Model1
## 2 train/test split  5.21     4       5.39 Preprocessor1_Model1
## 3 train/test split  5.28     5       5.28 Preprocessor1_Model1
## 4 train/test split  5.27     8       5.28 Preprocessor1_Model1
## 5 train/test split  5.28    10       5.28 Preprocessor1_Model1
```

"When using validation sets, `last_fit()` has an argument called `add_validation_set` to specify if we should train the final model solely on the training set (the default) or the combination of the training and validation sets."
