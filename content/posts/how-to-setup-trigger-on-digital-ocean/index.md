---
title: Workflow Automation with Trigger.dev
author: Joey Reed
date: 2024-12-20
draft: true
summary:     
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

Even with no programming experience, people are able to build amazingly sophisticated software systems that have a major impact on their business's bottom line. 

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

* Instead of connecting blocks on a canvas, you implement workflows with their npm package and normal Typescript or Javascript.  
* Integrations aren't included.  Connecting to an API requires adding appropriate npm packages and writing code.  Just like you would in any Node appication. 
* Webhooks aren't included.   Workflows can be triggered from the dashboard or any modern Javascript or Node project.
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

![carrer-advice-workflow](./figures/career-advice-flow.png)
  
Changing the dropdown state from "Waiting" to "Get Jobs" triggers a Google Apps Script.  The script packages up some information about the sheet and the edited row,  and sends it to a DigitalOcean Function sitting at a particular URL.  The DigitalOcean Function does almost no work.  It's only purpose is to trigger a deployed Trigger task with data previously packaged up by the Apps Script.  Think of it as a webhook.  Inside the task, the payload is destructured and integrated into an OpenAI chat prompt.  Once the model's response is returned, the task uses the Google Sheets API to update the spreadsheet with AI-generated career advice.

Yes, this workflow is simple enough that the whole thing could be be put in one DigitalOcean Function.  But if I want to extend the capabilities of this system later on, I'd rather get the right tools in place now.     

## Google Apps Scripts

Google Apps Scripts let you interact with Google Workspace products, like Google Sheets and Google Documents, in Javascript.  To attach one to to Google Sheet, open up a Google Sheet, click the "Extensions" Tab in the top toolbar, and select "Apps Script".  This opens up an editor that lets you implement and test your "container-bound" script, setup triggers, and add credentials.           

I wanted the Google Apps Script to

1. Retrieve information about the sheet
2. Forward the sheet information to an API

whenever the "Trigger" column was edited.  To achieve this, I created an "installable trigger" by attaching the "On Edit" event to my primary function.  

![apps-script-trigger-editor](./figures/apps-script-trigger-editor.png)

This causes the function to execute anytime the spreadsheet is edited.  I added a little extra logic to avoid hitting the API when irrelevant cells were edited.   

There are a few different ways to add credentials to an Apps Script without revealing them in the source code.  I decided to use the Script Properties construct found in "Project Settings".  Eventually, this will hold the URL and authorization token for my DigitalOcean Function.  Once saved, they can be accessed in the Apps Script in much the same way you access variabled in a `.env` file. 


## Configuring a DigitalOcean Function Project

The Trigger website has several examples showing how to trigger tasks from other applications.  They don't talk about DigitalOcean's serverless function product, so I decided to try it out.  Maybe someone using DigitalOcean will find it useful.  

I set everything up with doctl, DigitalOcean's command line tool.  Instructions, including links to the official documentation will be on my GitHub repository.

Serverless function projects intialized with doctl generate a `project.yml` configuration file and a `packages/` directory.  The `packages/` directory contains everything for one "package" and one serverless function.  If you want more packages and/or more functions, just add more directories.  Chances are, you'll want to change the default package and function names.  No problem, just make sure that the new directory names match the package and function names in the `package.yml` file.  Otherwise your build will fail.  

Here's the `project.yml` file for my project:

![DigitalOcean project.yaml](./figures/digital-ocean-yaml.png)

It contains one package called `career-package` and one function in that package called `skills-to-careers`.  When I started working
on this, DigitalOcean only supported Node versions 14 and 18.  To minimize compatibility issues, I went with 18. 

To be on the safe side, I maxed out the memory the function can access (1GB) and increased the timeout to 5 minutes.  Allocating this much memory to a function that does almost no work might seem a bit extreme.  For whatever reason though, during testing, executions occasionally failed because the function ran out of memory.  At least that's what the logs said.  Bumping it up fixed the issue.                        

I also added a place holder for my Trigger credentials in the form of a "template variable".  That way I can add `project.yml` to my git repository without revealing sensitive information.  To finish this up, I created a `.env` file at the top-level of the project and added 

```sh
TRIGGER_SECRET_KEY=<add your secret key here>
```
Just make sure that you don't add `.env` to version control!  I'll discuss the `TRIGGER_SECRET_KEY` in the next section.

