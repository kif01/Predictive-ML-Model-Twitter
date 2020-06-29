# Predictive-ML-Model-Twitter

<!-- markdownlint-disable MD026 -->

In the era that we currently live in, all the focus has shifted towards data. Each day, the amount of data that is generated and consumed is increasing, adding somewhere around 5 exabytes of data. Everything we do generates data, be it turning on and off the light, or commuting from home to work. This data can be used to generate information that can be used for insights to predict and extract patterns. Data Mining or Data Science is the term that has taken the industry abuzz. It is the process of discovering patterns, insights, and associations from data. In this how-to guide we’ll learn how to use data and implement a predictive model on it to get insights. Our intended audience include developers, general users with basic knowledge of programming, and organizations that want to enhance customer experience. It will enable a user to create a predictive model using AutoAI on Watson Studio, which is a cloud-based environment for Data Scientists. By using this how-to user can predict and optimize their twitter interaction and would lead to optimum traffic on their tweets.


## Learning objectives
This is an end-to-end tutorial that shows how we can extract the data, create a csv file and upload it to Cloud Object Storage (COS), create a data connection from Watson Studio to COS, and finally to refine the data and use it to build deploy and test our predictive model with AutoAI.<br><br> 
After completing this how-to, the reader will be able to:
* Work with Cloud Functions to extract data from Twitter 
* Learn to create upload a CSV file in COS from a Cloud Function.
* Learn how to use Watson Studio and AutoAI to build a predictive model using CSV data.
* Leverage Twitter to predict and optimize their twitter interactions.

## Prerequisites

