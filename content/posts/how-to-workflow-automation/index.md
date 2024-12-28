---
title: Workflow Automation with Trigger.dev
author: Joey Reed
date: 2024-12-20
draft: true
summary:  Improve Business Efficiency
tags: ["automation"]
ShowToc: true
---

How can small businesses use modern technology to improve ...  

I started investigating this question a couple months ago.  One of the trends that came up consistently in my research was the use of generative AI and "workflow automation tools" to automate critical, but repetitive, business tasks.  Here's a list of some use cases taken straight from the landing page for one of the most
popular automation products, [zapier](https://zapier.com):

* Sales handoffs
* Markeing campagins
* Data management
* IT helpdesk
* Lead Management
* Revops 

Even with no programming experience, people are able to build amazingly sophisticated and useful software systems that have a major impact on their business's bottom line. 

Over the course of my career, I've used many different software products.  Not one has helped me automate a business task.  Sounds pretty good to me.  I want to spend my time doing what I get paid to do - engineering.  If these tools can help me codify and reduce the time I spend on some business process, then I figure it's worth learning more about them.    

## What We'll Cover

In this post we're going to 

1. describe what workflow automation is,
2. discuss why "no-code" platforms may not be the right choice,
3. introduce a powerful open source alternative called [Trigger.dev](https://trigger.dev/)

After that, I'll describe how I used Trigger.dev, Google Sheets, and generative AI to build a proof-of-concept career advice system.

## What is a Workflow Automation Tool

Workflow automation tools let you build software systems by conveniently combining third-party APIs.  The most popular ones, like

* [make.com](https://www.make.com/en)
* [zapier](https://zapier.com/)
* [n8n](https://n8n.io/)

require no programming.  Instead, you design workflows visually by dropping blocks onto a canvas and connecting them together. Like this: 

![n8n automation](./figures/example-n88-automation.png)
 
Some of the blocks implement simple programming constructs like boolean logic or iteration.  Others, usually called "integrations", communicate with APIs.  To use an integration, you just need to plug in your API credentials.

Obviously this is great for someone with a product idea but doesn't have the development experience to build it.  It's also an efficiency booster for experienced programmers.  Working through API documentation might be interesting for awhile.  But at some point, your time is probably better spent doing something else.  These tools can give you some of your time back.            

## Experimenting with make.com (aka Make)

In Make, workflows are called "scenarios".  As an experiment, I built a simple scenario that uses an OpenAI assitant to convert old written content into short blog post drafts, and emails them on for review.  Everything is managed through a single Google Sheet.  Here's what the scenario looks like in the Make dashboard:

![Make.com automation](./figures/make_automation.png)

Implementing this was very easy.  The most tedious part was setting up my Google credentials.  Even that wasn't hard, just unfamiliar.  You end up having to access your Google Cloud account, which I hadn't done before.

I have nothing bad to say about Make.  It's a fantastic service, it's just not the right tool for me.  I prefer programming over working a user interface, no matter how great it is.  Programming takes more work, but in return you get more power, more flexibility, and more independence.  

This last point is particulary important to me.  As an electrical engineer,  it's very easy to become a victim of "vendor lock-in".  Adopting software tools is unavoidable, but you don't want to become so dependent on them that you forget the fundamentals.  If you can afford the time, it's best to strive for a first-principles as much as possible.  Then you can jump around and use whatever tool you want.                     

Make simply doesn't conform to my "first-principles" philosophy, so I started looking into alternatives.  I was planning on giving n8n a try for several reasons, but then I came across [Trigger.dev](https://trigger.dev/).    

## Introduction to Trigger.dev (aka Trigger)

Trigger is much different than other popular automation solutions.  

* Instead of connecting blocks on a canvas, workflows are implemented by adding normal Typescript/Javascript to their carefully engineered "task" object.
* Integrations aren't included.  If your task needs to communicate with an API, you need to being in the appropriate dependencies and write the code yourself. 
* General webhooks aren't included.  Tasks can be executed from Trigger's dashboard, with their Node SDK, or their REST API.  
* It's implemented as a serverless architecture.

If you aren't getting drag-and-drop functionality, and you're aren't getting built-in integrations, what are you getting?    

![trigger.dev home](./figures/trigger-dot-dev-home-annotated.png)

Like their website suggests, you're getting a serverless deployment solution where you

> "Write workflows in normal async code and we'll handle the rest ..."

They've figured out a cost-effective way to package and execute long-running tasks on the web, without worrying about timeouts.  Time-consuming calculations,
multiple API calls, long-delays, and complex scheduling can all be expressed in code.  This opens up all sorts of possibilities that would be extremely difficult to implement with other serverless computing options.  No Function as a Service (FaaS) that I know of, will successfully run for longer than 15 minutes. Here's a list of popular options with their run-time limits:

* DigitalOcean Function: [15 minutes](https://docs.digitalocean.com/products/functions/details/limits/)
* Supabase Edge Functions: [150s-400s](https://supabase.com/docs/guides/functions/limits#runtime-limits)
* Vercel Functions: [15 minutes](https://vercel.com/docs/functions/runtimes#max-duration)
* AWS Lambda Functions: [15 minutes](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)

There are cetainly workflows that could take longer than 15 minutes.  For example, you might want to schedule a workflow to execute every day at 8:45am.  You can do this with serverless functions, but you'll be forced to either add the schedule to a database or a configuration file.  You can't do it in code.  

Other applications might force you to distribute the work you'd like to do in one serverless function, over several of them.  Now you'll probably have to bring in a task queue, or some other way to manage state between function calls.  This introduces a lot of complexity that you may not have the time, or expertise, to deal with.  Trigger.dev handles all the extra complexity for you!

## What we're going to Build

We're going to build a proof-of-concept system that helps teachers give career advice to students based on information stored in a Google Sheet. 

![spreadsheet](./figures/spreadsheet.png)

You start by putting the student's name, age, and skills/interests in an open row.  When you're ready to get career advice, change the "Trigger" dropdown from "Waiting" to "Get Jobs".  A few seconds later, you'll get a list of possible careers for the student to consider.        

There are many ways to make this system more useful and visually appealing.  I'll continue working on it.  My main objective was to simply build something functional.

### System Architecture

If we used Make for this workflow, we'd only need to set up a few credentials, install the "Make for Google Sheets" extension, and drag in a few integrations.  Easy-peasy!

![Make extension](./figures/make-dot-com-extension.png)

With Trigger, we need to figure out a lot of the details ourselves: 

1. how to trigger an event from a Google Sheet,
2. how to send data from a Google Sheet to an API,
3. how to interact with the OpenAI API,
4. how to programatically update a Google Sheet with an API 

Here's what I came up with:

![carrer-advice-workflow](./figures/career-advice-flow-trigger-api.png)
  
Changing the dropdown state from "Waiting" to "Get Jobs" triggers a Google Apps Script.  The script packages up some information about the sheet and the edited row,  and posts the payload to my task's URL endpoint.  Once triggered, the task extracts the student's skills from the payload and adds it to an OpenAI chat prompt.  When OpenAI returns with a valid response, the task uses the Google Sheets API to update the Google Sheet remotely.
  
## Setting up the Google Apps Script

Google Apps Scripts let you interact with Google Workspace products, like Google Sheets and Google Documents, in Javascript.  To attach one to to Google Sheet, open up a Google Sheet, click the "Extensions" Tab in the top toolbar, and select "Apps Script".  This opens up an editor that lets you implement and test your "container-bound" script, setup triggers, and add credentials.           

I wanted the Google Apps Script to

1. Retrieve information about the sheet
2. Forward the sheet information to an API

whenever the "Trigger" column was edited.  To achieve this, I created an "installable trigger" by attaching the "On Edit" event to my primary function.  

![apps-script-trigger-editor](./figures/apps-script-trigger-editor.png)

This causes the function to execute anytime the spreadsheet is edited.  I added a little extra logic to avoid hitting the API when irrelevant cells were edited.   

I also needed a way to store my Trigger credentials in the Apps Script project without revealing them in the source code.  One of the recommended approaches is adding them to the Script Properties key-value store available under the Script Properties tab.  Once they're stored, the credentials are accessible in the Apps Script through the `PropertiesService` class.

## Creating the Trigger Project

To use Trigger, you need to setup a Free account.  So far, I've been able to perform all my experiments without needing to upgrade to one of the paid tiers.

Tasks are the most important concept in Trigger.  A Task is a clevery engineered object that get bundled and shipped to infrastructure that can run Docker containers.  Different life-cycle methods are used to control different aspects of a Task's behavior.  Most, if not all, of the custom logic describing a workflow is added to the asynchronous `run` life-cycle method.          

Individual Tasks gets attached to Projects under Organizations.  Organizations and Projects must be created in the Trigger dashboard before tasks can be attached.  You can create as many Organizations, Projects, and Tasks as you want.  Even under the free account!  

From the perspective of an automation freelancer or agency, this naming convention makes sense.  You'll likely have multiple projects with multiple clients (aka Organizations) running at the same time.  


I needed to create a task inside my DigitalOcean Function.  To do that, I navigated to the `skills-to-careers` folder and typed

```sh
npx trigger.dev@latest init
```

in the terminal.  This starts up a dialog that guides you through the project creation process: 

1. Asks you to select a project you created in the Dashboard. 
2. Installs the SDK and any other npm dependencies.
3. Creates a `trigger` folder with a sample task.
4. Creates a configuration file. 


### Adding Credentials

Each project has an independent set of environment variables that are accessible from the "Environment variables" tab.  This is where you want to store API keys, authorization credentials, and any other data you don't want to necessarily reveal in your task source code.  From inside your task, you access them like you would any other environment variable, with `process.env[NAME_OF_VAR]`.  

I added my OpenAI API key and the Base64 encoding of my Google Service Account JWT to my project's environment variables.  Part of the task source code converts the Base64 encoding back to JSON for subsequent authorization.  This idea came straight from the Trigger documentation.      


## Deployment and Finishing Up

1. Deploy trigger
2. Add TRIGGER_SECRET_KEY
3. Deploy DigitalOcean
4. Add URL to Google Apps Script.  

## Conclusion