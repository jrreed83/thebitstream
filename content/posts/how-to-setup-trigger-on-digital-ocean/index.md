---
title: Workflow Automation with Trigger.dev
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

* DigitalOcean Function: [15 minutes](https://docs.digitalocean.com/products/functions/details/limits/)
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
takes the data from that row and sends it to a DigitalOcean Function that's waiting for requests at a 
particular URL.  The DigitalOcean Function does almost no work.  It's only purpose is to trigger a deployed Trigger.dev 
task with the spreadsheet data.  Think of it as a webhook.  Inside the task, the payload is destructured and integrated into
an OpenAI chat prompt.  Once the model's response is returned, the task uses the Google Sheets API to update the spreadsheet
with the career advice.

Yes, this workflow is simple enough that the whole thing could be put in the DigitalOcean function.  But if I want to 
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

The last thing we need to do is find a way to store credentials without revealing them in the Apps Script.  Once we set it up, we'll need to store the URL and authorization token for our DigitalOcean function.  The simplest way to do this is adding the key and value as a script property, and retreve the value in the script when you needed.  Very similar to creating an `.env` file.  Script properties are accessible under the "Project Settings" tab. 

## DigitalOcean Functions

The Trigger.dev website covers several ways of triggering tasks from web frameworks and different serverless functions.  They don't talk about DigitalOcean's serverless function product, so I decided to try that.  Maybe someone using DigitalOcean will find it useful.  

We'll do everything with doctl, DigitalOcean's command line tool.  Instructions, including links to the official documentation are in my GitHub repository.

When you initialize a serverless function project with doctl you get a `project.yml` configuration file and a `packages/` directory.  The `packages/` directory contains everything you need for one "package" and one serverless function.  If you want more packages and/or more functions, just add more directories.  Chances are, you'll want to change the default package and function names.  No problem, just make sure that the new directory names match the package and function names in the `package.yml` file.  Otherwise your deployment will fail.  

Here's the `project.yml` file for my project:

![DigitalOcean project.yaml](./figures/digital-ocean-yaml.png)

It contains one package called `career-project` and one function in that package called `skills-to-careers`.  When I started working
on this, DigitalOcean only supported node versions 14 and 18.  To minimize compatibility issues, I decided to go with node version 18. 

To be on the safe side, I maxed out the memory the function can use (1GB) and increased the timeout to 5 minutes.  Allocating this much memory to a function that does almost no work might seem a bit extreme.  For whatever reason though, during testing, executions failed because the function ran out of memory.  At least that's what the logs said.  Bumping it up fixed the issue.                        

I also added my Trigger.dev credentials to the function's environment section as a "template variable".  That way I can add `project.yml` to my git repository without sharing API credentials.  To get this to work, open up a `.env` file at the top-level of the project and add 

```sh
TRIGGER_SECRET_KEY=<add your secret key here>
```
Just make sure that you don't add `.env` to version control!  I'll explain how to get your `TRIGGER_SECRET_KEY` in the next section.

DigitalOcean Functions use file-based routing.  This means that a deployed project's API routes match the directory layout.  Parameters submitted to a route are bundled with a bunch of http data, and passed as an object to the specified entry-point.  The default entry-point is called "main".  You can change it to something else by modifyign the function's `main` value in the `project.yml`.  But why would I want to do that; "main" seems like a good name to me.  The build process that runs when you start a deployment figures out which javascript file defines the entry-point.  I'm not sure what would happen if you had multiple javascript files with their own `main` functions.  Probably nothing good.





