# Joke Generator

Created using:

 - Lambda
 - API Gateway
 - DynamoDB
 - CloudWatch

# Quick Rundown

Joke Generator is a serverless API that queries a joke site every five minutes and stores the queries in a database. The database is then called using API Gateway to display new jokes to the user. Jokes can be retrieved: randomly, all at once, or through a joke_id search.

![Diagram] (jokegendiatrue.png)

## CloudWatch

A CloudWatch rule is used to incrementally store jokes from an external source. The rule will then make a call to be processed through lambda and finally stored into DynamoDB.  
Using a cron expression, CloudWatch will query the site every five minutes to retrieve and store a new joke in the table via the retrieval function target.

Cron expression

    0/5 * * * ? *

## Lambda

Four lambda functions will be used in this project, one to propogate the DynamoDB table and the rest for different API functions. Lambda as a micro-service will query the joke site and insert the queried joke into the DyanmoDB table. In order for the function to run properly an IAM role featuring two IAM policies was created, the first to access DynamoDB and the other to execute lambda functions. All lambda functions will be given the same IAM role, which also has the ability to write to CloudWatch.

Two Policies:
 - DynamoDBFullAccess
 - LambdaBasicExecutionRole

At the moment the policies are not very secure and I'll need to add a custom policy statement for it.

**Insert Joke Function** is the code necessary to query the site and insert the joke into DynamoDB. This function will have variables applied to it as well, with key/value pairs of table_name and email. This function will later be used in the manually **PUT** joke uploading process to DynamoDB.

**List All Jokes Function** will provide the user with all the jokes in the DyanamoDB table in the return all jokes function. The function will have a key/value pair of table_name.

**Get Random Joke Function** provides the user with a new joke with every page refresh. The function will have the key/pair of table_name

**Get Joke By ID Function** will let users search for a specific joke using DyanamoDB's joke_id. The function will have the key/pair of table_name




## DynamoDB

Setting up the DynamoDB table is very quick and easy compared to the rest of the API process. The table will be broken down as follows:

 - TableName: jokes-api-table
 - primaryKey: joke_id and will be configured as a string

The rest of the setup of the table and input stream will be automated through lambda.



## API Gateway

Create an API from scratch with regional endpoint types chosen. Resources will be used to break down the API into different methods of RESTful endpoints. Stages will then be used to test the API and to allow deployment of the API.

The API Gateway will consist of several endpoints, that each do different things. Each endpoint will trigger it's own custom lambda function:

 - get a list of all jokes
 - get a random joke
 - get a joke by joke_id
 - manually add a joke to the DyanmoDB table

The first resource created in the API will be a simple retrieval of all the jokes. I had named this resource **/jokes**.  A method will then be created within **/jokes** in the form of a **GET** method. This is where the **List All Jokes Function** comes into play, returning all the jokes to the user.

The next resource will be placed at the root of the API and will be to provide a random joke. A  **GET** method wil then be placed within it, with the **Get a Random Joke Function**. If at the root of the API you will simply get a random joke if you make a request to it

With the **/jokes** highlighted, a new child resource will be created to get a joke by specific ID. For this resource, a simple `{joke_id}` will be implemented into the path with the final path of `/jokes/{joke_id}`. The resource name will be joke_id and a new **GET** method will then be added to it. The **Get a Joke By ID Function** will be used in this specific method. In order for it to work correctly, it needs to be told how to handle and map the request. Under **Integration Request** in the highlighted method a **Mapping Template** can be found detailing a request passthrough. Select no template defined and a application/json mapping template with the code below. This code is telling API Gateway how to map the parameter we are passing in.

    {
		"joke_id" : "$input.params('joke_id')"
	}

The final method will allow us to insert jokes manually into DyanmoDB. With **/jokes** highlighted a new **PUT** request will be added and the lambda function **Insert Joke Function**. Since a **PUT** request is now enabled the CloudWatch cron job is not needed anymore and can now be deleted.

The final job will be to deploy the API Gateway and find the invoke URL in order to do a final test of the API. Without a proper site to call upon the different methods, only a random function will be seen as it is in the /root of the API. However if properly configured you would be able to access all functions of the API.
