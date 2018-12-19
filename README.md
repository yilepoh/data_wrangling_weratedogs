# data_wrangling_weratedogs
Data Wrangling and Analysing on the tweets from WeRateDogs from Twitter

## Table of Contents
- [Introduction](#intro)
- [Gathering Data](#gather)
- [Assessing Data](#assess)
- [Cleaning Data](#clean)
- [Analysing and Visualising](#analyse)

<a id='intro'></a>
## Introduction
In our daily encounters with data, data rarely comes in a clean and usable format. As a data analyst, a key skill is to wrangle data. So what do we mean by wangling data?

The data wrangling process includes gathering data from a variety of sources and formats, assessing the quality and tidiness of the data, followed by cleaning the data into a usable format.

In our data wrangling project, there are **3 data sources**:
1. Tweet archives of Twitter user @dog_rates, aka WeRateDogs.
2. Twitter API
3. Additional image prediction data provided by Udacity

This project is separated into **4 sections**
- Gathering data
- Assessing data
- Cleaning data
- Analysing and Visualising

The following libraries are used in the project:
- numpy
- pandas
- os
- json
- requests
- sqlalchemy
- matplotlib

<a id='gather'></a>
## Gathering Data
Gathering data includes downloading the data and importing the data into the jupyter notebook as pandas dataframes.

First, we need to locate the sources of data used in this project. As mentioned above, there are 3 sources of data.
We will go through the sources one by one.

### A. Twitter Archive

Twitter archive is given as a .csv file, pandas has a read_csv module which can be used to import flat files. Here we want to take note of the separator used in the flat file, to correctly import the data into pandas dataframe

### B. Twitter API
Tweepy is used to query the Twitter API, to obtain additional information of:
- retweet_count
- favourite_count
- language

*In this project, Tweepy is not used as the access Twitter Developer account is not available for me.*

However, we can explain the workflow of the codes in Tweepy as below

- import the relavant libraries (tweepy, json, timeit)
- define the conusmer and access key to access Twitter API
- select the tweet id to be queried 
- perform query on twitter's API and collect the data (JSON format) in a text file, tweet_json.txt

From Tweepy, we extract a text file named 'tweet-json.txt'.We will read the text file into a Pandas Dataframe by reading the text file into a list, followed by creating a dataframe from it

The data structure of the file is in json. Which can be access in python via classes like python lists and python dictionaries.
Looking at the data extracted directly from Twitter API, it can be seen that some information are deeply nested within the columns.Since we only require certain information from the original extracted information, we carefully selected information to be used in our analysis.

### C. Image Prediction Data

Image prediction data is provided by Udacity on the [website]('https://d17h27t6h515a5.cloudfront.net/topher/2017/August/599fd2ad_image-predictions/image-predictions.tsv'). 
This data is generated using a neural network to predict the breed of the dog by providing pictures. We use Requests library to programatically download the required files.

When opening the file, we use a resource manager, utilizing the `with` statement, so that the files are closed after executing the codes. Since the file is a tsv file, we could use the pandas read_csv function to parse it into a dataframe.

We have gathered all the raw data from all 3 sources into Pandas Dataframe, namely:
- df_archive
- df_additional
- df_prediction

**df_archive** provides the main structure for our analysis. It contains the tweet_id for tweet with ratings, which is used to query Twitter API to obtain more information

**df_additional** is the data we gathered from Twitter API. It contains the additional information such as 'lang', 'favorite_count', 'retweet_count'.

**df_prediction** is provided and it contains the result of the prediction and the probability of the result being accurate.

<a id='assess'></a>
## Assessing Data
Data provided could potentially have quality and tidiness issues. Which could lead to inaccurate conclusions being drawn.

Below are the questions we ask before we look at the data.
We hope to answer the questions through assessing the imported data.

- Are there missing data?
- What needs to be cleaned? (quality)
- Does the dataframe needs restructured? (tidiness)
- What are the original data? What are the derived data? (check out twitter_archive_enhanced)
- How to merge the 3 dataframe? What is the primary key? (name the merged dataframe as: 'twitter_archive_master.csv')
- What is the date range of the information? (do not need data beyond 1st Aug)
- How to identify original ratings? What is the telltale sign that shows that the information is retweeted?

### Observation

This section is used to document the observation after exploring the datasets

1. in_reply_to_status_id & in_reply_to_user_id are stored in float64 instead of int64
2. retweeted_status_id & retweeted_status_user_id are stored in float64 instead of int64
3. time_stamp is stored as object instead of datetime object
4. source in twitter_archive_enhanced is stored as object instead of category
5. ratings_denominator should be a constant value of 10, but other numbers are observed (non-ideal programatic extraction)
6. ratings_numerator have outlier values (non-ideal programatic extraction)
7. dog_stages (pupper, puppo, doggo, floofer) are stored as separate columns instead of a single column
8. Majority of the dog_stages are 'None' (non-ideal programatic extraction)
9. 'None' are stored as objects instead of NaN in dog_names & dog_stages
10. tweet_api_df have lesser number of data compared to twitter_archive_enhanced
11. dataset contains retweeted and replied tweets, which needs to be removed
12. predictions in the image_predictions are stored as separate columns

After assessing the data, we have found 12 observations that needs to be attended to. We will first categorise them into tidiness and quality. 


### Quality
Quality is related to the content of the data. A clean data enables accurate information to be captured. The dimension of quality data are: completeness, validity, accuracy and consistency (with descending priority). We will list down the quality issues in each dataframe.

#### df_archive
- dog_stages: 'None' are stored as objects instead of NaN
- timestamp: stored as object instead of datetime object
- in_reply_to_status_id, in_reply_to_user_id, retweeted_status_id, retweeted_status_user_id: indicates retweets and replies
- rating_numerator: outlier numbers (>50)
- rating_denominator: outlier numbers (>50)
- source: strings of url provided instead of the name of the source
- source: stored as object instead of categorical object

#### df_additional
- lesser number of data compared to df_archive

#### df_prediction
- lesser number of data compared to df_archive


### Tidiness
Tidiness is related to the structure of the data. A tidy data enables easier analysis

- df_additional and df_archive are part of a single observational unit
- df_archive: pupper, puppo, doggo, floofer are observations of dog_stages variable
- (iterated) df_archive: dog_stages: multiple stages in a single tweet
- df_prediction: p1, p2, p3 are consecutive predictions of the same image, the prediction is a variable

<a id='clean'></a>
## Cleaning Data

We will clean the data one dataframe at a time, starting from df_archive, to df_additional and df_prediction.

Before cleaning, we will make a copy of all the dataframe, adding a suffix '_clean' to the name of the dataframe

### df_archive_clean

#### Define
- convert 'None' in pupper, puppo, doggo, floofer columns into NaN
- create new column named 'None' which has a 'None' entry when there are no dog_stages
- use pd.melt function to convert pupper, puppo, doggo, floofer, None columns into a single column named dog_stages
- select the timestamp column and change the datatype into datetime object
- remove rows with in_reply_to_status_id, in_reply_to_user_id, retweeted_status_id, retweeted_status_user_id
- remove columns: in_reply_to_status_id, in_reply_to_user_id, retweeted_status_id, retweeted_status_user_id, retweeted_status_timestamp
- extract name of source from the url provided in source column using regex
- select source column and change the datatype to categorical
- create a separate dataframe to store the dog_stages information and the tweet_id, remove duplicate and drop the dog_stages column from the main dataframe (df_archive_clean)

**Comments**

---

1. It seems like the unreasonable numerator and denominator numbers are part of the tweet and not an extraction error. We will leave it for now and possibly remove the outliers in the future, if it affects the analysis

2. To resolve the duplicated tweet_id problem, we will create a separate dataframe to store the dog_stages information. This will keep the tweet_id unique for every entry in the main dataframe


### df_additional

#### Define
- merge df_additional_clean on df_archive_clean, using df_archive_clean as the main dataframe, and tweet_id as the primary key

### df_prediction

#### Define
- remove rows where the tweet_id is not found in df_archive_merged
- melt predictions_result, prediction_confidence, prediction_dog

In the df_prediction, I have decided to resturcture the data such that the all the predictions are in a column, with an additional column of prediction_num. My justification is that the the predictions are observations of the same variable.


### Storing the Dataframes
At this point we have 3 cleaned dataframes:
- df_archive_merged (main dataframe)
- df_stages_clean
- df_predictions_clean

We will save all the dataframe as separate csv files and create a sql database

To create a sql database, we first check if the computer already has the file using the OS library. We then set up an 'engine' using sqlalchemy, defining the type of database we want. In this project, we chose a sqlite database, which can be saved locally as a file in our local machine. 

After setting up the engine, we use pandas to_sql function to convert the dataframe into sql databases. The sql database can be read using pandas and it makes editting and analysing the data very simple, as we can bypass the traditional SQL query language. One problem we face during storing process is that there is an operational error using chuncksize=1000. The solution was found on stackoverflow to reduce the chucksize to avoid that error.

After cleaning our data, we stored them into files as below:
- twitter_archive_master.csv
- tweet_dog_stages.csv
- tweet_image_prediction.csv
- database.db

<a id='analyse'></a>
## Analysing and Visualising

In this section we analyse the data and create visualisations for insights.

### Question 1: What is the distribution of the scores given by weratedogs?

**Insights:**

We have plotted a frequency distribution of the ratings tweeted by WeRateDog. During analysis, it is found that there are some outliers which made plotting the histogram not possible. Ratings above 2 were removed from the visualisation as these are rare occurances. 

The frequency distribution is heavily skewed to the left, and the median is greater than the mean rating. Both mean and median ratings are above 1.0. This indicates that WeRateDogs are very generous in their ratings, most likely due to their love for dogs. It is also possible that the dogs submitted to WeRateDogs are very good, since only owners who are proud of their dogs would take time to submit the pictures of their dog to be rated.

### Question 2: What is the correlation between favourite count and retweet count?

**Insights:**

We have plotted a scatter plot with x-axis as the favorite count and y-axis and the retweet count.
There is strong positive correlation between the favorite count and the retweet count. The spread is lower at low favorite counts as compared to higher favorite counts.

From the scatter plot, we find that Twitter users generally uses the favorite and retweet function side by side, when they encounter the tweet the like.

### Question 3: Does the use of dog_stages affect the favourite and retweet counts?

We will do a short analysis (A/B Testing) on the effect of using the terms in dogtionary on the likability of the tweet. The likability of the tweet is indicated by the retweet count and the favourite count

We define our null hypothesis as: mean retweet count for the tweets using words from the dogtionary is the same as the mean retweet counts for the tweets not using words from the dogtionary.

Let N = mean retweet count 

$$H_0: N_{dogtionary} - N_{normal} = 0$$
$$H_1: N_{dogtionary} - N_{normal} \neq 0$$

**Insights:**

From the A/B testing, we obtained a p-value of 0.0076, which is lower than the alpha value. We can reject the null hypothesis which says there is no difference in retweet count between tweets using dogtionary terms and tweets not using them. There is statistical significance in the difference between the retweet counts.

In a practical sense, an increase of ~1200 in the retweet count is relavant as most of the tweets have retweet counts below 10,000. An increase of 1200 would equate to a 10% increase. 

Therefore we suggest the use of terms in dogtionary to increase the retweet counts.
