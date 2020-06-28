# Predictive-ML-Model-Twitter

<!-- markdownlint-disable MD026 -->

In the era that we currently live in, all the focus has shifted towards data. Each day, the amount of data that is generated and consumed is increasing, adding somewhere around 5 exabytes of data. Everything we do generates data, be it turning on and off the light, or commuting from home to work. This data can be used to generate information that can be used for insights to predict and extract patterns. Data Mining or Data Science is the term that has taken the industry abuzz. It is the process of discovering patterns, insights, and associations from data. In this how-to guide we’ll learn how to use data and implement a predictive model on it to get insights. Our intended audience include developers, general users with basic knowledge of programming, and organizations that want to enhance customer experience. It will enable a user to create a predictive model on Watson Studio, which is a cloud-based environment for Data Scientists. By using this how-to user can predict and optimize their twitter interaction and would lead to optimum traffic on their tweets.

## Learning objectives

After completing this how-to, the reader will be able to:
* Work with Cloud Functions to extract data from Twitter 
* Learn to create upload a CSV file in COS from a Cloud Function.
* Learn Watson Studio and AutoAI to build a predictive model using CSV data.
* Leverage Twitter to predict and optimize their twitter interactions.

## Prerequisites

* IBM Cloud account - [sign up](https://cloud.ibm.com/registration?cm_sp=ibmdev-_-developer-tutorials-_-cloudreg/) if you don't have an account yet.
* A [Twitter account](https://twitter.com/)
* A [Twitter Developer account](https://developer.twitter.com/)

## Estimated time

To complete this tutorial it should take around 1 hour.

## Steps

### Use sample data or get your own?

The first thing we'll need to do is get a bunch of tweets to analyze. In this step we'll go through how to get a bunch of tweets, but if you're not interested in doing that, we provide a sample data set:

* **[ufone_tweets.csv](static/ufone_tweets.csv)**: Tweets from a Ufone, a phone operator, cleaned up and ready for Watson Studio. (Use this one!)
* **[ufone_tweets_raw.csv](static/ufone_tweets_raw.csv)**: Same as above, but raw, taken directly from tweepy. (Only added for completeness.)

### Step 1: Getting Twitter API access (optional)

*If you're using the sample data, then skip to Step 3.*

Before we use [tweepy](https://github.com/tweepy/tweepy) to get tweets we need to generate our Consumer api keys :<br><br>
Go to your [Twitter Developer account](https://developer.twitter.com/),  hover over your name on the top right create your app. Fill the information required.

![](images/twitter1.png)

Once your app is created select the Keys and Tokens tab. You will see your Consumer API key and Consumer API secret key which we will be using later in the tutorial. These can be revoked and regenerated, but as with any other key, you should keep these secret. (Here we won't be using the api tokens so you can ignore them)

![](images/twitter2.png)

### Step 2: Creating a Cloud Object Storage (COS) service in IBM Cloud

*Again, if you're using the sample data, then skip to Step 3.*

Log into your ibm cloud account at https://cloud.ibm.com/login, click on Create Resource and search for Object Storage. 

![](images/cos1.png)

Choose the lite plan which is free, change the name if you want to and click on Create.

![](images/cos2.png)

You can now find your object storage instance created in ressources under Storage. Once you open your instance click on Buckets from the left side panel and create a bucket (You can choose any type of bucket). Make sure to note down the name of your bucket once you create it.

![](images/cos3.png)

Go to service crendentials from the panel, select the service credential that has just been created. If nothing is there then click on "New credential". Click on the arrow to expand the credentials. Note down  the "api_key" and the "iam_serviceid_crn".

![](images/cos4.png)

Go to endpoint from the panel. Choose your resilency and region, and note down the private url since it will be needed later in our cloud function.

![](images/cos5.png)

Our bucket is now ready, make sure to have your:
 * Bucket name
 * Api Key
 * Service ID
 * Endpoint Url
 
 ### Step 3: Create a Cloud Function
 
Usually we create cloud functions directly from IBM cloud, but in our case, we want to use [tweepy](https://github.com/tweepy/tweepy) which is an external Python library for accessing Twitter API. External libraries are not supported in the Cloud Function runtime environment. We will have to write our python code and package it with a local environemnt in a .zip file, and then push it to IBM Cloud.<br><br>

If you don't have Python, then [download and install the latest version](https://www.python.org/downloads/). Once installed make sure to install **virtualenv**. 
```dos
pip install virtualenv 
``` 
<br>Create a directory that you can use to create your virtual environment. In this example I named it twitterApp
```console
cd desktop; mkdir twitterApp; cd twitterApp
```

<br>From the twitterApp directory, create  virtual environment named virtualenv. Your virtual environemt must be named virtualenv
```console
virtualenv virtualenv
 ```
 
 <br>From your directory (in this case twitterApp), activate your **virtualenv** virtual environment
```console
source virtualenv/bin/activate
 ```
 <br>Insall the **tweepy** module
 ```console
pip install pyjokes
 ```
 <br> Stop the **virtualenv**
  ```console
deactivate
 ```
 <br> Copy the following code and save it into a file called **__main__.py** in the twitterApp directory, and add the corresponding credentials that we got from step 1 (Customer keys) and step 2 (COS credentials)
 ```python
import tweepy
import sys, json
import pandas as pd
import csv
import os
import types
from botocore.client import Config
import ibm_boto3

# Twitter API credentials
consumer_key = <"YOUR_CONSUMER_API_KEY">
consumer_secret = <"YOUR_CONSUMER_API_SECRET_KEY">
screen_name = "@CharlizeAfrica"  #you can put your twitter username, here we are using Charlize Theron username

def main(dict):
    tweets = get_all_tweets()
    createFile(tweets)

    return {"message": 'success' }


def get_all_tweets():
    # initialize tweepy
    auth = tweepy.AppAuthHandler(consumer_key, consumer_secret)
    api = tweepy.API(auth)

    alltweets = []
    for status in tweepy.Cursor(api.user_timeline, screen_name = screen_name).items(3200):
        alltweets.append(status)

    return alltweets

def createFile(tweets):
    outtweets=[]
    for tweet in tweets:
        outtweets.append([tweet.created_at.hour,
                          tweet.text, tweet.retweet_count,
                          tweet.favorite_count])

    client = ibm_boto3.client(service_name='s3',
    ibm_api_key_id=<"COS_API_KEY">,
    ibm_service_instance_id= <"COS_SERVICE_ID">,
  
    config=Config(signature_version='oauth'),
    endpoint_url=<"COS_ENDPOINT_URL">)
    
    body = client.get_object(Bucket=<'BUCKET_NAME'>,Key='tweets.csv')['Body']
    if not hasattr(body, "__iter__"): body.__iter__ = types.MethodType( __iter__, body )
    
    cols=['hour','text','retweets','favorites']
    table=pd.DataFrame(columns= cols)  
    for i in outtweets:
        table=table.append({'hour':i[0], 'text':i[1], 'retweets': i[2], 'favorites': i[3]}, ignore_index=True)
        
    table.to_csv('tweets2.csv', index=False)
    try:
        res=client.upload_file(Filename="tweets2.csv", Bucket=<'BUCKET_NAME'>,Key='tweets.csv')
    except Exception as e:
        print(Exception, e)
    else:
        print('File Uploaded') 
```    
<br> From the twitterApp directory, create a .zip archive of the virtualenv folder and your **main.py** file. These files must be in the top level of your .zip file.
```console
zip -r jokes.zip virtualenv main.py
```

<br> Now it's time to push this function to IBM Cloud. Create an action called twitterAction using the zip folder that was just created


<br>Now that we've got our Twitter API keys and secrets, we can use [tweepy](https://github.com/tweepy/tweepy) to save tweets into a CSV file. Free developer accounts on Twitter will limit the amount of tweets that are retrieved, but that's enough for our purposes.

If you don't have Python, then [download and install the latest version](https://www.python.org/downloads/), and then install [tweepy](https://github.com/tweepy/tweepy). This can be done using `pip install tweepy`, if you have [pip](https://pip.pypa.io/en/stable/quickstart/) installed.

Copy the code below into a new file and save it. There are a few lines to update at the top, add values to the variables for keys, secrets, and the twitter handle you want to analyze.

```python
import csv
import tweepy

# Twitter API credentials
consumer_key = ""
consumer_secret = ""
access_key = ""
access_secret = ""
screen_name = ""


def get_all_tweets():
    # initialize tweepy
    auth = tweepy.OAuthHandler(consumer_key, consumer_secret)
    auth.set_access_token(access_key, access_secret)
    api = tweepy.API(auth)

    alltweets = []

    # request first 200 tweets, the max allowed
    new_tweets = api.user_timeline(screen_name=screen_name, count=200)
    alltweets.extend(new_tweets)
    oldest = alltweets[-1].id - 1

    # keep grabbing tweets until the 3200 tweet limit is hit
    while len(new_tweets) > 0:
        print("getting tweets before id: %s" % (oldest))
        new_tweets = api.user_timeline(screen_name=screen_name,
                                       count=200,
                                       max_id=oldest)
        alltweets.extend(new_tweets)
        oldest = alltweets[-1].id - 1
        print("...%s tweets downloaded so far" % (len(alltweets)))

    return alltweets


def write_tweets_to_csv(tweets):
    # transform the tweepy tweets into an array
    outtweets = [[tweet.id_str, tweet.created_at,
                  tweet.text.encode("utf-8"), tweet.retweet_count,
                  tweet.favorite_count] for tweet in tweets]

    # write the csv
    with open('%s_tweets.csv' % screen_name, 'w') as f:
        writer = csv.writer(f)
        writer.writerow(["id", "created_at", "text", "Retweets", "Favorites"])
        writer.writerows(outtweets)

    pass


if __name__ == '__main__':
    # pass in the username of the account you want to download
    tweets = get_all_tweets()
    write_tweets_to_csv(tweets)
```

Run the script by running `python tweets.py` in a terminal, a CSV file will be output, containing various tweets and information about those tweets, for example:

![](images/csv1.png)

You can remove the `id` and `created_at` columns, and remove empty rows to clean the data a bit.

![](images/csv2.png)

### Step 3: Log into Watson Studio

IBM Watson Studio is an easy-to-use, collaborative and cloud based environment for data scientists where they can use tools like Scala, R, Jupyter Notebookc etc.

Log into [https://dataplatform.cloud.ibm.com/](https://dataplatform.cloud.ibm.com/?cm_sp=ibmdev-_-developer-tutorials-_-cloudreg) and choose to create a `New Project`, the `Complete` option will work for this tutorial.

![](images/new_project.png)

At the new project wizard, enter a `Name` and `Description`, You will also be required to create a new `Object Storage` service or choose an existing service during project creation. Once created, you'll be able to see a project overview, for example:

![](images/dashboard.png)

Once created, we can add an asset, by clicking `Add to project` and in this case, we'll click `Model`, to add a new model.

![](images/add_model.png)

### Step 4: Create a new model

Give your model a `Name` and `Description`. We will also set the `Model type` option to `Model builder` and choose the `Manual` for this exercise.

Before proceeding we need to associate two services. An `Apache Spark` service, and a `Machine Learning` service. You can use the UI to create a new one or select an existing one. For an example of how to do that with `Apache Spark`, refer to this [IBM Code Tutorial](https://developer.ibm.com/tutorials/create-a-spark-service-for-ibm-watson-studio/). To do that with `Machine Learning` is the same exercise.

![](images/create_model.png)

### Step 5: Add data to the model

We're now going to add the CSV file to the model. Click `Add Data Assets`, browse to either the generated CSV file or the saved sample CSV file. The data should appear in the dashboard, for example:

![](images/add_tweets.png)

Click on the `Next` button to continue. Loading the data may take a few minutes.

### Step 6: Select a training technique

For this example we're trying to predict the best time to send a tweet, so let's set the `Column value to predict` to be `hour`. Leave the `Feature columns` unchanged and set to `All`. The important choice here is the **technique** used, we'll be using the `Regression` technique. We'll also be leving the `Validation Split` unchanged.

*It should be noted that because the classifier is set to `hour`, which has around 20 values, Watson Studio will suggested `Multiclass classification`. But in this case the best technique according to our data is `Regression`.*

![](images/technique.png)

We also need to add estimators. To do that, click on `Add Estimators` and select all avilable choices, then click `Add`.

![](images/estimators.png)

Once we have our technique and estimators selected we can click `Next`. This will start training and testing data. This step will take a few minutes to fully complete.

### Step 7: Wrapping up

The results show just how accurate each estimator is, with the most optimal estimator at the top. Here it is `Isotonic Regression`, click on the first one and select the `Save` option, for example:

![](images/evaluation.png)

Once saved, you will be redirected to an overview of the model, for example:

![](images/summary.png)

From here, we can create a web deployment so our model is accessible over a REST call.

![](images/deploy.png)

Congratulations! Your model is saved, deployed, and you can start testing it out with the generated `cURL`, `Java`, `JavaScript` and `Python` snippets.

## Summary

In this tutorial we learned to extract user data from twitter and then perform data science predictive model on it to optimize future tweeting and increasing the users audience. This tutorial of building a model on Watson Studio can be applied on any other CSV file as well and can be further deployed on a web application. We also learned how to deploy the model as a web application to allow REST calls.
