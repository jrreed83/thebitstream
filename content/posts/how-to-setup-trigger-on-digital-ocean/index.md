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
marketing in the process.  One of the trends that came up consistently was using generative AI with 
*no-code automation tools*.  I know what generative AI is, but I've never come across no-code automation tools.

## What is a No-Code Automation Tool

No-code automation tools let you build software systems by graphically combining prebuilt "integrations" without programming.  Obviously this is 
great for someone with a product idea but doesn't have the development experience to build it.    

The 3 most popular no-code platforms on the market today are  

* [make.com](https://www.make.com/en)
* [zapier](https://zapier.com/)
* [n8n](https://n8n.io/)

As an experiment, I built a simple make.com "scenario" that uses generative AI to convert old written content into short blog post drafts and emails them on for review.  Everything is managed through a single Google Sheet.

![Make.com automation](./figures/make_automation.png)

Implementing this scenario was very easy.  The most tedious part was setting up the credentials and authorization.  But this isn't make.com's fault; that's always a pain.

I have nothing bad to say about make.com.  It's a great tool, it's just not the right tool for me.  I prefer programming over navigating a user interface, no matter how great it is.  Programming just gives you more power, flexibility, and independence.  Becoming dependent on a tool is a dangerous thing for an engineer.  In my opinion it's best to do things from a "first-principles" approach as much as possible.  Then you can jump around and use whatever tool you want.                 

So I started looking into alternatives.  I was planning on giving n8n a try because is has a reputation for being geared toward software developers.  But then I came across [Trigger.dev](https://trigger.dev/).    

## Introduction to Trigger.dev 

Trigger.dev is definitely not a no-code, or even low-code, tool.  Instead of dragging, dropping, and connecting blocks on a canvas, you build workflows by 
writing normal typescript (or javascript) code.  Unlike other automation tools, it doesn't include any prebuilt integrations that connect to APIs.  If you want to connect to an API, you need to add the dependencies and write the code yourself.  It doesn't get more "first-principles" than that.

So if I have to write all the code myself, what's the point of using Trigger.dev?  


They focus on optimizing the deployment.    

![trigger.dev home](./figures/trigger-dot-dev-home-annotated.png)


Besides testing with their dashboard, there's no builtin way to trigger a task.  You need to set up a webservice capable of triggering the Trigger.dev task when it receives the proper request.  They have a few examples on their website that show how to set this up 

*
*

and serverless function providers


## What we're going to Build

We're going to build a prototype system that helps teachers give career advice to students based on information stored in a Google Sheet. 

![spreadsheet](./figures/spreadsheet.png)

You start by putting the student's name, age, and skills/interestes in an open row.  When you're ready to get career advice, change the "Trigger" dropdown from "Waiting" to "Get Jobs".  A few seconds later, you'll get a list of possible careers for the student to consider.        

I'm not claiming this is production ready or anything.  It can be elaborated and definitely spruced up.  My main objective was to figure out how to get the necessary parts and pieces to work together.

### System Architecture

If we used make.com for this workflow, we'd only need to set up a few credentials, install the "Make for Google Sheets" extension, and connect up a few integrations.  Easy-peasy!

![make.com extension](./figures/make-dot-com-extension.png)

With Trigger.dev, we need to figure out the details ourselves including 

1. how to trigger an event from a Google Sheet,
2. how to send data from a Google Sheet to another webservice,
3. how to interact with the OpenAI API,
4. how to programatically update the Google Sheet 

After a lot of research and experimentation, here's what I came up with:

![carrer-advice-workflow](./figures/career-advice-flow.png)
  
When the dropdown in the "Trigger" column goes from "Waiting" to "Get Jobs", a Google Apps Script is triggered.  It
takes the data from that row and sends it to a Digital Ocean Serverless Function that's waiting for requests at a URL.  The only 
thing the serverless function does is trigger a Trigger.dev task with the data from the Google Sheet.   

## Google Apps Scripts

## Digital Ocean Serverless Functions


They don't specifically cover Digital Ocean's serverless functions, so I decided to try that.  I like to keep things interesting.    

Before doing anything, create a Digital Ocean account.  Once that's out of the way, we can start setting up our serverless function using 
Digitial Ocean's command line interface - **doctl** .  

We could do a lot of the heavy lifting using Digital Ocean's control panel, but I prefer operating from the command line as much as possible.  Here are the instructions for getting doctl setup.  I tried to keep them short, and added help comments along the way.  If you want more details, each step starts with a link to Digital Ocean's documentation. 

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
