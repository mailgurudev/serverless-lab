# serverless-lab
Hands on lab for a serverless architecture

High Level Design of the Architecture
![Screenshot 2025-02-24 at 10 28 04 PM](https://github.com/user-attachments/assets/3cf8346e-7467-4765-aa72-85f8f138e7a5)

An Amazon API Gateway represents a fully managed service that facilitates the creation, deployment, and management of APIs. Within this architecture, an API Gateway resource is a logical entity that encapsulates a set of methods exposed via an endpoint. In this tutorial, you define a single resource (DynamoDBManager) and configure an HTTP method (POST) on it. This method is integrated with an AWS Lambda function (LambdaFunctionOverHttps) via a Lambda proxy integration, enabling event-driven execution. When a client issues an HTTPS request to the API Gateway endpoint, the request is routed to the associated Lambda function, which processes the event and returns a response to the caller.

he POST method on the DynamoDBManager resource supports the following DynamoDB operations:

* Create, update, and delete an item.
* Read an item.
* Scan an item.
* Other operations (echo, ping), not related to DynamoDB, that you can use for testing.

The request payload you send in the POST request identifies the DynamoDB operation and provides necessary data. For example:

The following is a sample request payload for a DynamoDB create item operation:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Guru"
        }
    }
}
```
Below is a sample request payload for a DynamoDB read item operation:
```json
{
    "operation": "read",
    "tableName": "lambda-apigateway",
    "payload": {
        "Key": {
            "id": "1"
        }
    }
}
```
## Steps for setting up the Architecture

### Step 1 - Create an IAM Role for Lambda

Create an execution role that gives your function permission to access AWS resources.

To create an execution role
1. Go to Policies Page in IAM Console
2. Create Policy with below properties
   * Policy Editor - JSON
   * Paste the below code for sutom policy - Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that  the function needs to write data to DynamoDB and upload logs.  
   * Policy Name - custompolicy   

```json
    {
    "Version": "2012-10-17",
    "Statement": [
    {
      "Sid": "Stmt1428341300017",
      "Action": [
        "dynamodb:DeleteItem",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan",
        "dynamodb:UpdateItem"
      ],
      "Effect": "Allow",
      "Resource": "*"
    },
    {
      "Sid": "",
      "Resource": "*",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Effect": "Allow"
    }
    ]
    }

```

3. Open the roles page in the IAM console.
4. Choose Create role.
5. Create a role with the following properties.
    * Trusted entity – AWS service.
    * service or use case - lambda
    * Role name – **lambda-apigateway-role**.
    * Permissions – Custom policy with permission to DynamoDB and CloudWatch Logs. This custom policy has the permissions that  the function needs to write data to DynamoDB and upload logs.    

### Create Lambda Function
** To create the function**
![Create function](./images/create_lambda_function.png)
1. Click "Create function" in AWS Lambda Console
2. Select "Author from scratch". Use name **LambdaFunctionOverHttps** , select **Python 3.7** as Runtime. 
Under Permissions, expand Change Default execution role select "Use an existing role", and select **lambda-apigateway-role** that we created, from the drop down
3. Click "Create function"
4. Under code section replace teh default code with the following code snippet and click "Save"

** Python Code**

```python
from __future__ import print_function

import boto3
import json

print('Loading function')


def lambda_handler(event, context):
    '''Provide an event that contains the following keys:

      - operation: one of the operations in the operations dict below
      - tableName: required for operations that interact with DynamoDB
      - payload: a parameter to pass to the operation being performed
    '''
    #print("Received event: " + json.dumps(event, indent=2))

    operation = event['operation']

    if 'tableName' in event:
        dynamo = boto3.resource('dynamodb').Table(event['tableName'])

    operations = {
        'create': lambda x: dynamo.put_item(**x),
        'read': lambda x: dynamo.get_item(**x),
        'update': lambda x: dynamo.update_item(**x),
        'delete': lambda x: dynamo.delete_item(**x),
        'list': lambda x: dynamo.scan(**x),
        'echo': lambda x: x,
        'ping': lambda x: 'pong'
    }

    if operation in operations:
        return operations[operation](event.get('payload'))
    else:
        raise ValueError('Unrecognized operation "{}"'.format(operation))
