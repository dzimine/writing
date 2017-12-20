
# Episode 3


Build a serverless app on AWS with [Serverless Framework][sf], AWS Lambda, StepFunctions, DynamoDB, API Gateway, and ready-to-use functions from StackStorm Exchange open-source catalog.

[sf]: https://serverless.com/framework

You are watching the Episode Three. 

In the [Episode One][ep-one], I described the application we are building, walked you over setting up the development environment and creating Serverless project, and shown how to build a first Lambda function from [StackStorm Exchange][st2-ex] action with [Serverless Framework][sf].

In the [Episode Two][ep-two], we added more actions: one native Lambda to record user info to DynamoDB, and another one from StackStorm Exchange to make a call to ActiveCampaign CRM system. You learned more of `serverless.yml` syntax and practiced developer workflow working with individual functions. 

[ep-one]:https://medium.com/@dzimine/tutorial-building-a-community-on-boarding-app-with-serverless-stepfunctions-and-stackstorm-b2f7cf2cc419

[ep-two]: https://medium.com/@dzimine/building-community-sign-up-app-with-serverless-stepfunctions-and-stackstorm-exchange-episode-2-b1efeb1b9bd6
[st2-ex]: https://exchange.stackstorm.org 

In this episode, I'll show how to use AWS StepFunction to wire the actions into a workflow.

