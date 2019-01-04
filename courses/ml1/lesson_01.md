
# Lesson 1 - random forests

- Length:  01:18
- Video:  https://www.youtube.com/watch?v=CzdWqFTmn0Y&feature=youtu.be
- Notebook:  [lesson1-rf.ipynb](https://github.com/fastai/fastai/blob/master/courses/ml1/lesson1-rf.ipynb)  

---

## Getting data using `curl`
https://www.kaggle.com/c/bluebook-for-bulldozers

- ML should help us understand a dataset, not just make predictions about it.

Firefox, to website, then Javascript console, Developer
- ctrl + shift + i to bring up web developer tool
- tab to Network
- go to data row
- right click, copy as Curl (unix command that downloads data, like `wget`)
- might want to delete "2.0" in url since it causes problems
- `curl url_link -o bulldozers.zip` `-o` means output, then give suitable file name
- `mkdir bulldozers`
- `mv bulldozers.zip bulldozers/`
- `sudo apt install unzip` or `brew install unzip`
- `unzip bulldozers.zip`

Python 3.6 format string:  
```python
df_raw = pd.read_csv(f'{PATH}Train.csv', low_memory=False, 
                     parse_dates=["saledate"])
``` 
- `f'{PATH}Train.csv'`  the `f` tells it to interpolate the "{PATH}"
- `low_memory=False` make it read more of the file to decide what the types are  

### Example
`name = 'Jeremy'`  
`age = 43`  
`f'Hello {name.upper()}, you are {age}'`  
output:  
>Hello, Jeremy, you are 43  

### Random Forest
- universal machine learning technique
- way of predicting something of any kind (dog/cat, price)
- can predict a categorical or continuous variable
- columns can be of any kind (pixel data, zip codes, revenues)
- in general, it doesn't overfit
- easy to stop it from overfitting
- don't need a separate validation set
- has few, if any statistical assumptions
  - doesn't assume data is normally distributed
  - doesn't assume relationships are linear
  - don't need to specify interactions
- requires few pieces of feature engineering (don't have to take log of data)
- it's a great place to start
- if your random forest doesn't work, it's a sign there is something wrong with the data

Both Curse of Dimensionality & No Free Lunch are largely false.  

#### Curse of Dimensionality - idea that the more columns you have, it creates more columns that are empty; that the more dimensions you have, the more they sit on the edge; in theory, distance between points is much less meaningful.  
- points **do** still have distance from each other
- in the 90's, theory took over machine learning
- we lost a decade of real practical development with these theories
- in practice, building models on lots and lots of columns works well

#### No Free Lunch Theorem
- there is no type of model that works well for any kind of dataset
- but, in the real world, that's not true; some techniques **do work**
- ensembles of decision trees works well

### sklearn
- RandomForestRegressor is part of `sklearn`, `scikit learn`
- Scikit learn is not the best, but perfectly good at nearly everything; popular library
- next part of course (with Yannet), will look at different kind of decision tree ensemble, called Gradient Boosting Trees, XGBoost which is better than gradient boosting trees in scikit learn

`from sklearn.ensemble import RandomForestRegressor, RandomForestClassifier`  
- RandomForestRegressor - predicts continuous variables  
- RandomForestClassifier - predicts categorical variable

## Convert to Pandas Categories
The categorical variables are currently stored as strings, which is inefficient, and doesn't provide the numeric coding required for a random forest. Therefore we call train_cats to convert strings to pandas categories.  
This is a fastai library function:  
`train_cats(df_raw)`  

## re-order Pandas categories
```python
df_raw.UsageBand.cat.categories
Out[9]:
Index(['High', 'Low', 'Medium'], dtype='object')
In [10]:
df_raw.UsageBand.cat.set_categories(['High', 'Medium', 'Low'], ordered=True, inplace=True)
```  
In the background, the code is 0, 1, 2 for the categories which is what is used in Random Forest.  -1 is assigned to NA.

## get percent of missing values for each column
We're still not quite done - for instance we have lots of missing values, wish we can't pass directly to a random forest.
`display_all(df_raw.isnull().sum().sort_index()/len(df_raw))`  
```bash
Backhoe_Mounting            0.803872
Blade_Extension             0.937129
Blade_Type                  0.800977
Blade_Width                 0.937129
Coupler                     0.466620
Coupler_System              0.891660
Differential_Type           0.826959
Drive_System                0.739829
Enclosure                   0.000810
```  

## save to feather format
- But let's save this file for now, since it's already in format can we be stored and accessed efficiently.
- saves to disk the same way it appears in RAM
- **feather** is fairly new
- fastest way to save to disk and fastest way to read it back  
```python
os.makedirs('tmp', exist_ok=True)
df_raw.to_feather('tmp/bulldozers-raw')
```  
So, then we don't have to re-run everything from start of notebook.  
#### Pre-processing
In the future we can simply read it from this fast format.
```python
df_raw = pd.read_feather('tmp/bulldozers-raw')
```
## Run Random Forest
- serial number (numbers): random forest works fine with these ID numbers that are not really continuous
- random forests are trivially parellelizable
- means it will split up data across CPUs and linearly scale
- `n_jobs=-1` means create a separate job for each CPU that you have

### Kaggle Competition
- generally speaking, if you're in the top half of a Kaggle competition, you're doing well
- so, here, with no thinking and using the defaults of the algorithm (random forest), we're in the top quarter of the competition
- random forests are insanely powerful

### HW
- take as many Kaggle competitions as you can
- try this process with Pandas set up
- 







