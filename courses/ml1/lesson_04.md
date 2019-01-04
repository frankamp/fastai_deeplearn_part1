# Lesson 4: RF Hyperparameters & Feature Importance

Length: 01:40  
Video:  [Lesson 4](https://www.youtube.com/watch?v=0v93qHDqq_g&feature=youtu.be)  
Notebook:  [lesson2-rf_interpretation.ipynb](https://github.com/fastai/fastai/blob/master/courses/ml1/lesson2-rf_interpretation.ipynb)  

---

## Topics
- R^2 accuracy
- How to make validation sets
- Test vs. Validation Set
- Diving into RandomForests
- Examination of One tree
- What is 'bagging’
- What is OOB Out-of-Box score
- RF Hyperparameter 1: Trees
- RF Hyperparameter 2: max Samples per leaf
- RF Hyperparameter 3: max features

## Repository / Notebook Workflow
- make a copy of the notebook
- name it with `tmp` prefix; this will then be ignored by `.gitignore`

## Hyperparameter `set_rf_samples()`  
- pick up a subset of rows
- summarize relationship between hyperparameters and its effects on overfitting, collinearity
- reference:  https://github.com/fastai/fastai/blob/master/courses/ml1/lesson1-rf.ipynb
- `set_rf_samples(20000)` determines how many rows of data in each tree
  - Step 1: we have a big dataset, grab a subset of data and build a tree
  - we either bootstramp a sample (sample with replacement) or subset a small number of rows
  - Q:  assuming the tree remains balanced as we grow it, how many layers deep would we want to go?  
    - A: log_2(20,000)  (depth of tree doesn't really vary based on sample size)
  - Q:  how many leaf nodes would there be?
    - A: 20,000  (because every leaf node would have a sample in it)
  - when you decrease the sample size, it means that there are less final decisions that can be made; tree will be less rich; it also is making less binary choices to get to those decision
  - setting `set_rf_samples` lower means you overfit less, but you'll have a less accurate tree model
  - each individual tree (aka "estimator") is as accurate as possible on the training set
  - across the estimators, the correlation between them is as low as possible, so when you average them out together, you end up with something that generalizes
  - by decreasing the `set_rf_samples()` number, we are actually decreasing the power of the estimator and increasing the correlation
  - it may result in a better or worse validation set result; this is the compromise you have to figure out when you do ML models
 - `oob=True` whatever your subsample is, take all the remaining rows, and put them into a dataset and calculate the error on those (it doesn't impact the training set); it's a quasi-validation set

## Information Gain
- "Information" used to describe the amount of additional info we gain from splitting
- how much better did the model get by adding another split point?

## Hyperparameter `min_samples_leaf`  
- Q:  if I change min_samples_leaf from 1 to 2, what would be my new **depth**?
  - A:  log_2(20,000) - 1
- Q:  how many leaf nodes would there be in that case?
  - A:  10000
- we have less depth, less decisions to make, and we have a smaller number of leaf nodes
  - we would expect each estimator to be less predictive, but also less correlated and result in less overfitting
- could speed up training with one less level; could generalize better
- TRY these options
  - 1, 3, 5, 10, 25, 100

## Hyperparameter `max_features`  
- `max_features=0.5` at each point in the tree, we pick a different half of the features 
- we do this because we want the trees to be as rich as possible
- picking a random subset of features at every decision point
- overall effect is that each individual tree will be less accurate, but the trees will be more varied
  - imagine if you had one feature that was super-predictive, so predictive that every single sub-sample split on the same feature
  - trees would have same intial split
  - some trees would create other splits, show interactions
  - gives more variation, creates more generalized trees
- TRY these options
  - `None`
  - `0.5`
  - `sqrt`

## Things that don't impact our training
- `n_jobs=-1` how many CPUs to run on
  - making more than 8 may have diminishing returns
  - -1 --> all cores
  - 1  --> default
- `oob=True` if you don't say "True", it won't print it out

## Other Parameters
- there are more hyperparameters
- the one's highlighted here are the ones that Jeremy has found useful
- you can try others

## Random Forest Model Interpretation
- fastai library is not available in Kaggle kernels
- can look in `fastai.structured`; use `??fastai.structured` to look inside it
- most of methods we use are a small number of lines of code; can copy functions and use whereever
- can link to library; give credit to fastai
- "Confidence based Tree Variance" does not exist anywhere else; is in fastai
- "Feature Importance" exists in Kaggle kernels

## Feature Importance
- works by randomly shuffling a column
- `set_rf_samples()` to a number where you can run a model < 10 seconds or so.  Ex:  50,000
- `rf_feat_importance` works by randomly shuffling a column

## Feature Importance in "CLASSIC TRADITIONAL STATISTICAL TECHNIQUES" (outside of ML) 
- in psychology, economics, psychology, marketing, etc
- assuming linear relationships between Xvars and Y (with a possible link function that could be sigmoid)
- determine feature importance by looking at weight vars, or coefficients (aX1 + bX2 + ... = Y); normalize first
- Note: if you were missing an interaction, or a transformation, if pre-processing were imperfect, than coefficients would be wrong
  - in your totally WRONG model, this is how important your coefficients are
- AND, _if_ they have done significant pre-processing that the model is accurate, now we're looking at coefficients of PCA, or clusters, which are difficult to interpret.

## Feature Importance in Random Forest
- in this extremely high parameter, highly flexible functional form with few if any statisticial assumptions, this is your feature importance

`41:00`  
## Machinery Example 
- we'll see 4 main important features

## One Hot Encoding
- Case 1:  multiple codes to one variable:  0, 1, 2, 3, 4, 5 (tree will have to do multiple splits to identify variable that has signifigance)
- Case 2:  with one hot encoding, the random forest can split between 0 and 1; it has the ability in a single step, to pull the category level of significance
- 1-hot encoding 
- `max_n_cat=7` if column value is say, a zip code, which has > 7 values, it will be left as a number
- always try 1-hot encoding for quite a few of your vars and look at feature importance

## Removing Redundant Features
- cluster analysis:  find which group of rows or columns are similar
- a common type of clustering is: **k-means**
  - assume we have no labels
  - we need to specify the number of clusters
  - move points closer to centroids, iterative process
- another type clustering:  **hierarchical** or **agglomerative** (underused, was more popular 20-30 years ago)
  - look at every pair of objects and see which are closest.  remove 2 points and replace with their mean.
  - keep doing that; we'll gradually reduce the number of points by pairwise combining
- can use rank corrrelation to identify similar features  (function must be monotonic for rank correlation to work)
- set `set_rf_samples=50000` to run things quickly

## Partial Dependence
1:07:30  
```python
from pdpdbox import pdp
from plotnine import *
```
- is a powerful technique
- not a lot of people know about it
- for the features which are important, how do they relate to the dependent variable?
- look at relationships, and then ask "what happened?  what [external factor] could be causing this?
- replacing whole column with constant.., 1961
- plot all 500 predictions

### Plot year made vs sale elapsed
- note that year made = 1000 ---> go to client, they tell us that "1000" is for when we don't know the year of the make
- in order to understand the plot better, remove the items that were made before 1930
- next, let's look at the relationship between year made and sale price

### ggplot
- there's a great package called ggplot
- it was originally in the R package
- "gg" = **Grammar of Graphics**
- powerful way of producing plots in a flexible way
- can pip install, already part of fastai environment
- similar Python API as in R (though R has better documentation)
- when you do plots, most datasets you will use will be too big
  - there are so many points, it will take forever, and may look like a big mess
- that's why Jeremy uses `get_sample` first:
  - this grabs 500 data points, using a random sample
  - `aes` = aesthetic (way to set up columns in ggplot)
- in ggplot `+` adds chart elements
  - add a smoother
  - a smoother creates a little linear regression for a subset of the graph, and then joins them 
  - this allows us to see a nice smooth curve
- this is the main way Jeremy looks at univariate relationships
- by adding `se=True`, it also shows the confidence interval of the smoother
```python
x_all = get_sample(df_raw[df_raw.YearMade>1930], 500)
ggplot(x_all, aes('YearMade', 'SalePrice')) + stat_smooth(se=True, method='loess')
```
### ggplot - interpretation
- upon looking at the graph, it's all over the place
- Jeremy would have expected that trucks sold more recently would be more expensive because of inflation and newer models
- when you look at a linear relationship like this, there are a lot of interactions 
- for example, why did the price drop between 1991 and 1997?
  - are they less valuable?
  - was there a recession then?
  - or maybe people at that time were buying vehicles that were less expensive?
`1:14:07`  
- as data scientists working at a company, people will come to you with univariate charts: "what happened / why?"
  - most times, there is something else going on
- ask this Q: what's the relationship between sale price and year made, all other things being equal?
  - "all other things being equal" means if we sold something in 1980 vs 1990, it was exact same thing, to the exact same person, with exactly the same options, etc., what would be the difference in price? ---> use Partial Dependency
  
### Partial Dependence Plot (`01:15`)
- nice library that is not well-known called `pdp` --> `pip install pdpbox` --> https://github.com/SauceCat/PDPbox
- we have our sample dataset of 500 data points
- we will do something very interesting:  
  - using our data 500 x num_covariates
  - in the column year_made, we will copy in the value "1960", for all 500 rows
  - then, we will take take this data and pass to our random forest, to predict the **sale price**
  - that will tell us, that for everything that was auctioned, how much do we think it would have been sold for, if that item was made in 1960?
  - we will plot that (year=1960 by selling price), for each of the years
  - we will run the model again, by setting each year now to 1961, etc.
  - Q from student: to be clear, we have already fit the random forest and then we pass in new year and see what the price will be?
  - A from JH:  Yes, this is a lot like how we did random forest, but rather than randomly shuffling the column, we replace the column with a constant value
  - randomly shuffling a column tells us how accurate it is when you don't use that column anymore
  - replacing the entire column with a constant tells us, or estimates for us, how much we would have sold that product for on that auction, on that day, in 1961
  - 


