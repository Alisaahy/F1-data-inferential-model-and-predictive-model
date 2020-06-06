# F1-data-inferential-model-and-predictive-model

The project is based on F1 dataset, which contains data about drivers, races, constructors, results, etc. of F1 races from 2010. The project contains 2 parts, one inferential part and one prediction part. 

## Inferential model: build a model to explain why a driver arrives in second place in a race between 1950 and 2010. 
In this part, I fit a model using features that make theoretical sense to describe F1 racing between 1950 and 2010. Basically, I did these steps:

- Import and load datasets from s3 bucket.

- Feature engineering: select and transform features by making reasonable theoretical inference. Features I add to the original datasets are:/
1. Age: Age matters in F1 drivers' career path. Mostly, late 20s and early 30s can be a driver's peak year. So, we can hypothesize that drivers who are in their late 20s or early 30s are more likely to achieve higher position in F1 race. Therefore, I added age feature, which is the (race date - driver's birth date) to the dataset to see whether age can help us explain why a driver arrives in second place in a race.
2. Qualifying: Qualifying is a session to determine driver's grid in a race. It is split into three parts, called Q1, Q2, and Q3. Qualifying position can reflect the driver's competence to some extent. So, I calculated the average qualifying time from a driver's Q1, Q2 and Q3 sessions.
3. Fastest lap: In F1, the fastest lap is the quickest lap run during a race. It's reasonable for us to hypothesize that the driver's fastest lap time may be positively related to the driver's final position. We can also hypothesize which lap the driver has the fastest speed can affect the driver's final position. 
4. StatusId: T hundreds of statuses that a driver can experience for a race, such as accident, collision, overheating, etc. These different kinds of status will largely determine a driver's final position. It's highly possible that drivers who ranks higher are those who finish their race. 
5. Grid: Grid position can affect the race. For example, pole sitter will have an advantage at the start, because they do not need to follow any other cars. So, I used grid as a useful feature in the dataset

- Build inferential model:
I chose Bayesian Additive Regression Trees (BART) model to inferentially explain why some drivers arrive in the second place. I trained BART on all of our data, and separated only the Y vector from all of our predictors to create two matrices.

Why Bart:
1.	It naturally identifies interactions and non-linearities among predictors
2.	It is straightforward to estimate uncertainty

I used 3 metrics to evaluate my model:
1.	Mean Absolute Error (MAE)
2.	Mean Squared Error (MSE)
3.	Root Mean Squared Error (RMSE)

- Explain the most important variable in the model:
To investigate the importance of a feature, I built a function to intuitively compare the model with all features, which is the baseline model and the model without the feature we concern about its importance. I calculated RMSE of these two models and the difference between those two RMSE can be seen as a proxy of how much RMSE will decrease if we add this feature to the model, which also is kind of importance of the feature.

- Explain marginal effects for some important variable:
In this part, I compute the marginal effects of 2 variables: statusId and qualifying_position. To compute marginal effects, I used the predictive posterior generated by fitting BART to all our data and create predictions changing the values of specific predictors to values of interest. For example, in my case, I was interested in whether a driver can rank second in a race given a specific number of the driver's qualifying_position for races which the driver finished (statusId == 1). So I created a new x_train matrix where all values of statusId are set to 1 and qualifying_position to a specific value between 1 and 24, maintaining the values of all other predictors unchanged, Then I used this new x_train matrix and BART's predictive posterior to generate synthetic values for each combination of statusId and qualifying_position. This produces a new y_test matrix that summarizes L = 1000 simulatons for ach of our N = 3490 events, which we summarize by averaging over N to get a single M x 1 vector of synthetic values for each combinations of values we are iterating over. Then I collected those vectors in a matrix of simulations with all columns for each value of qualifying_position we are allowing to change

## Prediction model: build a model to predict which driver can come in a second place in races between 2011 and 2017, using data from 1950 to 2010.
To predict second place between 2011 and 2017, I built a step ahead forecasting model with rolling window cross validation. My hypothesis was, among all drivers from 1950 to 2010, drivers who can participate in races from 2011 to 2017 are highly possible to also be active in the previous 6 years, which is 2004 to 2010. I confirmed this hypothesis in 1950-2010 dataset. 
So, I used the driver data from 2004 to 2010 to foresee who can get a second rank in 2011 to 2017. To build the train set, I extracted driver data from 1990 to 1996 and attached the second rank driver from 1997 to 2003 as the target variable. To test the model, I used driver data from 1997 to 2003 with the target variable, which is second driver from 2004 to 2010. And if the model works on both the train set and test set, I'll apply it to 2004 to 2010 and predict who of them can get the second rank in the coming period.

To build the model, I also did these following steps:

- Feature engineering: features I add to the original datasets include:
1. History ranking: the driver's history records statistics in the past period, including the driver's highest position, lowest position, average position, most frequent position and the standard deviation of position.
2. race_count and recent_race: how many races the driver has participated in in this time period, and when is his most recent race.
3. constructor_count and change_couns_date: whether the driver has changed his constructor in this time period. If changed, how many times has he changed and what is the last time he changed the constructor.
4. finished_percent: could the driver always finish all the laps in the past race, and what is the percentage that he finished all the laps.
5. min_quali, max_quali, mean_qualy, std_quali: what is the driver's highest, lowest and average mean_qualifying_time in the past race, what is the standard deviation of his mean_qualifying_time.
6. most_status: what is the driver's most status id in the past races

- Model building:
I chose random forest model for prediction. I firstly did a grid search to find the best hyperparameters of my model. To do the grid search, l defined a customer cross validation function, from which I can simply use 1990 data for training and 1997 data for validation.
Then I used the optimized random forest model from the above grid search and fit it to my dataset. The train data will be the 1990 dataset and the test data is the 1997 dataset. 

- Model evaluation:
I gathered some evaluation metrics from the model, including precision, recall for the train set and test set, as well as F1 and the confusion matrix for the test set. Precision is a ratio of the number of true positives divided by the sum of the true positives and false positives. Recall is the ratio of the number of true positives divided by the sum of the true positives and the false negatives. And F1 score is the harmonic mean of precision and recall. Confusion matrix can give us a direct result of model prediction.

- Explain the most important variable:
To get feature importance of variables, I use the feature importance package in sklearn for tree models. In tree models, every node is a condition of how to split values in a single feature, so that similar values of the dependent variable end up in the same set after the split. The condition is based on impurity, which in case of classification problems is Gini impurity. So when training a tree we can compute averaging the decrease in impurity over trees each feature can contribute to.
