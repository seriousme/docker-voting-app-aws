Docker Voting App done Serverless Style
=======================================

## Introduction
I was reading [Deploy the Voting App to AWS ECS with Fargate] by Tony Pujals where he describes how to run the [Docker Voting app demo] on AWS using [AWS Fargate]

  [Deploy the Voting App to AWS ECS with Fargate]:https://medium.com/@tonypujals/deploy-the-voting-app-to-aws-ecs-with-fargate-
  [Docker Voting app demo]:https://github.com/subfuzion/docker-voting-app-nodejs
  [AWS Fargate]:https://aws.amazon.com/fargate/

Of course I understand that the Docker Voting app is a showcase of Docker technology and that it's not the most exciting application from a business perspective. I also understand that people want to show how you can take Docker technology to AWS. However in my mind I started wondering: if I would take this to AWS would I be following the same path ? Or would I go Serverless ?

Given the title: I went Serverless!

The voting application is recreated using AWS ApiGateway and AWS DynamoDB only. The queue service could be implemented using SNS, however one typically uses a queue to ensure scalability of the database or to allow for maintenance. Both are handled by AWS DynamoDB automatically.

## Creating the schema
Looking at the sources of the voting app there are two endpoints:
- POST /vote where you can post a vote by posting JSON like `{"vote":"a"}` or `{"vote":"b"}`
- GET /results which will give you results like:
```json
{
  "success" : true,
  "result" : {
    "a" : 0,
    "b" : 0
     }
}
```

Creating a JSON schema produces the following for /vote

```json
{
  "Vote": {
    "type": "object",
    "required": ["vote"],
    "properties": {
      "vote": {
        "type": "string",
        "enum": ["a", "b"]
      }
    },
    "title": "Vote Schema"
  }
}
```
and for /results

```json
{
  "type": "object",
  "properties": {
    "success": {
      "type": "boolean"
    },
    "result": {
      "type": "object",
      "properties": {
        "a": {
          "type": "integer"
        },
        "b": {
          "type": "integer"
        }
      }
    }
  },
  "title": "Results schema"
}
```

Loading these in APIgateway ensures for /vote that only valid POST requests are accepted.

## DynamoDB
The DynamoDB instance with partion key `topic` only holds one record with `topic` value `default` and a numeric value for `a` and `b`

## APIgateway integration
A succesful POST operation must result in a increment of the counter for the subject of the vote in the database.
This is achieved by adding the following integration request mapping template to the POST operation:
```json
{
    "TableName" : "VoteAppDynamoDBTable",
    "Key": {
        "topic": {
            "S":"default"
        }
     },
    "UpdateExpression": "ADD $input.path('$.vote') :inc",
    "ExpressionAttributeValues": {
        ":inc": {
          "N": "1"
        }
    }
}
```
This, together with the rest of the integration configuration, will transform the POST operation into a call to DynamoDB to increment the number of votes for the subject provided.

Then when pulling results the GET operation needs to be transformed into a query on DynamoDB using an integration request mapping template
```json
{
    "TableName" : "VoteAppDynamoDBTable",
    "KeyConditionExpression": "topic = :v1",
    "ExpressionAttributeValues": {
        ":v1": {
            "S": "default"
        }
    }
}
```
and the response needs to be mapped to the schema listed above using an integration response mapping template
```
#set($inputRoot = $input.path('$'))
{
  "success" : true,
  "result" : {
#if($inputRoot.Count==0)
    "a" : 0,
    "b" : 0
#{else}
    "a" : #if($inputRoot.Items[0].a=="")0#{else}$inputRoot.Items[0].a.N#end,
    "b" : #if($inputRoot.Items[0].b=="")0#{else}$inputRoot.Items[0].b.N#end
#end
     }
}
```
As long as no votes have been casted it could be that the whole record does not exist yet or that `a` or `b` is still non-existent. This template takes care of these edge cases.

The full template can be found [here].

  [here]:voteApp-CF-template.json
