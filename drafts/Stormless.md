
# Tutorial: Building community on-boarding app with Serverless, StepFunctions, and StackStorm Exchange.

Build a real-world serverless application on AWS, using Serverless Framework and ready-to-use functions from StackStorm Exchange open-source catalog. The tutorial also touches on AWS Lambda, API Gateway, StepFunctions, DynamoDB, Slack, and ActiveCampaign. 

Who should read it: 

* Serverless developers using Serverless framework who want to check out ready-to use functions from StackStorm Exchange open-source catalog,
* StackStorm users who live in AWS and miss the breadth of StackStorm exchange integrations there. 
* Anyone who has 3 hours to follow-alone and learn serverless by example more elaborate and real-life than a Hello-world app. 

If you only have _TODO_ min and already into serverless, skim the text and examples, spend extra 30 seconds browsing StackStorm Exchange to see the potential, and bookmark this post to get back to it when you need it.

### Intro

When I [explored Serverless with Python, StepFunctions, and Web Front-end](https://medium.com/@dzimine/exploring-serverless-with-python-stepfunctions-and-web-front-end-8e0bf7203d4b), one thing I missed is a catalog of reusable integrations.  Something like 200 connectors for Azure Logic apps. Or 130 integration packs for StackStorm. When we need to wire in Slack, Jira, Stripe, or Nest, could we skip digging into their APIs and authentication intrinsics, and just grab a ready-to-use function?. 

Now we can do exactly that: StackStorm just pushed a plugin for Serverless framework that turns integrations from StackStorm Exchange into AWS Lambda functions. 

In this tutorial, I’ll show how to use the plugin and Exchange integrations, in the context of building a serverless community on-boarding application from a ground-up.  It’s a holiday season, the spirits are high, I have no time or inclination to rigorous writing; instead let’s make it conversational and fun. 

I assume no familiarity with neither Serverless framework nor StackStorm. But you should know how to code, and gotta be smart to compensate for mistakes and omissions I'll inevitably make. If you're lost... never mind, no one gets lost nowadays with Google @ Stack overflow, and GPS on smartphone. 

We will be goings slow, with excruciating details, thus it is going to be three episodes. In the first episode, I'll set everything up and deploy my first StackStorm Exchange action. In the second episode, we'll add more actions. In the third, we use StepFunctions to wire the actions into the workflow, and sum things up. Each episode should take about an hour to follow alone. 

Let's rock.


### What we build

We will build a community on-boarding application. Actually, rebuilding from scratch the one we run at StackStorm. It’s like “…”, with customizable on-boarding workflow. The app presents a registration web-form, which passes new user info through API Gateway to the StepFunction workflow that carries on-boarding steps. In my case, the steps are 1) invite using to Slack 2) create contact record in ActiveCampaign CRM tool and 3) put a user record into internal DynamoDB table. Here is how it looks:

<pic>

The previous implementation is here, you useit here and join StackStorm community for questions about StackStorm Exchange integration.


### Getting Ready

First, you'll need AWS account, NodeJS, Docker, and Serverless framework. And Slack! As our first action will be inviting users to Slack. 

