Does imputing model labels using the model predictions can improve it's performance?
================
Iyar Lin
20 December, 2018

-   [Motivation](#motivation)
-   [The proposed procedure](#the-proposed-procedure)
-   [Simulation study](#simulation-study)
    -   [The benign case](#the-benign-case)
    -   [The ugly case](#the-ugly-case)
-   [Conclusion](#conclusion)

Motivation
==========

In some scenarios a data scientist may want to train a model for which there exists an abundance of observations, of which only a small fraction is labeled, making the sample size available to train the model rather small. Although there's plenty of literature on the subject (e.g. "Active learning", "Semi-supervised learning" etc) one may be tempted (maybe due to fast approaching deadlines) to train a model with the labelled data and use it to impute the missing labels.

While for some the above suggestion might seem simply incorrect, I have encountered such suggestions on several occasions and had a hard time refuting them. To make sure it wasn't just the type of places I work at I went and asked around in 2 Israeli (sorry non Hebrew readers) machine learning oriented Facebook groups about their opinion: [Machine & Deep learning Israel](https://www.facebook.com/groups/543283492502370/permalink/1158551544308892/) and [Statistics and probability group](https://www.facebook.com/groups/statprob/permalink/767687730239611/). While many were referring me to methods discussed in the literature, almost no one indicated the proposed method was utterly wrong.

This post is intended to prove to myself and anyone else interested that imputing labels with your model predictions can't help and in some situations may even hurt your model performance.

The proposed procedure
======================

Formally speaking, given feature matrix ![X](https://latex.codecogs.com/png.latex?X "X"), labels ![y](https://latex.codecogs.com/png.latex?y "y") and a set of indices ![I](https://latex.codecogs.com/png.latex?I "I") for which the labels are missing the procedure discussed is:

1.  Train "small" model ![\\hat{f\_S}(x)](https://latex.codecogs.com/png.latex?%5Chat%7Bf_S%7D%28x%29 "\hat{f_S}(x)") based on the observations ![\\{X\_i,y\_i\\}, \\, i \\in \\{1 \\dots n\\} \\setminus I](https://latex.codecogs.com/png.latex?%5C%7BX_i%2Cy_i%5C%7D%2C%20%5C%2C%20i%20%5Cin%20%5C%7B1%20%5Cdots%20n%5C%7D%20%5Csetminus%20I "\{X_i,y_i\}, \, i \in \{1 \dots n\} \setminus I")
2.  Predict the labels ![\\hat{y}\_i=\\hat{f\_S}(x\_i), \\, i \\in I](https://latex.codecogs.com/png.latex?%5Chat%7By%7D_i%3D%5Chat%7Bf_S%7D%28x_i%29%2C%20%5C%2C%20i%20%5Cin%20I "\hat{y}_i=\hat{f_S}(x_i), \, i \in I")
3.  Train "large" model ![\\hat{f\_L}(x)](https://latex.codecogs.com/png.latex?%5Chat%7Bf_L%7D%28x%29 "\hat{f_L}(x)") on the full sample ![\\{X, y\\}](https://latex.codecogs.com/png.latex?%5C%7BX%2C%20y%5C%7D "\{X, y\}")

Simulation study
================

The benign case
---------------

First, simulate our full feature matrix ![X](https://latex.codecogs.com/png.latex?X "X"):

``` r
# I'm going to use a correlation matrix from which I'll simulate the feature matrix X. 
# I'll use the mtcars dataset to get some "real world" parameter values. 
# In the next few lines of code I closely follow this great post on r-bloggers: https://www.r-bloggers.com/simulating-data-following-a-given-covariance-structure/

correlation_matrix <- cor(mtcars %>% select(mpg, disp, hp, wt))

# Cholesky decomposition
L <- chol(correlation_matrix)
nvars <- dim(L)[1]
nobs <- 1000
# Random variables that follow an M correlation matrix:
sim_data = setNames(object = as.data.frame(t(t(L) %*% matrix(rnorm(nvars*nobs), nrow=nvars, ncol=nobs))), 
                    nm = c("pred1", "pred2", "pred3", "pred4"))
```

Next we'll assume a linear regression model and simulate the labels ![y](https://latex.codecogs.com/png.latex?y "y") for the full dataset, then split it to train and test sets:

``` r
beta <- setNames(object = c(0.3, -0.2, 0.5, 0.3, -0.6), nm = c("beta0", "beta1", "beta2", "beta3", "beta4"))
sim_data$y <- as.numeric(cbind(1, as.matrix(sim_data)) %*% beta) + rnorm(nobs)

train_data <- sim_data[1:800, ]; test_data <- sim_data[801:1000, ]
```

Assuming the data scientist has labels for the first 20 observations.

``` r
available_train_data <- train_data
available_train_data$y[21:800] <- NA
```

We start implementing the above procedure by training the small model on the labelled data:

``` r
small_model <- lm(y ~ ., data = available_train_data)
```

Imputing the rest of the labels and training our model on the full train sample:

``` r
imputed_labels <- predict(small_model, available_train_data[21:800, ])
available_train_data$y[21:800] <- imputed_labels
large_model <- lm(y ~ ., data = available_train_data)
```

Next we'll measure the performance (Root mean squared error, lower is better) of the small and large models on the test set:

``` r
RMSE <- function(y, y_hat) sqrt(mean((y - y_hat)^2))
small_model_performance <- RMSE(y = test_data$y, y_hat = predict(small_model, test_data))
large_model_performance <- RMSE(y = test_data$y, y_hat = predict(large_model, test_data))

pandoc.table(data.frame(model = c("small model", "large model"), RMSE = c(small_model_performance, large_model_performance)))
```

<table style="width:29%;">
<colgroup>
<col width="19%" />
<col width="9%" />
</colgroup>
<thead>
<tr class="header">
<th align="center">model</th>
<th align="center">RMSE</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="center">small model</td>
<td align="center">1.008</td>
</tr>
<tr class="even">
<td align="center">large model</td>
<td align="center">1.008</td>
</tr>
</tbody>
</table>

We can see that imputing the labels didn't improve neither did it hurt the model performance.

The ugly case
-------------

There are cases however where the above procedure can be harmful. In some organizations one has access to man-power that can manually label observations. Since such labelling process is costly in time and money, one may try to "enhance" it with additional labels from the small model using the above procedure. In the simulation below I demonstrate this has the effect of "diluting" the information gained from the manual labelling process thus reducing its effectiveness.

We begin with a train data where only the first 20 observations are labelled:

``` r
available_train_data <- train_data; available_train_data$y[21:800] <- NA
```

Next we have 180 more observations tagged:

``` r
available_train_data$y[21:200] <- train_data$y[21:200] # this represents manual labelling of additional 180 observations
```

Next we impute the rest of the labels using our small model and train our model on the full train sample:

``` r
new_small_model <- lm(y ~ ., data = available_train_data)
imputed_labels <- predict(small_model, available_train_data[201:800, ])
available_train_data$y[201:800] <- imputed_labels
new_large_model <- lm(y ~ ., data = available_train_data)
```

Below are the resulting RMSE:

``` r
new_small_model_performance <- RMSE(y = test_data$y, y_hat = predict(new_small_model, test_data))
new_large_model_performance <- RMSE(y = test_data$y, y_hat = predict(new_large_model, test_data))

pandoc.table(data.frame(model = c("new small model", "new large model"), RMSE = c(new_small_model_performance, new_large_model_performance)))
```

<table style="width:36%;">
<colgroup>
<col width="25%" />
<col width="11%" />
</colgroup>
<thead>
<tr class="header">
<th align="center">model</th>
<th align="center">RMSE</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td align="center">new small model</td>
<td align="center">0.9733</td>
</tr>
<tr class="even">
<td align="center">new large model</td>
<td align="center">0.9856</td>
</tr>
</tbody>
</table>

We can see that while both models improved, the large model with imputed labels improved less than the small model using only the observations with labels.

Conclusion
==========

If you face a problem of sparse labels and want to improve your model, try some of the methodologies developed to deal with that situation. Don't impute the labels using the model you're trying to improve.
