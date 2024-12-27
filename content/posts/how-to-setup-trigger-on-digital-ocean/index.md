---
title: How to use Trigger.dev using Digital Ocean serverless functions.
author: Joey Reed
date: 2024-12-20
draft: true
summary:     
tags: ["automation"]
ShowToc: true
---

How do small businesses use modern technology to balance getting work done with lining up new work?  

I started investigating this question a couple months ago and learned a lot about business development and 
marketing in the process.  One of the trends that came up consistently was using generative AI and "workflow automation tools" to 
automate critical, but repetitive, business tasks.  Here's a list taken straight from the landing page for one of the most
popular automation products out there, [zapier](https://zapier.com):

* Sales handoffs
* Markeing campagins
* Data management
* IT helpdesk
* Lead Management
* Revops 

Even with no programming experience, people are able to build amazingly sophisticated software systems that have a major impact on their business's bottom line. 

Over the course of my career, I've used many different types of software products and tools.  None of them helped me automate a business task.  Sounds pretty good to me.  I appreciate the value of good business strategy, but I want to spend my time doing what I get paid to do - engineering.  If these tools can help me speed-up and codify even a small part of the business, then it's worth learning more.    

## What We'll Cover

In this post we're going to start out by

1. describing what workflow automation tools are,
2. discussing why "no-code" platforms may not be the right choice,
3. introducing a powerful open source alternative called [Trigger.dev](https://trigger.dev/)

After that, I'll use Trigger.dev, Google Sheets, and generative AI to build a proof-of-concept career advice system.

## What is a Workflow Automation Tool

Workflow automation tools let you build software systems by conveniently combining third-party APIs.  The most popular ones, like

* [make.com](https://www.make.com/en)
* [zapier](https://zapier.com/)
* [n8n](https://n8n.io/)

require no programming.  Instead, you design the workflows visually by dropping blocks onto a canvas and connecting them together. Like this: 

![n8n automation](./figures/example-n88-automation.png)
 
Some of the blocks implement simple programming constructs like logic or iteration.  Others, usually called "integrations", communicate with APIs.  To use an integration,  you just need to plug in your API credentials.

Obviously this is great for someone with a product idea but doesn't have the development experience to build it themselves.  It's also an efficiency booster for experienced programmers.  Working through API documentation might be interesting for awhile.  But at some point, your time is probably better spent doing something else.  These tools can give you some of your time back.            

## Experimenting with make.com 

As an experiment, I built a simple make.com (aka Make) "scenario" that uses an OpenAI assitant to convert old written content into short blog post drafts, and emails them on for review.  Everything is managed through a single Google Sheet.

![Make.com automation](./figures/make_automation.png)

Implementing this scenario was very easy.  The most tedious part was setting up Google crededentials.  But this isn't Make's fault; that's always a pain.

I have nothing bad to say about Make.  It's a fantastic service, it's just not the right tool for me.  I prefer programming over working a user interface, no matter how great it is.  Yep, programming takes more work.  But in return you get more power, more flexibility, and more independence.  

This last point is particulary important to me.  As an electrical engineer,  it's very easy to become a victim of "vendor lock-in".  Adopting software tools is unavoidable, but you don't want to become so dependent on them that you forget the fundamentals.  If you can afford the time, it's best to strive for a first-principles as much as possible.  Then you can jump around and use whatever tool you want.                     

Make simply doesn't conform to my "first-principles" philosophy, so I started looking into alternatives.  I was planning on giving n8n a try because is has a reputation for being geared toward software developers.  But then I came across [Trigger.dev](https://trigger.dev/).    

## Introduction to Trigger.dev 

Trigger.dev is much different than other popular automation solutions.  Instead of connecting blocks on a canvas, you implement workflows using their node package and normal javascript.  You trigger the task from a separate application. There aren't any pre-built integrations.  If you want to connect to an API, you add the dependencies and write the code yourself.  

If you aren't getting a UI, and you're aren't getting integrations, what are you getting?    

![trigger.dev home](./figures/trigger-dot-dev-home-annotated.png)

Like their website suggests, you're getting a serverless deployment solution where you

> "Write workflows in normal async code and we'll handle the rest ..."

They've figured out a cost-effective way to containerize and execute long-running tasks on the web, without worrying about timeouts.  Time-consuming calculations,
multiple API calls, long-delays, and complex scheduling can all be expressed in code.  This opens up all sorts of possibilities that would be extremely difficult to implement in other serverless computing options.  No Function as a Service (FaaS) that I know of, will successfully run for longer than 15 minutes. Here's a list of popular options with their run-time limits:

* Digital Ocean Function: [15 minutes](https://docs.digitalocean.com/products/functions/details/limits/)
* Supabase Edge Functions: [150s-400s](https://supabase.com/docs/guides/functions/limits#runtime-limits)
* Vercel Functions: [15 minutes](https://vercel.com/docs/functions/runtimes#max-duration)
* AWS Lambda Functions: [15 minutes](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)

Based on these metrics, you won't be able implement an entire complex workflow in a single function.  What if you distributed the work between several functions?  Sure, you can do that, but this introduces more complexity.  Serverless functions can't be scheduled in code.  If you want to execute a function at some
time in the future, you'll need to modify a configuration file or database.  And even if you manage to nail the scheduling, you'll probably need to bring in a task queue to store data between functions.  

Trigger.dev handles all that extra complexity for you!

## What we're going to Build

We're going to build a proof-of-concept system that helps teachers give career advice to students based on information stored in a Google Sheet. 

![spreadsheet](./figures/spreadsheet.png)

You start by putting the student's name, age, and skills/interests in an open row.  When you're ready to get career advice, change the "Trigger" dropdown from "Waiting" to "Get Jobs".  A few seconds later, you'll get a list of possible careers for the student to consider.        

There are many ways to make this system more useful, accurate, and visually appealing.  I'll continue working on it.  My main objective here was to build something functional.

### System Architecture

If we used Make for this workflow, we'd only need to set up a few credentials, install the "Make for Google Sheets" extension, and connect up a few integrations.  Easy-peasy!

![Make extension](./figures/make-dot-com-extension.png)

With Trigger.dev, we need to figure out a lot of the details ourselves: 

1. how to trigger an event from a Google Sheet,
2. how to send data from a Google Sheet to another webservice,
3. how to interact with the OpenAI API,
4. how to programatically update the Google Sheet 

After some research, here's what I came up with:

![carrer-advice-workflow](./figures/career-advice-flow.png)
  
Changing the state of the dropdown from "Waiting" to "Get Jobs" triggers a Google Apps Script.  The script
takes the data from that row and sends it to a Digital Ocean Serverless Function that's waiting for requests at a 
particular URL.  The Digital Ocean Function does almost no work.  It's only purpose is to trigger a deployed Trigger.dev 
task with the spreadsheet data.  Think of it as a webhook.  Inside the task, the payload is destructured and integrated into
an OpenAI chat prompt.  Once the model's response is returned, the task uses the Google Sheets API to update the spreadsheet
with the career advice.

Yes, this workflow is simple enough that the whole thing could be put in the Digital Ocean function.  But if I want to 
extend the capabilities of this system later on, I'd rather get the right tools in place now.     

## Google Apps Scripts

Google Apps Scripts let you interact with Google Workspace products, like Google Sheets and Google Documents, in javascript.
All you need to do is open up a Google Sheet, click the "Extensions" Tab in the top toolbar, and select "Apps Script".  This opens up an editor 
that lets you implement and test your "container-bound" script, setup triggers, and add credentials.           

For our career-advice system, we implemented a so-called "installable trigger".  With an "installable trigger", you write custom logic and attach it
to a trigger through the Trigger Editor.  "Simple triggers" let you bypass the Trigger Editor, but we can't use them because we'll need to hit an API
that requires authorization. 

![apps-script-trigger-editor](./figures/apps-script-trigger-editor.png)

Remember, we want to run the script when the "Trigger" dropdown transitions from "Waiting" to "Get Jobs.  This is a job for the "On Edit" trigger.  

The last thing we need to do is find a way to store credentials without revealing them in the Apps Script.  Once we set it up, we'll need to store the URL and authorization token for our Digital Ocean function.  The simplest way to do this is adding the key and value as a script property, and retreve the value in the script when you needed.  Very similar to creating an `.env` file.  Script properties are accessible under the "Project Settings" tab. 

## Digital Ocean Serverless Functions

The Trigger.dev website covers several ways of triggering tasks from web frameworks and serverless functions.  They don't discuss using Digital Ocean's Serverless Function product, so I decided to try that.  Maybe someone using Digital Ocean will find it useful.  

We'll do everything with doctl, Digital Ocean's command line tool.  Instructions, including links to the official documentation are in my GitHub repository.    

When you initialize a serverless function wih doctl, you get a configuration file, 3 nested folders, and a single javascript function.    

With the setup out of the way, we can start customizing our Digital Ocean serverless project.  

The `project.yml` file in the 

![digital ocean project.yaml](./figures/digital-ocean-yaml.png)


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