DigitalOcean Functions use file-based routing, which means that the API route layout and the directory layout match.  Parameters submitted to a function's route are bundled with a bunch of http data, and passed as an object to the specified entry-point.  By default, the entry-point is a function named `main`.  If you need to change the entry-point for some reason, modify the function's `main` value in the `project.yml`.  I'm happy with `main` though.  The build process figures out which Javascript file contains the entry-point.  Moving forward, I'll call this the main function.   

The `skills-to-careers` function will eventually need a few dependencies.  In preparation,  I turned the `skills-to-career` directory into an npm package by running `npm init -y`.  Because I don't like to mix source code with configuration files, I created a `src/` folder for all my Javascript files.  I moved the main function under `src/`, and made it the entry point for my npm package by adding it's relative path to the `main` section in `package.json`. 

To be clear, there are two entry points to be aware of: the DigitalOcean Function's entry point, and the npm package entry point.  You'll get a runtime error if either of them are set incorrectly.

## Adding Trigger

To use Trigger, you need to setup a Free account.  So far, I've been able to perform all my experiments without needing to upgrade to one of the paid tiers.

The most important concept in Trigger.dev is the task, which is ultimately a special asynchronous Javascript function.  Every task gets attached to an "Project" belonging to an "Organization".  Organizations and Projects must be created in the Trigger dashboard before tasks can be attached.



I needed to create a task inside my DigitalOcean Function.  To do that, I navigated to the `skills-to-careers` folder and typed

```sh
npx trigger.dev@latest init --javascript
```

in the terminal.  This starts up a dialog that guides you through the project creation process: 

1. Asks you to select a project you created in the Dashboard. 
2. Installs the SDK and any other npm dependencies.
3. Creates a `trigger` folder with a sample task.
4. Creates a configuration file. 


### Why Javascript and not Typescript?

If you poke around the Trigger.dev docs, you'll get the impression that there's a strong preference for Typescript over Javascript.  Unfortunately, getting this to work with DigitalOcean Function's [48MB upload limit](https://docs.digitalocean.com/products/functions/reference/build-process/#control-which-build-artifacts-are-installed) required an ugly hack.  

The problem stems from how the DigitalOcean build process works.  According to the documentation, if a function folder has a `package.json` file with a `build` script, then `npm install` and `npm run build` runs.  On the other hand, if you don't have a `build` script, then `npm install --production` runs.  The key difference is

* `npm install` adds production and development dependencies to `node_modules`.       
* `npm install --production` doesn't add development dependencies to development dependencies.

Unless I'm missing something, I need a `build` script to convert Typescript files to Javascript.  This has the unfortunate consequence of blowing past the 48MB upload size constraint.     

My work around was manually switching between two `package.json` files; one for the DigitalOcean Function deployment, and the other for the Trigger deployment.    
The `package.json` for the Trigger deployment was the original one with all the dependencies.  For the DigitalOcean deployment, I removed all Trigger-specific dependencies from the `package.json`.  This did the trick, but I don't like this solution.  

Maybe I'll come up with something better, but until then, I'll stick with Javascript.  The workflow is so small, that I don't think Typescript is that beneficial anyway.    

### Adding Dependencies 

I added the OpenAI API and Google APIs Client as development dependencies, even though they are production dependencies as far as Trigger is concerned.  Why did I do this?  With Trigger, there isn't a difference between production and development dependencies.  They're all bundled together by default.  However, as the previous section explains, they won't be included in the DigitalOcean build.  And why should they, they aren't used in the main function at all.

### Adding Credentials

Each project has an independent set of environment variables that are accessible from the "Environment variables" tab.  This is where you want to store API keys, authorization credentials, and any other data you don't want to necessarily reveal in your task source code.  From inside your task, you access them like you would any other environment variable, with `process.env[NAME_OF_VAR]`.  

I added my OpenAI API key and the Base64 encoding of my Google Service Account JWT to my project's environment variables.  Part of the task source code converts the Base64 encoding back to JSON for subsequent authorization.  This idea came straight from the Trigger documentation.      


## Deployment and Finishing Up

1. Deploy trigger
2. Add TRIGGER_SECRET_KEY
3. Deploy DigitalOcean
4. Add URL to Google Apps Script.  

## Conclusion