---
title: "Pratical Machine Learning Writeup"
author: "KONAN  Amani Dieudonne"
date: "Thursday, March 12, 2015"
output: html_document
---

# Background

Using devices such as *Jawbone Up, Nike FuelBand*, and Fitbit it is now possible to collect a large amount of data about personal activity relatively inexpensively. These type of devices are part of the quantified self movement - a group of enthusiasts who take measurements about themselves regularly to improve their health, to find patterns in their behavior, or because they are tech geeks. One thing that people regularly do is quantify how *much* of a particular activity they do, but they rarely quantify *how well they do it*. In this project, your goal will be to use data from accelerometers on the belt, forearm, arm, and dumbell of 6 participants. They were asked to perform barbell lifts correctly and incorrectly in 5 different ways. More information is available from the website here: <http://groupware.les.inf.puc-rio.br/har> (see the section on the Weight Lifting Exercise Dataset). 

# Data 

The training data for this project are available here: 

<https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv>

The test data are available here: 

<https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv>

#Data getting


```r
#Remove all objects in the R current environment
rm(list=ls())
#Define the working directory
setwd("C:/Users/AMANIDD/Documents/datasciencecoursera/PracticalMachineLearning")
#Importing provided datasets from the current directory into R
training<-read.csv("pml-training.csv", header=TRUE, dec=".", sep=",")
testing<-read.csv("pml-testing.csv", header=TRUE, dec=".", sep=",")
```

#Data cleaning

Looking at the dataset, we notice than some columns have mainly "NAs" or empty values. So we plan to drop with a missing inexisting values rate over the threshold of 95%. 


```r
#Load required packages
library(caret)
library(kernlab)
library(randomForest)
```


```r
# Defining and threshold level of NA and empty values
max_level <- 0.95 * dim(training)[1]
# Let remove columns which are fully 'NA' or full empty or have more than 95 % NA or empty values
usefull_cols <- !apply(training, 2, function(x) sum(is.na(x)) > max_level || sum(x=="") > max_level)
training1 <- training[, usefull_cols]
```

Next, it is viewable that the 7 firsts are not useful. they will be removed and variable this very low variable this be also dropped.


```r
# Now, let drop 7 first columns which don't seem very usefull
training2 <- subset(training1, select=-c(1:7))
# We can also remove variable with very low variance
nearZvar <- nearZeroVar(training2, saveMetrics = TRUE)
training2 <- training2[ , nearZvar$nzv==FALSE] 
dim(training2)
```

```
## [1] 19622    53
```

Finally, the too correlated variable will not be taken into account for model building.


```r
#Finally, we remove variable strongly correlated with a correlation coefficient more than 95%
corr_matrix <- abs(cor(training2[,-dim(training2)[2]]))
# Making the default diagonal values from '1' to '0', so that these values aren't included later 
diag(corr_matrix) <- 0
# Here we will be removing the columns which are highly correlated
correlated_col <- findCorrelation(corr_matrix, verbose = FALSE , cutoff = .95)
training2 <- training2[, -c(correlated_col)]
dim(training2)
```

```
## [1] 19622    49
```

# Model building 

We first split the training dataset into a training and a test dataset as follow:


```r
#Splitting training2 dataset into 2 datasets
inTrain<-createDataPartition(training2$classe, p=0.9, list=FALSE)
training2A<-training2[inTrain,]
training2B<-training2[-inTrain,]
```

Let now build the model via the randomForest package on the training2A dataset. The confusion matrix obtained will help to calculated the out-of sample error.


```r
#Data Modeling
set.seed(3267)
randMod<-randomForest(classe~., data=training2A, importance=TRUE)
randMod
```

```
## 
## Call:
##  randomForest(formula = classe ~ ., data = training2A, importance = TRUE) 
##                Type of random forest: classification
##                      Number of trees: 500
## No. of variables tried at each split: 6
## 
##         OOB estimate of  error rate: 0.39%
## Confusion matrix:
##      A    B    C    D    E  class.error
## A 5020    1    0    0    1 0.0003982477
## B   13 3398    6    0    1 0.0058513751
## C    0   11 3068    1    0 0.0038961039
## D    0    0   24 2869    2 0.0089810017
## E    0    0    2    6 3239 0.0024638128
```

```r
#Testing previous model on the training2B dataset
predictions<-predict(randMod, newdata=training2B)
#Confusion matrix
confusionMatrix(predictions, training2B$classe)
```

```
## Confusion Matrix and Statistics
## 
##           Reference
## Prediction   A   B   C   D   E
##          A 558   0   0   0   0
##          B   0 379   2   0   0
##          C   0   0 340   0   0
##          D   0   0   0 321   0
##          E   0   0   0   0 360
## 
## Overall Statistics
##                                           
##                Accuracy : 0.999           
##                  95% CI : (0.9963, 0.9999)
##     No Information Rate : 0.2847          
##     P-Value [Acc > NIR] : < 2.2e-16       
##                                           
##                   Kappa : 0.9987          
##  Mcnemar's Test P-Value : NA              
## 
## Statistics by Class:
## 
##                      Class: A Class: B Class: C Class: D Class: E
## Sensitivity            1.0000   1.0000   0.9942   1.0000   1.0000
## Specificity            1.0000   0.9987   1.0000   1.0000   1.0000
## Pos Pred Value         1.0000   0.9948   1.0000   1.0000   1.0000
## Neg Pred Value         1.0000   1.0000   0.9988   1.0000   1.0000
## Prevalence             0.2847   0.1934   0.1745   0.1638   0.1837
## Detection Rate         0.2847   0.1934   0.1735   0.1638   0.1837
## Detection Prevalence   0.2847   0.1944   0.1735   0.1638   0.1837
## Balanced Accuracy      1.0000   0.9994   0.9971   1.0000   1.0000
```

Out of sample error calculation:


```r
#Let check the out of sample error based on the confusion matrix
sum(diag(confusionMatrix(predictions, training2B$classe)$table))/
sum(confusionMatrix(predictions, training2B$classe)$table)
```

```
## [1] 0.9989796
```
 
#Applying model on the testing dataset.


```r
#Apply model on test dataset provided
predictions_test<-predict(randMod, newdata=testing)
predictions_test
```

```
##  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 
##  B  A  B  A  A  E  D  B  A  A  B  C  B  A  E  E  A  B  B  B 
## Levels: A B C D E
```

#Submit the predicted answers in the testing dataset
Using the stub given function, we submit the answers.

```r
# Let run the following function to submit the answer in the testing dataset.
pml_write_files= function(x){
  n = length(x)
  for(i in 1:n){
    filename = paste0("problem_id_",i,".txt")
    write.table(x[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)
  }
}

pml_write_files(predictions_test)
```


