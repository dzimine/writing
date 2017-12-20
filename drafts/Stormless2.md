# Tutorial: Building a community sign-up app with Serverless, StepFunctions, and StackStorm Exchange - Episode 2


Continue from [episode one][ep-one] to build a serverless app on AWS with [Serverless Framework][sf], AWS Lambda, StepFunctions, DynamoDB, API Gateway, and ready-to-use functions from StackStorm Exchange open-source catalog.

[sf]:(https://serverless.com/framework)

In this short episode, I'll add two more actions: create a contact in ActiveCampaign, and record a user in Dynamo DB. The full code for this episode is on Github at [2-add-more-actions branch](https://github.com/dzimine/slack-signup-serverless-stormless/tree/DZ/2-add-more-actions).

> If you missed [Episode One][ep-one], please go there. It will be the right place to start, whether you are following along, or just skimming over to grasp the idea. 

[ep-one]:(https://medium.com/@dzimine/tutorial-building-a-community-on-boarding-app-with-serverless-stepfunctions-and-stackstorm-b2f7cf2cc419)

### Add more actions

To continue to the next step, you'll need an [ActiveCampaign](https://www.activecampaign.com) account. Fear not: it takes 2 minutes - I just tried - they didn't ask me for a credit card or anything stupid. Just email. Once in, go to "My Settings", and select "Developer". 

<TODO: add screen shot> 

Note the `URL` and `Key` fields in the "API access" section. Add them into `env.yml` like this:

```
# Copy to env.yml and replace with your values.
# Don't commit to Github :)
slack:
  admin_token: "xoxs-111111111111-111111111111-..."
  organization: "your-team"
active_campaign:
  url: "https://YOUR-COMPANY.api-us1.com"
  api_key: "1234567a9012a12345z12aa2..."

```


> PRO TIP: If you're not in the mood for Active Campaign, or any sign-ups, just mock up the API endpoint with something like [Mocky](https://www.mocky.io/) or [mockable.io](https://www.mocky.io/) and adjust the examples accordingly. For bonus points, create another serverless function with API Gateway endpoint in your `serverless.yml` and use it to mock the ActiveCampaign API call. 


Now I am ready to add an [Active Campaign](https://exchange.stackstorm.org/#activecampaign) lambda function. 
Same routine as in [Episode One][ep-one]: start with finding a new action in a pack with `sls stackstorm info --pack activecampaign` (hint: the action you're looking is `contact_sync` which adds and updates a contact). Inspect the action with `sls stackstorm info --action activecampaign.contact_sync`: 

```
$ sls stackstorm info --action activecampaign.contact_sync

activecampaign.contact_sync ... Sync a contact.
Parameters
  api_action [string]  ........ contact_sync
  api_key [string]  ........... Your API key
  api_output [string]  ........ xml, json, or serialize (default is XML)
  email [string] (required) ... Email of the new contact. Example: 'test@example.com'
  field [object]  ............. 'value' (You can also use the personalization tag to specify which field you want updated)
  first_name [string]  ........ First name of the contact. Example: 'FirstName'
  form [string]  .............. Optional subscription Form ID, to inherit those redirection settings. Example: 1001. This will allow you to mimic adding the contact through a subscription form, where you can take advantage of the redirection settings.
  instantresponders [object]  . Use only if status = 1. Whether or not to set "send instant responders." Examples: 1 = yes, 0 = no.
  ip4 [string]  ............... IP address of the contact. Example: '127.0.0.1' If not supplied, it will default to '127.0.0.1'
  last_name [string]  ......... Last name of the contact. Example: 'LastName'
  lastmessage [object]  ....... Whether or not to set "send the last broadcast campaign." Examples: 1 = yes, 0 = no.
  noresponders [object]  ...... Whether or not to set "do not send any future responders." Examples: 1 = yes, 0 = no.
  orgname [string]  ........... Organization name (if doesn't exist, this will create a new organization) - MUST HAVE CRM FEATURE FOR THIS.
  p [object]  ................. Assign to lists. List ID goes in brackets, as well as the value.
  phone [string]  ............. Phone number of the contact. Example: '+1 312 201 0300'
  sdate [object]  ............. Subscribe date for particular list - leave out to use current date/time. Example: '2009-12-07 06:00:00'
  status [object]  ............ The status for each list the contact is added to. Examples: 1 = active, 2 = unsubscribed
  tags [string]  .............. Tags for this contact (comma-separated). Example: "tag1, tag2, etc"
Config
  api_key [string] (required) . ActiveCampaign API Key.
  url [string] (required) ..... ActiveCampaign Account URL
  webhook [object]  ........... Webhook sensor specific settings.
```

Bunch of parameters, but only `email` is required. I also want to use `first_name` and `last_name`. The function definition in `serverless.yml` will look like this: 

```yaml:
  RecordAC:
    timeout: 10
    memorySize: 128
    stackstorm:
      action: activecampaign.contact_add
      input:
        email: "{{ input.body.email }}"
        first_name: "{{ input.body.first_name }}"
        last_name: "{{ input.body.last_name }}"
      config: ${file(env.yml):activecampaign}
```  

I had to bump up the timeout as the ActiveCampaign API is taking longer than Lambda's default 6 seconds during Christmas season. But I can reclaim invocation cost by lowering memory use from default 512Mb. This time I don't bother to expose it to AWS API Gateway - we can conveniently test it with `sls`. 

The function is ready to fly to AWS. We could do it at once with `sls deploy` but let's take it slow again and repeat the deployment steps just like in [episode 1 of this tutorial][ep-one] to engrave the workflow in your mind:

1. Build the bundle:

    ```
    sls package
    ```

2. Test locally (note I'm using `--passthrough` to only test transformations, remove it to make an actual call): 

    ```
    sls  stackstorm docker run --function RecordAC --passthrough \
    --verbose --data \
    '{"body":{"email":"santa@mad.russian.xmas.com", "first_name":"Santa", "last_name": "Claus"}}'
    ```
    
3. Deploy to AWS: 

    ```
    sls deploy
    ```

4. Test on AWS with `sls invoke`:

    ```
    sls invoke --function RecordAC --logs \
    --data '{"body":{"email":"santa@mad.russian.xmas.com", "first_name":"Santa", "last_name": "Claus"}}'
    ```

5. Check the logs

    ```
    sls logs --function RecordAC
    ```


and I checked: Santa Claus appeared in ActiveCampaign's contact list. How do I know it's our Lambda? Because I no longer believe Santa is real enough to subscribe to community without our little Lambda function.

> PRO TIP: If you hit a bug or want a feature in ActiveCampaign or another pack you use from StackStorm Exchange, you can fix it on the spot. Packs are cloned under `./~st2/packs`. Find your action there, modify the code, and use a local run to debug and validate. Big bonus for pushing fixes back to the Exchange: each pack is already a git clone that makes it naturally easy to contribute back to the community.


Let's add the final action, RecordDB, that writes contact info into a DynamoDB table. I could have used the `aws.dynamodb_put_item` action from the [AWS pack on StackStorm Exchange](https://exchange.stackstorm.org/#aws) - the pack is heavily used and well maintained. But I decided to make it a native Lambda: it's just 30 lines of code, with no extra Python dependencies, since the Boto library is already in the AWS Lambda environment. It is also good to see native Lambda functions and StackStorm Exchange actions side by side in `serverless.yml`. 

The code goes into `./record_db/handler.py`:

(TODO: replace code with gist https://gist.github.com/dzimine/279643fac9f99bf503dea6c802881346)

```
# ./record_db/handler.py
import json
import logging
import os

import boto3
dynamodb = boto3.resource('dynamodb')

logger = logging.getLogger()
logger.setLevel(logging.INFO)


def endpoint(event, context):
    logger.info("Event received: {}".format(json.dumps(event)))
    try:
        event = event['body']
        event['email']
    except KeyError:
        raise Exception("Couldn't create the record: `email` not found.")

    table = dynamodb.Table(os.environ['DYNAMODB_TABLE'])

    item = {k: event[k] for k in ['email', 'first_name', 'last_name']}

    table.put_item(Item=item)

    return {
        "statusCode": 200,
        "item": item
    }
```

The serverless.yml with all three actions looks like this:

<script src="https://gist.github.com/dzimine/79545222f4a4aa1c21e0424fa9d375b1.js"></script>


(TODO: replace code with [gist](https://gist.github.com/dzimine/79545222f4a4aa1c21e0424fa9d375b1)

```
service: signup-stormless

provider:
  name: aws
  runtime: python2.7
  timeout: 12
  memorySize: 128
  environment:
    DYNAMODB_TABLE: ${self:service}-${opt:stage, self:provider.stage}
  iamRoleStatements:
    - Effect: Allow
      Action: [dynamodb:Query, dynamodb:Scan, dynamodb:GetItem, dynamodb:PutItem, dynamodb:UpdateItem, dynamodb:DeleteItem]
      Resource: "arn:aws:dynamodb:${opt:region, self:provider.region}:*:table/${self:provider.environment.DYNAMODB_TABLE}"

functions:
  InviteSlack:
    stackstorm:
      action: slack.users.admin.invite
      input:
        email:       "{{ input.body.email }}"
        first_name:  "{{ input.body.first_name }}"
      config:
        admin: ${file(env.yml):slack}
      output:
        statusCode: 200
        body: "{{ output }}"
  RecordAC:
    stackstorm:
      action: activecampaign.contact_add
      input:
        email: "{{ input.body.email }}"
        first_name: "{{ input.body.first_name }}"
        last_name: "{{ input.body.last_name }}"
      config: ${file(env.yml):active_campaign}

  RecordDB:
    handler: record_db/handler.endpoint

resources:
  Resources:
    UsersDynamoDbTable:
      Type: 'AWS::DynamoDB::Table'
      DeletionPolicy: Retain
      Properties:
        AttributeDefinitions:
          -
            AttributeName: email
            AttributeType: S
        KeySchema:
          -
            AttributeName: email
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
        TableName: ${self:provider.environment.DYNAMODB_TABLE}

plugins:
  - serverless-plugin-stackstorm
```

Wow! The file doubled. The function itself is just 2 lines (35:36), but quite a few interesting additions took place. Let's review them: 

1. Line 9 - Table name generated to avoid collisions between regions and environments.
2. Lines 10:13 - IAM Role created to give the function access to the DynamoDB table.
3. Line 39:56 - AWS CloudFormation template defining the table.

The last point brings up an important observation: [Serverless framework][sf] simplifies mainly the FaaS part of serverless.
But because servereless is more than FaaS, you'll find yourself writing quite a lot of CloudFormation to provision other services.
Serverless framework leaves space for provider specific resources. To be seriously serverless on AWS, master CloudFormation. 
Looking for "provider-agnostic serverless"? Double-check your assumptions.

Also, I moved `memorySize` and `timeout` to apply to all functions (lines 6:7). And the API Gateway endpoint is gone from the InviteSlack function: it served the demonstration role in the first episode, but now we've learned to test Lambda functions directly. We'll return to API Gateway later to invoke the final StepFunction end-to-end workflow. 

Let's get the RecordDB function up to the sky and running. Deploy, invoke, logs. Rinse repeat.

```
sls deploy 
...
sls invoke --function RecordDB --logs --data '{"body":{"email":"santa@mad.russian.xmas.com", "first_name":"Santa", "last_name": "Claus"}}'
...

sls logs --function RecordDB. 
```


Now all three actions are here, waiting to be wired together into a final workflow with a StepFunction. But I'm just reminded not to be Mr. Scrooge: it's the holiday season, let's take it easy and put it off to the next episode. Until then, keep the spirits high! 

-----

The complete code example for this episode is on Github at [2-add-more-actions](https://github.com/dzimine/slack-signup-serverless-stormless/tree/DZ/2-add-more-actions)

Coming up in the next episode: Wiring the actions with a StepFunction.
