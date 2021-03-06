---
layout: post
title: Extreme Learning Machine
tags: [R, package, R-bloggers]
comments: true
---


As of 2018-06-17 the [elmNN](https://CRAN.R-project.org/package=elmNN) package was archived and due to the fact that it was one of the machine learning functions that I used when I started learning R (it returns the output results pretty fast too) plus that I had to utilize the package last week for a personal task I decided to reimplement the R code in Rcpp. It didn't take long because the R package was written, initially by the author, in a clear way. In the next lines I'll explain the differences and the functionality just for reference.

<br>

### Differences between the elmNN (R package) and the elmNNRcpp (Rcpp Package)

* The reimplementation assumes that both the predictors ( *x* ) and the response variable ( *y* ) are in the form of a matrix. This means that *character*, *factor* or *boolean* columns have to be transformed (onehot encoded would be an option) before using either the *elm_train* or the *elm_predict* function.
* The output predictions are in the form of a matrix. In case of regression the matrix has one column whereas in case of classification the number of columns equals the number of unique labels
* In case of classification the unique labels should begin from 0 and the difference between the unique labels should not be greater than 1. For instance,  *unique_labels = c(0, 1, 2, 3)* are acceptable whereas the following case will raise an error : *unique_labels = c(0, 2, 3, 4)*
* I renamed the *poslin* activation to *relu* as it's easier to remember ( both share the same properties ). Moreover I added the *leaky_relu_alpha* parameter so that if the value is greater than 0.0 a leaky-relu-activation for the single-hidden-layer can be used.
* The initilization weights in the *elmNN* were set by default to uniform in the range [-1,1] *( 'uniform_negative' )* . I added two more options :  *'normal_gaussian' ( in the range [0,1] )* and *'uniform_positive' ( in the range [0,1] )* too
* The user has the option to include or exclude *bias* of the one-layer feed-forward neural network

<br>

### The elmNNRcpp functions


The functions included in the *elmNNRcpp* package are the following and details for each parameter can be found in the package documentation,

<br>


|    elmNNRcpp                                                                              | 
| :------------------:                                                                      | 
|   **elm_train**(x, y, nhid, actfun, init_weights = "normal_gaussian", bias = FALSE, ...)  | 
|   **elm_predict**(elm_train_object, newdata, normalize = FALSE)                           | 
|   **onehot_encode**(y)                                                                    |



<br>

### elmNNRcpp in case of Regression

The following code chunk gives some details on how to use the *elm_train* in case of regression and compares the results with the *lm ( linear model )* base function,

<br>

```R

# load the data and split it in two parts
#----------------------------------------

data(Boston, package = 'KernelKnn')

library(elmNNRcpp)

Boston = as.matrix(Boston)
dimnames(Boston) = NULL

X = Boston[, -dim(Boston)[2]]
xtr = X[1:350, ]
xte = X[351:nrow(X), ]


# prepare / convert the train-data-response to a one-column matrix
#-----------------------------------------------------------------

ytr = matrix(Boston[1:350, dim(Boston)[2]], nrow = length(Boston[1:350, dim(Boston)[2]]),
             
             ncol = 1)


# perform a fit and predict [ elmNNRcpp ]
#----------------------------------------

fit_elm = elm_train(xtr, ytr, nhid = 1000, actfun = 'purelin',
                    
                    init_weights = "uniform_negative", bias = TRUE, verbose = T)
                    
```

```R

## Input weights will be initialized ...
## Dot product of input weights and data starts ...
## Bias will be added to the dot product ...
## 'purelin' activation function will be utilized ...
## The computation of the Moore-Pseudo-inverse starts ...
## The computation is finished!
## 
## Time to complete : 0.09112573 secs

```


```R

pr_te_elm = elm_predict(fit_elm, xte)



# perform a fit and predict [ lm ]
#----------------------------------------

data(Boston, package = 'KernelKnn')

fit_lm = lm(medv~., data = Boston[1:350, ])

pr_te_lm = predict(fit_lm, newdata = Boston[351:nrow(X), ])



# evaluation metric
#------------------

rmse = function (y_true, y_pred) {
  
  out = sqrt(mean((y_true - y_pred)^2))
  
  out
}


# test data response variable
#----------------------------

yte = Boston[351:nrow(X), dim(Boston)[2]]


# mean-squared-error for 'elm' and 'lm'
#--------------------------------------

cat('the rmse error for extreme-learning-machine is :', rmse(yte, pr_te_elm[, 1]), '\n')

## the rmse error for extreme-learning-machine is : 22.00705


cat('the rmse error for linear-model is :', rmse(yte, pr_te_lm), '\n')

## the rmse error for linear-model is : 23.36543

```


<br>

### elmNNRcpp in case of Classification

The following code script illustrates how *elm_train* can be used in classification and compares the results with the *glm ( Generalized Linear Models )* base function,

<br>


```R


# load the data
#--------------

data(ionosphere, package = 'KernelKnn')

y_class = ionosphere[, ncol(ionosphere)]

x_class = ionosphere[, -c(2, ncol(ionosphere))]     # second column has 1 unique value

x_class = scale(x_class[, -ncol(x_class)])

x_class = as.matrix(x_class)                        # convert to matrix
dimnames(x_class) = NULL 



# split data in train-test
#-------------------------

xtr_class = x_class[1:200, ]                    
xte_class = x_class[201:nrow(ionosphere), ]

ytr_class = as.numeric(y_class[1:200])
yte_class = as.numeric(y_class[201:nrow(ionosphere)])

ytr_class = onehot_encode(ytr_class - 1)                                     # class labels should begin from 0 (subtract 1)


# perform a fit and predict [ elmNNRcpp ]
#----------------------------------------

fit_elm_class = elm_train(xtr_class, ytr_class, nhid = 1000, actfun = 'relu',
                          
                          init_weights = "uniform_negative", bias = TRUE, verbose = TRUE)
                          
```

```R

## Input weights will be initialized ...
## Dot product of input weights and data starts ...
## Bias will be added to the dot product ...
## 'relu' activation function will be utilized ...
## The computation of the Moore-Pseudo-inverse starts ...
## The computation is finished!
## 
## Time to complete : 0.03604198 secs

```

```R

pr_elm_class = elm_predict(fit_elm_class, xte_class, normalize = FALSE)

pr_elm_class = max.col(pr_elm_class, ties.method = "random")



# perform a fit and predict [ glm ]
#----------------------------------------

data(ionosphere, package = 'KernelKnn')

fit_glm = glm(class~., data = ionosphere[1:200, -2], family = binomial(link = 'logit'))

pr_glm = predict(fit_glm, newdata = ionosphere[201:nrow(ionosphere), -2], type = 'response')

pr_glm = as.vector(ifelse(pr_glm < 0.5, 1, 2))


# accuracy for 'elm' and 'glm'
#-----------------------------

cat('the accuracy for extreme-learning-machine is :', mean(yte_class == pr_elm_class), '\n')

## the accuracy for extreme-learning-machine is : 0.9337748


cat('the accuracy for glm is :', mean(yte_class == pr_glm), '\n')

## the accuracy for glm is : 0.8940397

```

<br>


### Classify MNIST digits using elmNNRcpp

I found an interesting [Python implementation / Code on the web](https://www.kaggle.com/robertbm/extreme-learning-machine-example) and I thought I give it a try to reproduce the results. I downloaded the MNIST data from my [Github repository](https://github.com/mlampros/DataSets) and I used the following parameter setting in combination with the HOG features of the OpenImageR package,


```R


# using system('wget..') on a linux OS 
#-------------------------------------

system("wget https://raw.githubusercontent.com/mlampros/DataSets/master/mnist.zip")             

mnist <- read.table(unz("mnist.zip", "mnist.csv"), nrows = 70000, header = T, 
                    
                    quote = "\"", sep = ",")

x = mnist[, -ncol(mnist)]

y = mnist[, ncol(mnist)] + 1


# use the hog-features as input data
#-----------------------------------

hog = OpenImageR::HOG_apply(x, cells = 6, orientations = 9, rows = 28, columns = 28, threads = 6)

y_expand = elmNNRcpp::onehot_encode(y - 1)


# 4-fold cross-validation
#------------------------

folds = KernelKnn:::class_folds(folds = 4, as.factor(y))
str(folds)

START = Sys.time()


fit = lapply(1:length(folds), function(x) {
  
  cat('\n'); cat('fold', x, 'starts ....', '\n')
  
  tmp_fit = elmNNRcpp::elm_train(as.matrix(hog[unlist(folds[-x]), ]), y_expand[unlist(folds[-x]), ], 
  
                                 nhid = 2500, actfun = 'relu', init_weights = 'uniform_negative',
                                 
                                 bias = TRUE, verbose = TRUE)
  
  cat('******************************************', '\n')
  
  tmp_fit
})

END = Sys.time()

END - START

# Time difference of 5.698552 mins


str(fit)


# predictions for 4-fold cross validation
#----------------------------------------

test_acc = unlist(lapply(1:length(fit), function(x) {
  
  pr_te = elmNNRcpp::elm_predict(fit[[x]], newdata = as.matrix(hog[folds[[x]], ]))
  
  pr_max_col = max.col(pr_te, ties.method = "random")
  
  y_true = max.col(y_expand[folds[[x]], ])
  
  mean(pr_max_col == y_true)
}))
  
  

test_acc

# [1] 0.9825143 0.9848571 0.9824571 0.9822857


cat('Accuracy ( Mnist data ) :', round(mean(test_acc) * 100, 2), '\n')

# Accuracy ( Mnist data ) : 98.3

```

<br>

The accuracy of the Extreme Learning Machine algorithm is very close to the one of the [KernelKnn using HOG features](http://mlampros.github.io/2016/07/10/KernelKnn/), however it is more than 5 times faster in my operating system in case of a 4-fold cross-validation.

<br>

An updated version of the elmNNRcpp package can be found in my [Github repository](https://github.com/mlampros/elmNNRcpp) and to report bugs/issues please use the following link, [https://github.com/mlampros/elmNNRcpp/issues](https://github.com/mlampros/elmNNRcpp/issues).

<br>