1. Make sure [Node.JS](https://nodejs.org/) is installed, and it's version is `8.4.0` or higher. 
2. [Install Serverless framework](https://serverless.com/framework/docs/providers/aws/guide/installation/), and setup AWS credentials for it [following this guide](https://serverless.com/framework/docs/providers/aws/guide/credentials/).
3. [Install Docker](https://docs.docker.com/engine/installation/). The plugin uses it for build environment to make the lambdas binary compatible to AWS execution environment no matter what OS you’re using for development. There is a way to make it work without docker but don't take chances. And if you don't have Docker installed, what are you doing here reading this anyways... Get out!
4. Slack! Our action will require admin access and will using undocumented Slack API (docs here, pun intended) to invite new users. The easiest is to just go ahead [create a new team](https://slack.com/get-started). Takes 4 min. Slack wouldn't mind - they'll show growth to their VC. 

   Once the workspace is created, time to get an authentication token. Go to [api.slack.com/custom-integrations/legacy-tokens](https://api.slack.com/custom-integrations/legacy-tokens). Fear not the "Legacy Warnings": this tutorial will turn legacy before they do. Do what they say, get your token.
  
  > PRO TIP: Use [this quick hack](https://github.com/StackStorm-Exchange/stackstorm-slack#obtaining-auth-token) to get and use your own user's auth token. Much faster, good for playing and debugging, but please never, never use it for production!  

### Create a project, add a first action

Try `sls --help` to make sure that at least something works. `sls` is shorthand for `serverless`, the Serverless framework CLI. Now put your coffee aside, time to create a project. Some like using templates that come with `serverless create --template`. I prefer to start from scratch: 

``` 
mkdir slack-signup-serverless-stormless
cd slack-signup-serverless-stormless

npm init

# Once you answer questions, the project is set up.

```

Next, install [`serverless-plugin-stackstorm`](https://github.com/StackStorm/serverless-plugin-stackstorm), the one that plugs in the [StackStorm Exchange](https://exchange.stackstorm.org) actions.

```
npm install --save-dev serverless-plugin-stackstorm
```

Create the first action. I'll use a battle-tested [Slack pack from StackStorm Exchange](https://exchange.stackstorm.org/#slack). While StackStorm Exchange is not smart enough yet to show pack's action list, `sls stackstorm` will rescue us.

`sls stackstorm info --pack slack` 

Oh my! There're so many! what are they? I guess I need a PR to print action description. Meantime, `| grep admin` will get us the one we need: `slack.users.admin.invite`. Let's inquire the action for it's parameters:

```
$ sls stackstorm info --action slack.users.admin.invite
slack.users.admin.invite ...... Send an invitation to join a Slack Org
Parameters
  attempts [integer]  ......... description is missing
  channels [string]  .......... Channels to auto join.
  email [string] (required) ... Email address to send invitation to.
  first_name [string]  ........ Users first name
  set_active [boolean]  ....... Should the user be active.
  token [string]  ............. Slack API token.
Config
  action_token [string]  ...... Slack Action token.
  admin [object]  ............. Admin-action specific settings.
  post_message_action [object]   Post message action specific settings.
  sensor [object]  ............ Sensor specific settings.
```


Awesome! We can see that there is only one required parameter, `email`, but I'll add `first_name` to be conversational. The token can be passed as parameters, or as config. And if I choose to use config, my prior tribal knowledge hints that the `admin [object]` requires only `admin_token`. The very one I asked you to remember when you were setting up Slack workspace.

> PRO TIP. While we are still polishing the plugin to expose all the Config details, you can find it out by exploring StackStorm Exchange pack config schema in `config.schema.yaml` file. For example, here is [config.example.yaml](https://github.com/StackStorm-Exchange/stackstorm-slack/blob/master/config.schema.yaml#L47-L78) for our Slack pack.


Now we have all we need to create the heart of any Serverless project: the `serverless.yml`. Here it comes: 

```
service: signup-stormless

provider:
  name: aws
  runtime: python2.7

functions:
  InviteSlack:
    events:
      - http:
          method: POST
          path: invite
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

plugins:
  - serverless-plugin-stackstorm
```

This is a good time to learn a bit of Serverless. Take a quick break to skim [Core Concepts](https://serverless.com/framework/docs/providers/aws/guide/intro/) and bookmark [Serverless.yml Reference](https://serverless.com/framework/docs/providers/aws/guide/serverless.yml/).

I threw in the `events` section here so that we can invoke the function with REST call through AWS API Gateway endpoint. Serverless framework will do all the Gateway magic. Note that this default configuration instructs API Gateway to pass the REST POST call with POST body under the `body` key ([details here](https://serverless.com/framework/docs/providers/aws/events/apigateway#simple-http-endpoint). When we POST `{"first_name": "Santa", 
"email": "santa@mad.russian.xmas.com"}`, the event passed to the Lambda is:

```
...
"body": {
    "first_name": "Santa", 
    "email": "santa@mad.russian.xmas.com"
}
```
Knowing the input data structure is important to map it to the action input parameters. It's intuitive: `input` represents `event` parameter of [AWS Lambda programming model](http://docs.aws.amazon.com/lambda/latest/dg/python-programming-model-handler-types.html) (BTW should we call it `event`? Vote with a PR!). [Jinja](http://jinja.pocoo.org/) is used to map the inputs; our JavaScript friends who're less familiar
with this common Python tool find it intuitive enough in simple cases; and Stack-overflow is full of magic Jinja tricks.

Here we obviously map the two parameters from input body to desired action input parameters. 

To keep the config separate, I created a file `env.yml` and put my config parameters there:

```
# ./env.yml
# WARNING: Don't commit to Github :)
slack:
  admin_token: "xoxs-111111111111-111111111111-111111111111-1111111111"
  organization: "my-team"

```
Then I used it in `serverless.yml` like this: `admin: ${file(env.yml):slack}`. Note how this syntax puts the object from the key in the file to the key in `serverless.yml`. 

Optionally, we can also form a function output from action results. I'll keep it simple for now and save more tricks for later.

Ok, that's it! The function is ready to go to AWS. But I take it sloooow. First, I'll package it locally. 

```
sls -package
```
The very first time takes a long time as this is the time when the plugin installs it's runtime dependencies. It pulls the Docker images from the Hub  It installs StackStorm runners - the code that knows to run StackStorm exchange packs. It pulls the `slack` pack from Exchange. It installs `slack` pack python dependencies. It does a lot of work for us, and it takes time. Good news: it's only the first time.

Oh, did I say you got to be connected? Or we assume internet connection a basic commodity like briefing air and electric power? At least before FCC repeals Network Neutrality? So yes, you need internet connection to live serverless.

Now let's run this locally.

```
sls stackstorm docker run --function InviteSlack --verbose --data '{"body": {"first_name": "Santa", 
"email": "santa@mad.russian.xmas.com"}}'
```

Local runs happen in the container. It takes a bit longer, but ensures that the execution environment matches AWS lambda very closely, so better safe than sorry. 

When debugging input and output parameter transformation, you may not want to call the actual function, like in case of Slack rate-limiting API. Use `--passthrough` parameter that tells the plugin to skip the action invocation.

Now we are really ready ready. Deploy the function to AWS, and run it "serverless".

```
sls deploy
```

It will take some while - now it's serverless (and honestly, our bundle is a bit bloated, patience! plugin developers are currently busy solving other problems, we will optimize it as soon as we can)


> PRO TIP: if something goes wrong at this point, most likely something is not right with your AWS setup. Go to "Getting Ready, step 2". Read [Serverless Installation](https://serverless.com/framework/docs/providers/aws/guide/installation/) doc. Google, Stack-overflow, Serverless [Gitter channel](https://gitter.im/serverless/serverless) or [Forum](https://forum.serverless.com/).
 

You might be curious to see how it looks in the AWS console. If the PRO in you is saying "no, you should stay cool and use CLI", don't listen. Go indulge yourself, open a browser and take a good look to your Lambda. While there, you might also inspect the API Gateway endpoint that `sls` created for you. 

But to test it, we'll go back to terminal. Here is how to run your Lambda with `sls`:

```
sls invoke --function InviteSlack --log --data '{"body": {"first_name": "Santa", "email": "santa@mad.russian.xmas.com"}}'
```

Finally, let's POST to the API endpoint. The endpoint was printed at the end of `sls deploy` and you should have taken notice, but it's OK if you didn't: you can always get it by typing `sls info`. 

You `curl` lovers go ahead use it to POST; be sure to set `Content-Type=application/json` header. Me - I'll show off with [httpie, aka CURL for humans](https://httpie.org/):

```
# DON'T copy-paste! Use YOUR endpoint!

http --json POST  https://YOUR-ENDPOINT.amazonaws.com/dev/invite \
email=test@example.com first_name=Dmitri
```

How did it go? Everything worked, at least for me. Let's fire yet another most useful `sls` command to check the CloudWatch logs:

```
sls logs --function InviteSlack
```

Success! And the end of the first episode. Enough for now: Christmas time is here, take it slow.

The code example so far is on Github at [1-add-first-action](https://github.com/dzimine/slack-signup-serverless-stormless/tree/DZ/1-add-first-action). Grab `.gitignore` from there if you are using Git. Yes, you should. And if you're following alone, it's a good time to commit now, before we move to next step. Grab this [`.gitignore`](https://github.com/dzimine/slack-signup-serverless-stormless/blob/DZ/1-add-first-action/.gitignore) if you are using Git.

> PRO TIP: Avoid pushing slack token or other credentials to GitHub by mistake. PRO who made this mistake are now preventing it by putting `env.yml` to `.gitignore`. 

 
In the next episode: Adding more actions.

# Episode 2.

### Adding more actions