* IBM Cloud account - [Sign up](https://cloud.ibm.com/registration?cm_sp=ibmdev-_-developer-tutorials-_-cloudreg/) if you don't have an account yet.
* A [Twitter account](https://twitter.com/)
* A [Twitter Developer account](https://developer.twitter.com/)

## Estimated time

To complete this tutorial it should take around 1 hour.

## Steps

### Use sample data or get your own?

The first thing we'll need to do is get a bunch of tweets to analyze. In this step we'll go through how to get a bunch of tweets, but if you're not interested in doing that, we provide a sample data set (If you choose to use the sample data set, then you will be skipping the **Twitter API access** and **Cloud Function** sections of this tutorial):

* **[ufone_tweets.csv](static/ufone_tweets.csv)**: Tweets from a Ufone, a phone operator, cleaned up and ready to use.


### Step 1: Getting Twitter API access

*If you're using the sample data, then skip to Step 2.*

Before we use [tweepy](https://github.com/tweepy/tweepy) to get tweets, we need to generate our **Consumer** api keys :<br/>
Go to your [Twitter Developer account](https://developer.twitter.com/), hover over your name on the top right create your app. Fill the required information.

<img width="1440" alt="twitter1" src="https://user-images.githubusercontent.com/15332386/85959798-8e894e80-b9af-11ea-8dc2-ef78b689b614.png">

Once your app is created select the `Keys and Tokens` tab. You will see your `Consumer API key` and `Consumer API secret key` which we will be using later in the tutorial. These can be revoked and regenerated, but as with any other key, you should keep these secret. (Here we won't be using the api tokens so you can ignore them)

<img width="1440" alt="twitter2" src="https://user-images.githubusercontent.com/15332386/85959799-921cd580-b9af-11ea-9ff3-8b16fa530ff9.png">

### Step 2: Creating a Cloud Object Storage (COS) service in IBM Cloud

Log into your ibm cloud account at https://cloud.ibm.com/login, click on `Create Resource` and search for **Object Storage**.

<img width="1440" alt="cos1" src="https://user-images.githubusercontent.com/15332386/85959761-5aae2900-b9af-11ea-959b-8441e5fdbf71.png">

Choose the `Lite` plan which is free, change the name if you want to and click on `Create`.

<img width="1440" alt="cos2" src="https://user-images.githubusercontent.com/15332386/85959763-5f72dd00-b9af-11ea-801f-c7334573053b.png">

You can now find your object storage instance created in resources under `Storage`. Once you open your instance, click on `Buckets` from the left side panel and create a bucket (You can choose any type of bucket). Make sure to note down the name of your bucket once you create it.

<img width="1440" alt="cos3" src="https://user-images.githubusercontent.com/15332386/85959764-600b7380-b9af-11ea-8b51-40e762b398bb.png">

Go to `Service Crendentials` from the panel, select the service credential that has just been created. If nothing is there then click on `New credential` to generate one. Click on the arrow to expand the credentials. Note down the `api_key`, `iam_serviceid_crn` and `resource_instance_id` .

<img width="1440" alt="cos4" src="https://user-images.githubusercontent.com/15332386/85959765-60a40a00-b9af-11ea-824d-c44c07d62e7c.png">


Go to `Endpoint` from the panel. Choose your `Resilency` and `Region`, and note down the `Private url` since it will be needed later in our cloud function.<br/>

<img width="1440" alt="cos5" src="https://user-images.githubusercontent.com/15332386/85959766-613ca080-b9af-11ea-990f-d2aef0d6c83c.png">

Our bucket is now ready, make sure to have your:
 * Bucket name
 * Api Key
 * Service ID
 * Resource Instance ID
 * Endpoint Url
 
*Again, if you're using the sample data, then you can directly upload the file in your bucket, and skip step 3 (Jump to step 4).*
 
 
### Step 3: Create a Cloud Function ( This step is only valid if you started with step 1 )
Cloud Function is IBM's Function-as-a-Service (Faas) programming platform where you write simple, single-purpose functions known as **Actions** that can be attached to **Triggers** which trigger the function when a specific defined event occurs.

#### Creating an Action

Usually we create the **Actions** directly from IBM cloud, but in our case, we want to use [tweepy](https://github.com/tweepy/tweepy) which is an external Python library for accessing Twitter API. External libraries are not supported in the Cloud Function runtime environment. We will have to write our Python code and package it with a virual local environemnt in a .zip file, and then push it to IBM Cloud.<br><br>

If you don't have Python, then [download and install the latest version](https://www.python.org/downloads/). Once installed make sure to install `virtualenv`. 
```dos
$ pip install virtualenv 
``` 
Create a directory that you can use to create your virtual environment. In this example I named it `twitterApp`.
```dos
$ cd desktop; mkdir twitterApp; cd twitterApp
```

<br>From the `twitterApp` directory, create virtual environment named `virtualenv`. Your virtual environemt must be named `virtualenv`.
```dos
$ virtualenv virtualenv
 ```
 
 <br>From your directory (in this case `twitterApp`), activate your `virtualenv` virtual environment.
```dos
$ source virtualenv/bin/activate
 ```
 <br>Insall the `tweepy` module.
 ```dos
(virtualenv) $ pip install tweepy
 ```
Stop the `virtualenv.`
  ```dos
(virtualenv) $ deactivate
 ```
Copy the following code and save it into a file called `main.py` in the `twitterApp` directory, and add the corresponding credentials that we got from step 1 (**Customer keys**) and step 2 (**COS credentials**). In addition, you can change the twitter handle that you want to analyze (In this example we are using **Charlize Theron**'s twitter handle to analyze).<br>
This code gets the data from twitter and then creates a CSV file that contains this data and upload it into the object storage service that we created at the beginning. Once we run this function, a CSV file containig tweets info will be uploaded in COS.
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
screen_name = "@CharlizeAfrica"  #you can put your twitter username, here we are using Charlize Theron twitter profile to analyze.

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
    endpoint_url= "https://" + <"COS_ENDPOINT_URL">)
    
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
From the `twitterApp` directory, create a .zip archive of the `virtualenv` folder and your `main.py` file. These files must be in the top level of your .zip file.
```dos
$ zip -r twitterApp.zip virtualenv main.py
```
Now it's time to push this function to IBM Cloud Log in to your IBM Cloud account and make sure to target your organization and space. You can check more about this here https://cloud.ibm.com/docs/cli?topic=cli-ibmcloud_cli#ibmcloud_target .
```doc
$ ibmcloud login
```

Create an action called `twitterAction` using the zip folder that was just created (right click on the file and check get info for Mac or Properties for Windows to get the path), by specifying the entry point which is our`main` function in the code, and the `--kind` flag for runtime.
```dos
$ ibmcloud fn action create twitterAction </path/to/file/>twitterApp.zip --kind python:3.7 --main main
```
Go back to IBM Cloud, and click on Cloud Functions on the left side of the window. 

<img width="1438" alt="CF1" src="https://user-images.githubusercontent.com/15332386/86049106-a077f980-ba62-11ea-8fc5-1cf4057dc6dc.png">

Click on `Action`, make sure the right namespace is selected, you will see the action that was created. Click on it and then click `Invoke` to run it.

<img width="1432" alt="CF2" src="https://user-images.githubusercontent.com/15332386/86049113-a4a41700-ba62-11ea-8f9f-9a3cd4e9bc92.png">

You can run it as well directly from the terminal using this command:
```dos
$ ibmcloud fn action invoke twitterAction --result
```
If you go to your bucket in the object storage service that you created at the beginning of the tutorial, you will see a file `tweets.csv` that has just been uploaded. This is the file that has all the extracted tweets from the Cloud Function.

#### Creating a Trigger

Let's create a **Trigger** that invokes our action. Choose `Triggers` from the left panel and click on `Trigger`

<img width="1440" alt="Trigger1" src="https://user-images.githubusercontent.com/15332386/86051297-3e20f800-ba66-11ea-9ab9-154caae76c50.png">

Choose `Periodic` for the trigger type. This means that our event here is time, the function will get invoked on a specific time.

<img width="1439" alt="Trigger2" src="https://user-images.githubusercontent.com/15332386/86051309-41b47f00-ba66-11ea-955b-2edbf6d9f394.png">

Give a name to your trigger and define set the timer and click `Create`. In this example, the timer is set on Sundays. Every Sunday the trigger will fire at 4:00 am GMT+4 and invokes our action to fetch twitter data and create a new csv file with new tweets.

<img width="1440" alt="Trigger3" src="https://user-images.githubusercontent.com/15332386/86051310-42e5ac00-ba66-11ea-8fea-1ec6c6f884f8.png">

Click `Add` so we can connect this trigger to our action.

<img width="1440" alt="Trigger4" src="https://user-images.githubusercontent.com/15332386/86051312-437e4280-ba66-11ea-8ff3-064eae2de16b.png">

Choose the `Select Existing` tab, select your action and click `Add`. Now your action is connected to this trigger and get fired based on the time that you specified before.

<img width="1440" alt="Trigger5" src="https://user-images.githubusercontent.com/15332386/86051314-437e4280-ba66-11ea-9940-ba7c10f849d4.png">


### Step 4: Create a Watson Studio Service
 
Just like we created the COS at the beginning, we will repeat the same process but this time we will create a Watson Studio service. Search for **Watson Studio** select the `Lite plan` to create it. You can find it instantiated under services in resource summary (Main dashboard of your ibm cloud account). Click on it and the click on `Get Started`. This will launch the Watson Studio platform.
 
<img width="1440" alt="WS1" src="https://user-images.githubusercontent.com/15332386/86030143-25ecb100-ba45-11ea-8424-fa34cf01dec8.png">
 
Click on `Create Project` and then `Create Empty Project`.

<img width="1440" alt="WS2" src="https://user-images.githubusercontent.com/15332386/85960269-64399000-b9b3-11ea-8d77-7277185a12ac.png">

Give a name your project and give it a description. Make sure to choose the COS that you created before<br>

<img width="1440" alt="WS3" src="https://user-images.githubusercontent.com/15332386/86039582-add9b780-ba53-11ea-920b-6c813218916a.png">


### Step 5: Create a connection to the COS
 
Click on `Add to projects`. Here you will see all kind of assets that we can use in Watson Studio. We want to create a connection to our COS so we can access the `tweets.csv` file. Once we can reach this file, we can then have access to the data inside it that we need to build our machine learning model with AutoAI.

<img width="1440" alt="WS4" src="https://user-images.githubusercontent.com/15332386/86039587-b0d4a800-ba53-11ea-9ef2-2868794d1a23.png">

Click on `connection` so we can start creating our connection to our COS 

<img width="961" alt="WS5" src="https://user-images.githubusercontent.com/15332386/85960273-6996da80-b9b3-11ea-9777-433d8189d512.png">

Click on `Cloud Object Storage`.
<img width="1440" alt="WS6" src="https://user-images.githubusercontent.com/15332386/85960274-6996da80-b9b3-11ea-82d7-abe832f6118b.png">

Add a name to your connection, and fill the information with the credentials that we got from **step 2** (COS credentials).
<img width="1440" alt="WS7" src="https://user-images.githubusercontent.com/15332386/85960276-6a2f7100-b9b3-11ea-9dc4-c07822487f14.png">

Click again on `Add to projects` and this time click on `connected data`. Select your source which is the connection created in the previous step, select your bucket and then choose `tweets.csv` file. Give a name to your asset and click on `Create`. <br>
<img width="1440" alt="WS8" src="https://user-images.githubusercontent.com/15332386/85960277-6ac80780-b9b3-11ea-9e06-2f787714f522.png">

### Step 6: Refine the Data
<img width="1440" alt="WS9" src="https://user-images.githubusercontent.com/15332386/85960278-6ac80780-b9b3-11ea-8285-39ebee795635.png">

Our data is already prepared  but we just need to convert the rows hour, favorites and retweets to integer. Let's start with hour: Click on the 3 dots, `Convert column` and then choose `integer`. Repeat the same process for favorites and retweets. <br>
<img width="1104" alt="WS10" src="https://user-images.githubusercontent.com/15332386/85960279-6b609e00-b9b3-11ea-8653-d95bfb37e31c.png">

Once you're done, click on `save and create job` <br>
<img width="1433" alt="WS11" src="https://user-images.githubusercontent.com/15332386/85960280-6b609e00-b9b3-11ea-842d-d0fc739ce5be.png">

Give the job a name, and click on `Create and Run`<br>
<img width="1438" alt="WS12" src="https://user-images.githubusercontent.com/15332386/85960281-6bf93480-b9b3-11ea-9906-356b13ff72c7.png">

This job will create a new data set based on the one that we already have but with our refinements that were responsible to convert 3 rows to integer. As we can see, the output of this job is a file is named `Tweets_shaped.csv`. Wait unitl the status of the job shows **Completed**. <br>
<img width="1440" alt="WS13" src="https://user-images.githubusercontent.com/15332386/85960282-6c91cb00-b9b3-11ea-9f93-d7fce0df6b93.png">

Now you should see 3 assets just like the image below. The `Tweets_shaped.csv` is now our main file that we will be using in AutoAI to create our predictive model. <br>
<img width="1440" alt="WS14" src="https://user-images.githubusercontent.com/15332386/85960283-6c91cb00-b9b3-11ea-8f31-2cfef8871480.png">

### Step 7: Create an AutoAI experiment

Click again on `Add to projects` and and this time choose `AutoAI experiment`.

<img width="850" alt="AI1" src="https://user-images.githubusercontent.com/15332386/86002816-f4162300-ba21-11ea-82cc-3434b458df24.png">

Give a name to your project and choose a machine learning instance. This is needed so we can deploy our model at the end. If you don't have one, Watson Studio will ask you to directly create it and you will be bale to proceed normally.

<img width="1440" alt="AI2" src="https://user-images.githubusercontent.com/15332386/86002824-f8424080-ba21-11ea-82b0-766288acce00.png">

Now you need to add your file, select the `Tweets_shaped.csv` file that was generated from the Data Refinery.

<img width="788" alt="AI3" src="https://user-images.githubusercontent.com/15332386/86002828-f8dad700-ba21-11ea-9045-538b3d631c1f.png">

Here we want to predict the best time to share our tweets, so choose **hour** as the prediction column. You will see that the prediction type is Regression and that's because we want to predict a continous value, and the optimized metric is RMSE (Root Mean Squared Error). You can change and customize your experiment if you want by clicking on `Experimenty Setting` 

<img width="1423" alt="AI4" src="https://user-images.githubusercontent.com/15332386/86002830-f9736d80-ba21-11ea-815a-bf976481e529.png">

In the Experiment settings, go to prediction. Here you can see all the algorithms that we can use in our experiment. You can change the number of algortithms to use. For example you can choose 3, which means that the experiment will use the top 3 algorithms for our use case. For every algorithms, AutoAI generates 4 pipelines. In other words, the first pipeline is the regular one with no enhancement added, the second one is with HPO (Hyperparameter Optimization), the third one is with HPO and Feature Engineering and the last one is with HPO, Feature Engineering and another HPO. Since here we are using 3 algortithms, we will have a total of 12 pipelines (3x4=12), so AutoAI will build and generate 12 candidates to find our best model.

<img width="1437" alt="AI5" src="https://user-images.githubusercontent.com/15332386/86002832-fa0c0400-ba21-11ea-9d3d-06fead0acfb2.png">

### Step 8: Build and Evaluate the models

AutuAI will be generating our 12 best models for our use case. There are different ways to understand and visualize the results. Here we are looking at a **Relationship Map** which shows how AutoAI is building and generating the pipelines. Every color represent a type of algorithms and each one has its 4 pipelines that we discussed about in the previous step.

<img width="1439" alt="AI6" src="https://user-images.githubusercontent.com/15332386/86002836-faa49a80-ba21-11ea-84f1-4c33fd4b7a30.png">

You can click on `swipe view` to check the **Progress Map** which is another way to visualize how AutoAI generated our pipelines in a sequence way.

<img width="1440" alt="AI7" src="https://user-images.githubusercontent.com/15332386/86002838-faa49a80-ba21-11ea-9d77-c08c6c56d014.png">

You can see the **Pipeline Leaderboard** to check which model is the best. In our case, **Pipeline 12** is the best model using Random Forest Regressor will all three enhancements (First HPO, Feature Engineering and the second HPO).

<img width="1440" alt="AI8" src="https://user-images.githubusercontent.com/15332386/86002839-fb3d3100-ba21-11ea-8ed2-06f25245da53.png">

AutoAI shows you the comparison between all these pipelines. If you click **Pipeline Comparison** you will see a metric chart comparing our candidates.

<img width="1440" alt="AI9" src="https://user-images.githubusercontent.com/15332386/86002840-fbd5c780-ba21-11ea-86d3-c983fa969b94.png">

Click on Pipeline 12 since it's our best model so can get a better understanding of it. For example you can check its **Feature importance** that shows the features that are keys in making deciosion for our predictive model. In this example, **retweets** is the most important factor for the prediction. We can see new featured generated like **NewFeature_3** and **NewFeature_0**. These are combinations of different features (for example a combination of retweets and favorites) that are generated with feature engineering to enhance the model.

<img width="1106" alt="AI10" src="https://user-images.githubusercontent.com/15332386/86002841-fbd5c780-ba21-11ea-9c11-c9e196ef0e00.png">

### Step 9: Save and Deploy the Model

Let's save and deploy our model so we can start using it. Click on `Save as` and choose `Model`. This will save our model and you can now access it from the main dashboard of your project in `Assets` under `Models` section.

<img width="1440" alt="AI11" src="https://user-images.githubusercontent.com/15332386/86002842-fc6e5e00-ba21-11ea-9e0e-2535cb0a011b.png">

Click on this new created model, select the `Deployments Tab` and click `Add Deployment` to create our deployment where you must give it a name. This is a web deployment that can be accessible using a REST call.

<img width="1440" alt="AI12" src="https://user-images.githubusercontent.com/15332386/86002847-fd06f480-ba21-11ea-937a-245ee7aa3b77.png">

Wait till the status of the deployment is **Ready** in the `Deployment` tab. Once it's ready, click on the deypoyment's name.

<img width="1422" alt="AI13" src="https://user-images.githubusercontent.com/15332386/86013620-655cd280-ba30-11ea-93d4-b53605f1e803.png">

### Step 10: Test the model

Our model is now ready, so we can start using it. Select the `Test` tab and fill the fields with some data. You can put the data in a JSON format if you prefer (this is easier in cases where we have a lot of fields, but here we have only 3 fields). Click `Predict` and you will see the result in `values`. In this example we have the value of 14.5 which is 2:30 pm. This means the best time to share a tweet that can get around 7000 retweets and 2000 favorites is 2:30 pm for Charlize Theron (Remember here we are using Charlize Theron's data (as we saw in **step 3**) so this time is suitable for Charlize. You can put your own user name in the cloud function if you want to predict the best time for your account).<br> 
If you want to inplmenent this model in your application check the `Implmentation` . It shows the endpoint url and code snippets for different programming languages (`cURL`, `Java`, `JavaScript`, `Python` and `Scala` snippets) that you can use for your application.

<img width="872" alt="AI14" src="https://user-images.githubusercontent.com/15332386/86014206-13687c80-ba31-11ea-81e7-95e61de33c1c.png">


## Summary

In this tutorial we learned to extract user data from twitter, create a CSV file that contains this data and upload it to COS using Cloud Functions. Then we learned how  perform data science predictive model on this data to optimize future tweeting and increasing the users audience using Watson Studio and AutoAI. This tutorial can be applied on any other CSV file as well and can be further implemented on a web application.