The complete code for this episode is on Github at [3-add-stepfunction](https://github.com/dzimine/slack-signup-serverless-stormless/tree/DZ/3-add-stepfunction) branch. 


#### Wire functions together with StepFunction

Now that the building blocks are lined up, time to wire them together. I'll use [`serverless-step-functions` plugin](https://github.com/horike37/serverless-step-functions) from Serverless Champion ... 

https://twitter.com/goserverless/status/941777242685767680

Install the plugin: 
```
npm install --save-dev serverless-step-functions
```

Add the plugin to your serverless.yml file

```
plugins:
  - serverless-plugin-stackstorm
  - serverless-step-functions
```

The Step Function definition will require my `accountID`; as this is a thing I don't like sharing with anyone, I put it to `env.yml`, which now look like this:

```
# env.yml
# Don't commit to Github :)
slack:
  admin_token: "xoxs-111111111111-..."
  organization: "your-team"
active_campaign:
  url: "https://YOUR-COMPANY.api-us1.com"
  api_key: "1234567a9012a12345z12aa2aaa..."
private:
  accountId: "000000000000"
```

Back to `serverless.yml` with a couple of key touches there:

```
custom:
  private: ${file(env.yml):private}
  stage: ${opt:stage, self:provider.stage}
  region: ${opt:region, self:provider.region}

stepFunctions:
  stateMachines:
    signup:
      events:
        - http:
            path: signup
            method: POST
            cors: true
      definition: ${file(stepfunction.yml)}
```

In the `custom` block, I assigned `private` from namesake key in `env.yml`, I also defined variables for `stage` and `region` so that the values are picked from CLI options, if provided, or defaults to the current AWS settings.

The `stepFunctions` block is here to define ... drumroll ... you have already guessed - StepFunctions. Mine is called"`signup`". The `events` section here looks, acts and servers just like `events` in function definition: it configures an API Gateway endpoint for invoking StepFunction from outside of AWS. We'll use it later in a web form. The definition takes. It can be written as YAML right here in `serverless.yml`, 
but I prefer to include it from a separate file to keep logic separate from configuration. Here it is: 


https://gist.github.com/dzimine/94f1a6dc1ba21b79ad09b24711a855b8

<script src="https://gist.github.com/dzimine/
94f1a6dc1ba21b79ad09b24711a855b8.js"></script>



```
# ./stepfunction.yml
# 
# StepFunction workflow definition file.
# Included in `serverless.yml`; variables are defined there.

Comment: "Community sign-up workflow"
StartAt: RecordDB
States:
  RecordDB:
    Type: Task
    Resource: arn:aws:lambda:${self:custom.region}:${self:custom.private.accountId}:function:${self:service}-dev-RecordDB
    Next: RecordAC
    ResultPath: $.results.RecordDB
  RecordAC:
    Type: Task
    Resource: arn:aws:lambda:${self:custom.region}:${self:custom.private.accountId}:function:${self:service}-dev-RecordAC
    Next: InviteSlack
    ResultPath: $.results.RecordAC
  InviteSlack:
    Type: Task
    Resource: arn:aws:lambda:${self:custom.region}:${self:custom.private.accountId}:function:${self:service}-${self:custom.stage}-InviteSlack
    ResultPath: $.results.InviteSlack
    End: true

```


The StepFunction definition is written in [Amazon States Language](https://states-language.net/spec.html); the spec is short, well written and worth a read. Using YAML instead of JSON is a nice perk from the plugin - it reads better and allows comments. But if you want JSON - no problem, help yourself.

`Resource` is used to refer the Lambda function by ARN. I used the variables we defined above to construct the ARSs matching account ID, region, and stage.

`arn:aws:lambda:${self:custom.region}:${self:custom.private.accountId}:function:${self:service}-${self:custom.stage}-RecordDB`

`ResultPath` is used to pass data between steps. By default, StepFunction works on the "need-to-know" basis: the step downstream receives the output from the step directly upstream. If you think it logical, think again: how does RecordAC and InviteSlack are gonna get contact's email? RecordDB may just return "200 OK". Forcing functions re-emit their input would introduce inappropriate intimacy. The trick is to use `ResultPaht` to append the output of each function under some key, like `ResultPath: $results.RecordDB`. Overriding `ResultPath` preserves initial workflow input for use from other Lambda steps. To fully grasp it, read the ResultPath section in the docs, or enjoy a ... < TODO: post from > 


> PRO TIP: I made workflow sequential to demonstrate data passing. It is more proper to run all three things in parallel: it is faster and more resilient as a failure of one integration point will not stop workflow execution and the others will still be invoked. Go ahead change the StepFunctions to parallel.

That is it. Time to try. Deploy, invoke, check the logs. 


You CURL funs know what to do with the new API Gateway endpoint for our StepFunction. where exposed our I'll show off again with httpie, you CURL fanatics know 

```
# DON'T COPY! Use YOUR ENDPOINT!

http POST https://YOUR.ENDPOINT.amazonaws.com/dev/signup \
body:='{"email":"santa@mad.russian.xmas.com", "first_name":"Santa", "last_name": "Claus"}'
```

Ok, httpie or curl, it returns the StepFunction execution ARN so that we can check on it and see how the StepFunction is being executed. How? I afraid gotta open a browser and login to your AWS Console. First, try to use AWS CLI, don't say I didn't show you how:

```
aws stepfunctions describe-execution --execution-arn arn:aws:states:us-east-1:00000000000:execution:SignupStepFunctionsStateMac-seo9CrijATLU:cbeda709-e530-11e7-86d3-49cbe4261318 --output json
{
    "status": "FAILED", 
    "startDate": 1513738340.18, 
    "name": "cbeda709-e530-11e7-86d3-49cbe4261318", 
    "executionArn": "arn:aws:states:us-east-1:00000000000:execution:SignupStepFunctionsStateMac-seo9CrijATLU:cbeda709-e530-11e7-86d3-49cbe4261318", 
    "stateMachineArn": "arn:aws:states:us-east-1:00000000000:stateMachine:SignupStepFunctionsStateMac-seo9CrijATLU", 
    "stopDate": 1513738370.481, 
    "input": "{\"body\":{\"email\":\"santa@mad.russian.xmas.com\",\"first_name\":\"Santa\",\"last_name\":\"Claus\"}}"
}
```


This is an output for the execution that failed because RecordAC function timed out. The only valuable info here is `FAILED`. No kidding. I must say AWS don't give StepFunction the love it deserves. Not in CLI. If you think I missed some, check [the docs](http://docs.aws.amazon.com/cli/latest/reference/stepfunctions/index.html) yourself. 

> PRO TIP: For debugging, invoke the StepFunction with `sls infoke stepf`: it creates the execution, waits for the completion, and prints the output to the terminal. Three AWS CLI commands in one. 
> 
>    ```
>    sls invoke stepf --name signup --data  '{"body": {"email":"santa@mad.russian.xmas.com", "first_name":"Santa", "last_name": "Clause"}}'
>    ```


The most irritating part is that CLI doesn't tell which step failed. They make me call the logs on every Lambda, one by one. Or, open a browser and hop on AWS Console. Even there, debugging some Lambda failures, like timeouts, is tricky. StepFunction execution "Exception" report says `"The cause could not be determined because Lambda did not return an error type."` Go to the lambda logs to see what happened there. There I find a line which I wish to see in StepFunction exception. Hope AWS will listen. 

```
2017-12-20T04:21:44.298Z 4230a73b-e53d-11e7-be6b-bff82b9b3572 Task timed out after 6.00 seconds
```

Your StepFunction executions may work from the first time - we already adjusted the timeouts. I took you on this debugging detour for a taste of  StepFunction troubleshooting, admittedly a bit bitter. On the sweet side, once debugged, StepFunctoins runs reliably like Toyota cars.


> PRO TIP: Symptom: step function execution fails with one of Lambda returning with this message`The cause could not be determined because Lambda did not return an error type.` Most likely reason: Lambda timeout. Fix: investigate the cause, increase timeout. 

As a backend developer, I am tempted to call the it done here and go home for Christmas. But something is still missing to call it "a complete example". Like a Web UI. Let's add some. 


### Add Web UI


We need a web form that takes user name and email, and posts to our StepFunction API Gateway endpoint. I just copied the one I used while [exploring Serverless with Python, StepFunctions, and Web Front-end Serverless](https://github.com/dzimine/slack-signup-serverless). Which, I confess, I grabbed it from an old [Serverless Slack Invite](https://github.com/serverless-london/serverless-slack-invite). Those of you who are Web developers can surely create something more elegant with ReactJS. PR is welcome! Weahter you build your own or grab mine, do this: place the static content into `web` directory.

I use [`http-server`](https://www.npmjs.com/package/http-server) for a quick look that at my static form:

```
cd web
http-server
Starting up http-server, serving ./
Available on:
  http://127.0.0.1:8080

```

Open a browser, check the form at [http://127.0.0.1:8080](http://127.0.0.1:8080) and let's move on. 

How serverless web front end is deployed to AWS? Typically, you put the static content on S3, configure it to serve the web, make it call your backend endpoint, and not forget to enable CORS on API Gateway endpoint. Alternatively, you add a path to your API Gateway resource to serve your static web content from the S3 bucket. Pros: 1) no mess with CORS, 2) no adjusting your web to point to the correct backend endpoint. Extra bonus: easily done with [serverless-apig-s3][apig-s3] plugin. Cons: 1) paying API Gateway charges for web requests 2) the plugin is a bit shaky: fine for examples and small apps, but I wouldn't use it for anything resembling production.

[apig-s3]: https://github.com/sdd/serverless-apig-s3


> PRO TIP: For serious web front ends with high loads: deploy to the CloudFront. Or use something like Netlify - check out ["How to build a static Serverless site with Netlify"](https://serverless.com/blog/how-built-static-serverless-website-netlify/)

I'll use the API Gateway approach here: we don't expect massive load,
and it is simple. Very simple. Just watch:


Install plugin (been there, done that)
```
npm install --save-dev serverless-apig-s3

```

Change the `serverless.yml`. Add a plugin (been there, don that). Add [plugin configurations][apig-s3]:

```
...
custom:
  private: ${file(env.yml):private}
  stage: ${opt:stage, self:provider.stage}
  region: ${opt:region, self:provider.region}
  apigs3:
    dist: web
    topFiles: true

...    

plugins:
  - serverless-plugin-stackstorm
  - serverless-step-functions
  - serverless-apig-s3
```
We are telling the plugin to pick our `web` directory and put it . `topFiles` expose our index.html and `formsubmit.js` at your endpoint under `/web/`

Now deploy the service, and separately the client side, with two commands:

```
sls deploy

sls client deploy
```

Deploying the service will update API Gateway with the paths to access web content. Deploying the client puts the web content up to the S3. Now you understand that if you add or remove web files, client deploy is not enough, a full sls deploy is required. But if you only change the web files, quick `sls client deploy` will put them up on AWS.

That's it! Go to https://YOUR-ENDPOINT.amazonaws.com/dev/web/index.html The form is right there. The button is orange. You type in the user info and email, hit the orange button, the green message tells you everything OK. Soon you receive an invitation from Slack. You check the StepFunction executions in AWS console and saw everything is green.


No?! Most likely, the [apig-s3 bug](https://github.com/sdd/serverless-apig-s3/issues/16), and you got to do a little final step manually. Open AWS Console, ->API Gateway->Resources->Actions-Deploy API. Or use AWS CLI:

```
# Find out your rest-api-id first
aws apigateway get-rest-apis

# Deploy the changes to the stage
aws apigateway create-deployment --rest-api-id  1nort0aotd  --stage-name dev
```

Hope it works now.

Congratulations! You made it. You might followed along and built your own app. You might drop it at some point of irritation. You might skimmed the text and examples. But if you made it to the end, you learned enough of serverless to be dangerous.


As you gathered by now, serverless is not simple. Many things can go wrong as you build the app. You can blame some on me: I surely made a few mistakes and omissions (BTW reports and comments much appreciated!) But serverless complexity is not all my fault. We wire together many different services - AWS and 3rd party - with many different wires - frameworks and tools. The services are of different quality and [in]convenience. The framework and tools are still maturing.

Why getting into serverless? Because for some classes of apps, it is substantially cheaper. This community signup is a case to point: we used to run StackStorm's with StackStorm. It was simple to build and solid like a rock, but took (before you get to multi-zone reliability). It didn't justify, given the expected amount of community signups. On the other extreme, high volume apps with high reliability requirements might be better off serverless for "infinite" elasticity which may be expensive to . I use "might" and "would" as it is not a clear cut and application specific. Do your math upfront, and keep watching the bill.

As for taming complexity, Serverless framework helps. To fully appreciate it, try and redo the same app without it, as reliable, repeatable, reviewable infrastructure-as-code. While Serverless framework is not the only game in town, I specifically like it for pluggable architecture and ecosystem of plugins.

Plugins help, picking up the areas not covered by Serverless core. We have enjoyed the simplicity of building StepFuncions and adding simple web front-ends (and if you haven't, try and redo it without the plugins). Skim and bookmark the [list of official plugins](https://github.com/serverless/plugins) to keep them in mind when for your next project.

StackStorm Exchange - the recent addition to the serverless eco-system to - brings up a catalog of reusable integrations. It is not difficult to scrape APIs for ActiveCampaing or find and use undocumented Slack API calls. But it adds up to low-value routines, taking away from the focus on building the application. 

<some good closing words> 


### Final Tips and Tricks

* Removing everything will not remove the DynamoDB table. A reasonable default, but when you try and re-deploy the service, it'll cry that the table can't be created as it already exists. Delete it: `aws dynamodb delete-table --table-name signup-stormless-dev`


* Due to apig-s3 bug, `sls remove` will fail complaining that the bucket is not empty. Remove the web s3 bucket manually when removing a stack. `aws s3 rb s3://bucketname --force`

* Removing the stack does not remove the DynamoDB table. If you really want to start from scratch, delete it manually (export the data first): aws dynamodb delete-table --table-name slack-signup-dev


* Sometimes `sls delete` fails to clean up. Reasons might be various, but the place to look is the same. Did I tell you to master CloudFormation? Go there, find a stack that might failed to delete, find the reason, fix, and delete the stack.  


# TODO's and notes
* [TODO] Split gist to multiple gists before inserting.
* [TODO] Fix package.json versions in all branches to `1.1.0`
* [TODO] Put "body" back in text, once I do the workflows. 
* [TODO] Try and do separate deployment...
* [NOTE] I like how data flow is visible in `serverless.yml`. In typical lambda, it's hidden inside lambda handlers and it's not obvious what goes in and what comes out. Here it's a clear view. 

* [FEATURE] StackStorm Exchange improvements: 1) see actions on pack 2) see open PRs across exchange + PRs per pack ? 



BUG] ActiveCampaign.contact_add shall not throw exception if the user already exist, but return the 200 and message just like AC API does. - FIXED! 



NOTE: I wanted to make a tiny change to RecordDB. As deployment takes a while, I wanted to use the new cool Cloud9 editor and hack right on the cloud... But I couldn't: deployment package is too big. REALLY?RecordDB size is the same as for other actions. On second though, it's right: one serverless service - one deployment package (you can check it in `./.serverless/SERVICE_NAME.zip`). So either native or Exchange, actions are bundled together and all dependencies of all actions got deployed.

To fix it, I shall use `package: individually: true`. And it's the right thing, too. But I hit an annoying bug. When serverless is trying to form Exclude files... It's likely related to my "npm link` problem so I'll test again.

```
  EMFILE: too many open files, scandir '/Users/dzimine/Dev/serverless/slack-signup-serverless-stormless/node_modules/serverless-plugin-stackstorm/node_modules/node-pre-gyp/node_modules/ajv/lib/dot/v5/_formatLimit.jst'
```
after removing link, it's now 
```
  ENFILE: file table overflow, scandir '/Users/dzimine/Dev/serverless/slack-signup-serverless-stormless/~st2/deps/share/doc/networkx-1.11/examples/3d_drawing/mayavi2_spring.py'
```

I'll save it for later. 



(sf): https://serverless.com/framework