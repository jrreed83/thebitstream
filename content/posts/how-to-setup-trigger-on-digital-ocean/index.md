---
title: How to use Trigger.dev using Digital Ocean serverless functions.
author: Joey Reed
date: 2024-12-20
draft: true
summary:     
tags: ["automation"]
ShowToc: true
---

How can small businesses use modern technology to balance getting work done with lining up new work?  I started investigating this 
question a couple months ago and learned a lot in the process.  The common themes were leveraging generative AI (no surprise) and no-code
automation tools.  I had never heard of no-code automation tools before, so I did a little more digging. 

## No-Code Automation Tools
No-code automation tools let you build software systems by graphically combining third-party APIs without any programing.  Obviously this is great for someone with a product idea, but can't build it themselves and can't justify paying a developer.  The tool replaces the developer.  

The 3 most popular no-code platforms on the market today are  

* [make.com](https://www.make.com/en)
* [zapier](https://zapier.com/)
* [n8n](https://n8n.io/)

So far, I've used make.com to build out a simple experimental system that uses generative AI (OpenAI models for now) to convert existing content into short blog post drafts and emails them on for review.  Everything is managed through a single Google Sheet.  Here's what my make.com "scenario" looks like

![Make.com automation](./figures/make_automation.png)

Setting up this scenario was very easy.  The interface is extremely simple and well designed.  In fact, the hardest part was learning how to get make.com and Google to cooperate.  And even that was too tough, I just hadn't gone through the process before.

I have nothing bad to say about make.com.  It's a great tool, it's just not the right tool for me.  When you get right down to it, I'd rather write code
than navigate a user interface that does magical things in the background.

## Introduction to Trigger.dev 

I was going to try n8n out because it's supposedly geared toward programmers, but then I came across [trigger.dev](https://trigger.dev/).  Trigger.dev is definitely not a no-code platform.  Instead of dragging, dropping, and connecting blocks on a canvas, workflows are build by writing typescript (or javascript).  You're responsible for bringing any npm packages you need, including the ones that help communicate with RESTful services like OpenAI. 


## What we're going to Build

We're going to build a prototype system that can help teachers provide career advice to students based on information entered into
a Google Sheet.  Here's a diagram that shows the overall architecture

![carrer-advice-workflow](./figures/career-advice-flow.png)

and here's the technology we'll use to build it:

* Google Sheets
* Google Apps Script
* Digital Ocean Serverless Functions
* Trigger.dev
* OpenAI API
* Google Sheets API 
  
The teacher using the system doesn't have to worry about any of the technical details.  They'll
be completely invisible.  

You can find the project on Github.

## Digital Ocean Functions

Besides testing with their dashboard, there's no builtin way to trigger a task remotely.  You need to set up a webservice that trigger's the task when it receives a request.  The documentation does a good job of walking you through the process with a few different services and frameworks.  They don't specifically cover Digital Ocean's serverless functions, so I decided to try that.  I figured that would maximize my learning.    

Before doing anything, create a Digital Ocean account.  Once that's out of the way, we can start setting up our serverless function using 
Digitial Ocean's command line interface - **doctl** .  

We could do a lot of the heavy lifting using Digital Ocean's control panel, but I prefer operating from the command line as much as possible.  Here are the instructions for getting doctl setup.  I tried to keep them short, and added help comments along the way.  If you want more details, each step starts with a link to Digital Ocean's documentation. 
   

### 1. Install the `doctl` command line tool.

[https://docs.digitalocean.com/reference/doctl/how-to/install/](https://docs.digitalocean.com/reference/doctl/how-to/install/)

I'm currently running Linux and decided to download the cli from GitHub.  Here are steps taken from the link:  

#### 1. Download the archive.

```sh
cd ~
wget https://github.com/digitalocean/doctl/releases/download/v1.119.0/doctl-1.119.0-linux-amd64.tar.gz
```

#### 2. Unzip the archive.

```sh
tar xf ~/doctl-1.119.0-linux-amd64.tar.gz
```

#### 3. Put the unarchived CLI program in the search path.

```sh
sudo mv ~/doctl /usr/local/bin
```

#### 4. Create an API token for the CLI by 

#### 5. Get access to doctl

Let's say the name of the API key you set up in Step 4 is "my-do-api-key".  Run this
command to get access.

```sh
doctl auth init --context my-do-api-key
```

#### 6. Get serverless support 

To work with Digital Ocean's servless function product through doctl, you need to install an extension.  You'll figure this
out when you start going through the next few instructions.  To save you the trouble, run this

```sh
doctl serverless install
```

Now we're in position to start setting up a function.

### 2. Create a namespace

[https://docs.digitalocean.com/products/functions/how-to/create-namespaces/](https://docs.digitalocean.com/products/functions/how-to/create-namespaces/)

Namespaces give you a way to organize groups of serverless functions.  To create one, you need to give it a label and specify it's location.  Because I'm on the
west coast, I choose San Francisco.

```sh
$ doctl serverless namespaces create --label "trigger-dot-dev-fn-namespace" --region "sfo" 
```

If this was successful, you're connected to the new namespace and doctl responds with a message showing you the namespace ID and the "API Host".  Don't
worry about writing them down.  You can always get the information by asking for a list of your namespaces:

```sh
$ doctl serverless namespaces list
```

### 3. Create a function

[https://docs.digitalocean.com/products/functions/how-to/create-functions/](https://docs.digitalocean.com/products/functions/how-to/create-functions/)

Now it's time to add a serverless function to the namespace.  First, go to the place you want the project to get scafolded out.  Once you've done that
create a javascript project

```sh
doctl serverless init --language js doctl-trigger-dot-dev-with-google
```

### 3. Deploy

Digital Ocean won't know about the project we just had doctl create until it's deployed to their servers.  To deploy,
run the following command:

```sh
 doctl serverless deploy doctl-trigger-dot-dev-with-google/
```
The message returned by doctl will tell you whether or not the deployment succeeded.  

### 4. Test

We can execute the function through doctl, cURL, or the Digital Ocean control panel.  Here's how you'd execute
the function through doctl.

```sh
doctl serverless functions invoke career-project/skills-to-careers
```

You should get a response that looks like this

```sh
{
  "body": "Hello stranger!"
}
```

The `invoke` subcommand lets you pass in data to the function. This will be extremely helpful later on.  To see the full list of options,
ask for help in doctl:

```sh
doctl serverless functions invoke --help
```

Even if you don't want to test through the control panel, I'd still recommend visiting it.  It has some useful 
information about your function including specific cURL commands, the endpoint URL, and the API 
key you need to include in a POST request.

If you want to run cURL without going to the control panel, the following command will get the function's URL:

```sh
doctl serverless function get career-project/skills-to-careers --url
```

## Adding Trigger.dev

With the setup out of the way, we can start customizing our Digital Ocean serverless project.  

Our first function works, great!  Now it's time to start modifying the javascript project we created in Step 3 above.  

## Setting up Trigger.dev project

I'm going to assume that you've set up an account (under the Free Tier) and an organization within that account, on Trigger.dev.  

### 1. Create npm project

```sh
npm init -y
```

### 2. Setup a Fresh Project

Note, project has a new set of API keys. 

Accounts -> Organizations -> Projects

### 2. Initialize Trigger.dev project 

```sh
npx trigger.dev@latest init --javascript
```

### Digital Ocean

1. Create a namespace
```sh
doctl serverless namespaces create
```

2. Connect to namespace so we can add functions
```sh
$ doctl serverless connect
```


### Triggers in Google Sheets


```sh
curl -d '{"key1":"value1", "key2":"value2"}' -H "Content-Type: application/json" -X POST API_URL
```

```sh
curl -X POST "https://<your-server>.doserverless.co/api/v1/namespaces/<your-namespace>/actions/sample/hello?blocking=false" \
  -H "Content-Type: application/json" \
  -H "Authorization: <your-token>"
```

For some reason, I can can run the url fetcher in Google App Script without the authorization token.  It doesn't work when I run the cURL command in the terminal.  Expects authorization.  Weird.


To run the post request asynchronously, add special parameter to post request ...

```js 
const obj = UrlFetchApp.fetch(`${api_end_point}?blocking=false`, options);
```

What connects a deployment to an application in trigger?

When I re-deploy my function, I can't seem to trigger, trigger. Why?

Put "exclude" in tsconfig.json file.  Put the trigger config there ...

Import my task as a type, 
```js 
import type { digitial_ocean_task } from "./trigger/example";
```

```sh
npx trigger.dev@latest deploy
```
```sh
curl -d 'name=Sammy' https://faas-sfo3-7872a1dd.doserverless.co/api/v1/web/fn-9972eb5c-4690-46d5-822b-5515212c189c/sample/hello
``````


Google Stuff:
1. Installable trigger in Google Appscript to trigger DO function
2. Store digital ocean auth info in "Script Properties" of AppScript DashBoard.
3. Google Cloud project using API to communicate back to spreadsheet.  Need to setup a project, enable Google Sheets API in that project.  Use Service Project style authorization with a JWT (JSON Web token)


Got tyypescript compilation errors in the trigger.dev file.  Added annotation at top of file.  The issues were related to the google sheets API.

Error: While deploying action 'sample/hello': 413 Payload Too Large.  Isn't there supposed to be some sort of tree shaking or something?
Remember, it worked before, is it because the dependencies are getting added?  Yes. I ended up removing the dependencies required for the trigger task, manually.
This seems too weird and wacky.  Is this because we want all the tasks used for trigger to be devdependencies ?  

Need to run typescript, and then not install dependencies ...

I think the better option is to separate the trigger task from the backend task that "calls" it.  The backend code doesn't need any typescript.  The SDK can 
call the task via id alone...