```
![Create function](./images/Lambda_function_over_https.png)

### Test Lambda Function

Let's test our newly created function. We haven't created DynamoDB and the API yet, so we'll do a sample echo operation. The function should output whatever input we pass.

1. Click the arrow on "Select a test event" and click "Configure test events"
![Create function](./images/test_lambda_function.png)

3. Paste the following JSON into the event. The field "operation" dictates what the lambda function will perform. In this case, it'd simply return the payload from input event as output. Click "Create" to save
```json
{
    "operation": "echo",
    "payload": {
        "somekey1": "somevalue1",
        "somekey2": "somevalue2"
    }
}
```

3. Click "Test", and it will execute the test event. You should see the output in the console
![Create function](./images/test_event_output.png)

We are good to create DynamoDB table and an API using our lambda as backend!

5. Deploy the Lambda function

### Create DynamoDB Table

Create the DynamoDB table that the Lambda function uses.

**To create a DynamoDB table**

1. Open the DynamoDB console.
2. Choose Create table.
3. Create a table with the following settings.
   * Table name – lambda-apigateway
   * Partition key – id (string)
   * Table settings - Default settings
4. Choose Create.

### Create API

**To create the API**
1. Go to API Gateway console
2. Click Create API


3. Under choose an API Type Scroll down and select  Build for "REST API"

4. Give the API name as "DynamoDBOperations", keep everything as is, click "Create API"

5. Each API is collection of resources and methods that are integrated with backend HTTP endpoints, Lambda functions, or other AWS services. 
   Typically, API resources are organized in a resource tree according to the application logic. At this time you only have the root resource, but let's add a resource next 

Click "Actions", then click "Create Resource"

6. Input "DynamoDBManager" in the Resource Name, Resource Path will get populated. Click "Create Resource"

7. Let's create a POST Method for our API. With the "/dynamodbmanager" resource selected, Click "Actions" again and click "Create Method". 

8. Select "POST" from drop down , then click checkmark

9. The integration will come up automatically with "Lambda Function" option selected. Select "LambdaFunctionOverHttps" function that we created earlier. As you start typing the name, your function name will show up.Select and click "Save". A popup window will come up to add resource policy to the lambda to be invoked by this API. Click "Ok"

API-Lambda integration is done!

### Deploy the API

In this step, you deploy the API that you created to a stage called prod.

1. Click "Actions", select "Deploy API"

2. Now it is going to ask you about a stage. Select "[New Stage]" for "Deployment stage". Give "Prod" as "Stage name". Click "Deploy"

3. To invoke our API endpoint, we need the endpoint url. In the "Stages" screen, expand the stage "Prod", select "POST" method, and copy the "Invoke URL" from screen

### Running our solution

1. The Lambda function supports using the create operation to create an item in your DynamoDB table. To request this operation, use the following JSON:

```json
{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1234ABCD",
            "number": 5
        }
    }
}
```
2. To execute our API from local machine, we are going to use Postman and Curl command. You can choose either method based on your convenience and familiarity. 
    * To run this from Postman, select "POST" , paste the API invoke url. Then under "Body" select "raw" and paste the above JSON. Click "Send". API should execute and return "HTTPStatusCode" 200.
      
    * To run this from terminal using Curl, run the below
    ```
    $ curl -X POST -d "{\"operation\":\"create\",\"tableName\":\"lambda-apigateway\",\"payload\":{\"Item\":{\"id\":\"1\",\"name\":\"Bob\"}}}" https://$API.execute-api.$REGION.amazonaws.com/prod/DynamoDBManager
    ```   
3. To validate that the item is indeed inserted into DynamoDB table, go to Dynamo console, select "lambda-apigateway" table, select "Items" tab, and the newly inserted item should be displayed.

4. To get all the inserted items from the table, we can use the "list" operation of Lambda using the same API. Pass the following JSON to the API, and it will return all the items from the Dynamo table

```json
{
    "operation": "list",
    "tableName": "lambda-apigateway",
    "payload": {
    }
}
```

We have successfully created a serverless API using API Gateway, Lambda, and DynamoDB!

## Destroy the Setup

Let's destroy the resources we have created for this lab.


### Deleting DynamoDB

To delete the table, from DynamoDB console, select the table "lambda-apigateway", and click "Delete table"


### Deleting Lambda

To delete the Lambda, from the Lambda console, select lambda "LambdaFunctionOverHttps", click "Actions", then click Delete 


### Deleting API

To delete the API we created, in API gateway console, under APIs, select "DynamoDBOperations" API, click "Actions", then "Delete"

