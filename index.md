---
title: "Practical Machine Learning Course Project"
author: "Dan C"
date: "8/22/2020"
output: 
  html_document: 
    keep_md: yes
---

## Summary
This project builds a machine learning model to identify weightlifting movements through accelerometer measurements. The data comes from a study of six test subjects who were instructed to lift a barbell in various correct or incorrect techniques. Each subject had several accelerometers fixed to their bodies as they performed 10 reps with each lifting technique. The goal of this project was to train a model that could correctly identify which lifting technique was used based on the accelerometer measurements and then apply the model to a test set of 20 observations.

After evaluating models based on linear discriminant analysis, boosted trees, and random forest, it was found that the random forest method produced the most accurate model.

**Acknowledgement**  
This report made use of course forum discussions, especially course mentor Len Greski's tutorial on parallel processing. 

## Loading and Cleaning Data
The data in the training set consists of accelerometer measurements taken over the course of a weight-lifting movement. These measurements are time series split up into "windows" of several seconds. In addition to the observations taken at each instant of time, the dataset includes aggregated statistics at the start of each new window (e.g., max/min and average values). 

The testing dataset, however, does not include any rows containing the aggregated window data (the columns are blank or NA), which suggests that we are meant to predict the movements based on the instantaneous measurements alone. If that is the case, we should omit the aggregated data columns from our training data, as they are blank or NA for most rows.    

```r
training = read.csv("pml-training.csv")
testing = read.csv("pml-testing.csv")

training1 <- training
training1 <- training1[,-grep("kurtosis",colnames(training1))]
training1 <- training1[,-grep("skewness",colnames(training1))]
training1 <- training1[,-grep("max_",colnames(training1))]
training1 <- training1[,-grep("min_",colnames(training1))]
training1 <- training1[,-grep("amplitude_",colnames(training1))]
training1 <- training1[,-grep("var_",colnames(training1))]
training1 <- training1[,-grep("stddev_",colnames(training1))]
training1 <- training1[,-grep("avg_",colnames(training1))]
```

Because we are attempting to predict based on instantaneous measurements, we can omit columns pertaining to time. We also omit row indexes from the predictors. Finally, we omit the subject names from the dataset so that this model will be applicable to users beyond the original six test subjects. These omissions are achieved by removing the first seven columns from the original training set.

```r
training1 <- training1[,-c(1:7)]
```

The final training dataset contains 52 predictors consisting of instantaneous accelerometer measurements plus the *classe* outcome column.

We conduct the same transformation to the testing dataset.


```r
testing1 <- testing
testing1 <- testing1[,-grep("kurtosis",colnames(testing1))]
testing1 <- testing1[,-grep("skewness",colnames(testing1))]
testing1 <- testing1[,-grep("max_",colnames(testing1))]
testing1 <- testing1[,-grep("min_",colnames(testing1))]
testing1 <- testing1[,-grep("amplitude_",colnames(testing1))]
testing1 <- testing1[,-grep("var_",colnames(testing1))]
testing1 <- testing1[,-grep("stddev_",colnames(testing1))]
testing1 <- testing1[,-grep("avg_",colnames(testing1))]
testing1 <- testing1[,-c(1:7)]
```

## Modeling

Three machine learning models were evaluated for this classification problem. We begin with a relatively simply linear discriminant analysis (LDA) model before attempting more computationally intensive tree models.

### Model 1: Linear Discriminant Analysis

The first machine learning model attempted was a linear discriminant analysis (LDA) model. We train all models with parallel processing enabled to reduce computing time. The following code initializes the parallel processing cluster.


```r
library(parallel)
library(doParallel)

cluster <- makeCluster(detectCores() - 1)
registerDoParallel(cluster)
```

Next we use the *traincontrol()* function to define settings for the *train()* function that will be called in the next step. We will enable parallel processing for train() in this step. We also define the resampling method here, selecting *method=cv* and *number=5* to use 5-fold cross-validation.     

```r
fitControl <- trainControl(method = "cv", number = 5, allowParallel = TRUE)
```

With *traincontrol()* set, we can now train an LDA model with classe as the response and all other columns in *training1* as predictors. The final two lines of code deactivate the parallel processing cluster created earlier.

```r
set.seed(12345)
fit1 <- train(classe ~ ., method="lda",data=training1,trControl = fitControl)

stopCluster(cluster)
registerDoSEQ()
```
The resulting LDA model has a training accuracy of **0.7023**. For this assignment, the target out-of-sample accuracy is at least 0.80 (the percent of correct predictions on the test set required to pass part 2 of the project). This LDA model is therefore insufficient for our purposes.  

### Model 2: Boosted Trees
We next try tree-based classification methods to predict the weight-lifting movements. The first tree-based method we will attempt is a boosted tree model using the *gbm* method in caret. For this model, we will use the same *traincontrol()* settings as the LDA model (5-fold cross-validation, parallel processing enabled). When calling the *train()* function, we again predict classe as a function of all other columns in the *training1* dataset and set the method to *gbm*.   


```r
cluster <- makeCluster(detectCores() - 1)
registerDoParallel(cluster)

fitControl <- trainControl(method = "cv", number = 5, allowParallel = TRUE)
set.seed(12345)
fit2 <- train(classe ~ ., method="gbm",data=training1,trControl = fitControl,verbose=FALSE)

stopCluster(cluster)
registerDoSEQ()
```
The resulting model has a training accuracy of **0.9619**. While this is much better than the LDA model's accuracy, we will continue to search for a more accurate model.

### Model 3: Random Forest
The third classification method evaluated is the random forest model. The random forest model will use slightly different *traincontrol()* settings. Rather than using the cross-validation method for resampling, we specify the resampling method as out-of-bag (oob), which is included in caret specifically for random forest models. The *train()* command is otherwise called in a similar fashion to the previous models. 

```r
cluster <- makeCluster(detectCores() - 1)
registerDoParallel(cluster)

oobControl2 <- trainControl(method = "oob", allowParallel = TRUE)
set.seed(12345)
fit3 <- train(classe ~ ., method="rf",data=training1,trControl = oobControl2)

stopCluster(cluster)
registerDoSEQ()
```

The resulting random forest model has a training accuracy of **0.9959**. Because this is the most accurate of the three models attempted in this project, we will select this as the final model to predict the weight-lifting movements in the test set.

### Out of Sample / Out of Bag Error
For random forest models, the out of bag (OOB) error estimate is the equivalent to out of sample error calculated by cross-validated methods. The OOB error estimate is generated when using *train()* on a random forest model. To retrieve this estimate, use the following code:


```r
fit3$finalModel
```

The above code indicates that the OOB error estimate is **0.42%**.


## Prediction on Test Set
To apply the random forest model to the testing dataset, we use the *predict()* function as follows.

```r
predict(fit3,newdata=testing1)
```

Executing this code generates a vector of predicted *classe* values for each of the 20 rows in the testing dataset. When these predictions were entered into the quiz in Part 2 of this assignment, all 20 predictions were found to be correct. 

## Conclusion
The random forest model provides the most accurate predictions in the weight lifting dataset. The model built has an out of bag error rate of 0.42%. It correctly predicted 20 out of 20 movements in the test set. Although the observations in the dataset were structured as time series, the dataset contained enough information for the random forest model to predict the correct movements based on a single isolated observation.

