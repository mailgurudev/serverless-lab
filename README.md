# serverless-lab
Hands on lab for a serverless architecture

High Level Design of the Architecture
![Screenshot 2025-02-24 at 10 28 04 PM](https://github.com/user-attachments/assets/3cf8346e-7467-4765-aa72-85f8f138e7a5)

An Amazon API Gateway represents a fully managed service that facilitates the creation, deployment, and management of APIs. Within this architecture, an API Gateway resource is a logical entity that encapsulates a set of methods exposed via an endpoint. In this tutorial, you define a single resource (DynamoDBManager) and configure an HTTP method (POST) on it. This method is integrated with an AWS Lambda function (LambdaFunctionOverHttps) via a Lambda proxy integration, enabling event-driven execution. When a client issues an HTTPS request to the API Gateway endpoint, the request is routed to the associated Lambda function, which processes the event and returns a response to the caller.

The POST method on the DynamoDBManager resource supports the following DynamoDB operations:

•	Create, update, and delete an item.

•	Read an item.

•	Scan an item.

•	Other operations (echo, ping), not related to DynamoDB, that you can use for testing.

The following is a sample request payload for a DynamoDB create item operation:

{
    "operation": "create",
    "tableName": "lambda-apigateway",
    "payload": {
        "Item": {
            "id": "1",
            "name": "Bob"
        }
    }
}

