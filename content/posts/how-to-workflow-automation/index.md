---
title: Workflow Automation with Trigger.dev
author: Joey Reed
date: 2024-12-20
draft: true
summary:  Improve Business Efficiency
tags: ["automation"]
ShowToc: true
---

How can small businesses leverage modern technology platforms to boost productivity and amplify it's.  

I started investigating this question a couple months ago.  One of the trends that came up consistently in my research was the use of generative AI and "workflow automation tools" to automate critical business tasks.  Here's a list of some use cases taken straight from the landing page for one of the most
popular automation products, [zapier](https://zapier.com):

* Sales handoffs
* Marketing campagins
* Data management
* IT helpdesk
* Lead Management
* Revops 

Even with no programming experience, people are able to build amazingly sophisticated and useful software systems that have a major impact on their business's bottom line. 

Over the course of my career, I've used many different software products.  Not one has helped me with a business task.  This started sounding pretty appealing.  I want to spend my time doing what I get paid to do - engineering.  If these tools can help me codify and reduce the time I spend on some business process, then it's worth giving them a try.

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

In Make, workflows are called "scenarios".  For my first foray into automation, I built a simple scenario that uses an OpenAI assitant to convert old written content into short blog post drafts, and emails them on for review.  Everything is managed through a single Google Sheet.  Here's what the scenario looks like in the Make dashboard:

![Make.com automation](./figures/make_automation.png)

Implementing this was very easy.  The most tedious part was setting up my Google credentials.  Even that wasn't hard, just unfamiliar.  You end up having to access your Google Cloud account, which I hadn't done before.

### Why No-Code is not my Ideal Choice 

I have nothing bad to say about Make.  It's a fantastic service, it's just not the right tool for me.  I prefer programming over working a user interface, no matter how great it is.  Programming takes more work, but in return you get more power, more flexibility, and more independence.  

This last point is particulary important to me.  As an electrical engineer,  it's very easy to become a victim of "vendor lock-in".  Adopting software tools is unavoidable, but you don't want to become so dependent on them that you forget the fundamentals.  If you can afford the time, it's best to strive for a first-principles as much as possible.  Then you can jump around and use whatever tool you want.                     

Make simply doesn't conform to my "first-principles" philosophy, so I started looking into alternatives.  I was planning on giving n8n a try for several reasons, but then I came across [Trigger.dev](https://trigger.dev/).    

## Introduction to Trigger

Trigger is much different than other popular automation solutions.  

* Instead of connecting blocks on a canvas, workflows are implemented by adding normal Typescript/Javascript to their carefully engineered "task" object.
* Integrations aren't included.  If you need to communicate with an API,  you need to add the appropriate dependencies and write the code yourself.   
* It's implemented as a serverless architecture.

If you aren't getting drag-and-drop functionality, and you're aren't getting built-in integrations, what are you getting?    

![trigger.dev home](./figures/trigger-dot-dev-home-annotated.png)

Like their website suggests, you're getting a deployment solution where you

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
  
Changing the state of a studen't dropdown from "Waiting" to "Get Jobs" triggers a Google Apps Script.  The script packages up some details about the sheet including the

* Sheet ID
* Sheet name
* Row and column the career advice will go
* Student details

and posts it to the task's API URL.  When the running task receives the payload, it extracts the student's skills from the payload and adds it to an OpenAI chat prompt.  When the OpenAI API returns with a valid response, the task uses the Google Sheets API to add career advice to the cell described by the Apps Script original POST request. 
  
## Setting up the Google Apps Script

Google Apps Scripts let you interact with Google Workspace products, like Google Sheets and Google Documents, in Javascript.  To attach one to a Google Sheet, open up a Google Sheet, click the "Extensions" Tab in the top toolbar, and select "Apps Script".  This opens up an editor that lets you implement and test your "container-bound" script, setup triggers, and add credentials.           

I wanted the Google Apps Script to

1. Retrieve information about the sheet
2. POST the information to to my Trigger tasks's API URL

whenever the "Trigger" column was edited.  Because the Trigger API requires an authorization header, I used an "installable trigger".  With an installable trigger you use the editor to manually attach an event to the function you want to run, when that event occurs.  It's just like a callback.

![apps-script-trigger-editor](./figures/apps-script-trigger-editor.png)

The "On Edit" event is the right choice for this application.  To avoid hitting the API when irrelevant cells were modified, I added a little extra logic.

I also needed a way to store my Trigger credentials in the Apps Script project without revealing them in the source code.  One of the recommended approaches is adding them to the Script Properties key-value store available under the Script Properties tab.  Once they're stored, the credentials are accessible in the Apps Script through the `PropertiesService` class.

## Creating a Trigger Project

Start by setting up a free Trigger account. Chances are, you won't need one of the paid tiers right away, unless you're building something really ambitious.  

The Free tier gives you access to as many tasks, projects, and organizations as you want.  It's pretty clear what a task is.  Organizations and projects are account mechanisms, added through the Dashboard, to keep track of your tasks.  They help keep your dashboard clean and tidy.  Every Trigger codebase must be attached to a project created under an organization.  

### Creating an Orgaization and Project 

Before initializing my Trigger codebase, I created an organization called "Career Assistant" and a project called "google-sheets-workflow".  You can delete organizations and rename project names, but you can't delete projects.      

### Initializing the Codebase 

First, I initialized an npm package and adding the OpenAI API and Google APIs Client as dependencies:  

```sh
mkdir my-trigger-project
cd my-trigger-project
npm init -y
npm install openai googleapis
```  

Once the dependencies were installed, I added Trigger by running the CLI initialization command:

```sh
npx trigger.dev@latest init 
```

No need to install their CLI tool - `npx` handles everything for you.  

This starts up a dialog that helps scaffold out a project:  

1. Asks you which Trigger project you want to attach your future tasks to.

2. Installs the SDK and any other npm dependencies.

3. [Optional] Creates `src/trigger` directory with sample task

4. Creates a `trigger.config.ts` configuration file. 

The configuration contains the autogenerated project ID associated with the project and organization previously set up.   

### Adding Credentials

Each project has its set of environment variables for storing API credentials and other sensitive data you don't want to reveal in source code.  You add them under the "Environment variables" tab as key/value pairs.  They're accessed in your source code like environment variables in any other Node project, with `process.env`.  

I needed to add two sets of credentials:

* OpenAI API key 
* Google Service Account JSON Web Token (JWT)

Creating a Google Service Account was only necessary because I wanted to update the Google Sheet through the API.  If I only wanted to read data from the Sheet, I could have gotten away with setting up API Key.  

Unlike the OpenAI API Key, the Google JWT is a JSON file you download after creating the service account.  It wasn't immediately obvious how to store this as an environment variable  Fortunately, Trigger's website has a straight forward [solution](https://trigger.dev/docs/deploy-environment-variables#using-google-credential-json-files).  They recommend adding the Base64 encoding of the JWT as your Google credential.  Then, in your task code, decode the Base64 encoding back to a Javascript object and pass it to the function responsbile for authorization. 

### Organizing Tasks 

My workflow was simple enough that I could have implemented it in a single task.  I thought about it, and decided that it might be better if I split the workload into 2 separate tasks (an OpenAI task and a Google Sheets task) so I could test each task individually.  So that's what I did.  Here's the "root" task that
encapsulates my `openai_task` and `google_sheets_task`:  

```js
export const career_advice_task = task({
  id: "career-advice",

  run: async function (payload: any, { ctx }) {

    // Just Grab the response
    let chatResponse = await openai_task.triggerAndWait( payload ).unwrap();
    
    console.log(`career_advice_task: ${chatResponse}`);

    await google_sheets_task.triggerAndWait({
      ...payload,
      chatResponse
    });

    return {
      message: "Done With Career Advice"
    }
  }
});
```

The `.triggerAndWait` method invoked on both subtasks executes them asynchronously and returns an object containing the output of the task, along with 
some extra metadata.  Adding `.unwrap()`, strips away the metadata and only keeps the task's return value.  In this case, I'm keeping OpenAI's chat completion, because that's what the OpenAI task returns and it's the missing piece I needed to send back to the Google Sheet.  The root task ends after the Google Sheets task successfully updates the Google Sheet.  I'm including the original payload in this task call because it includes information about what spreadsheet (sheet Id), sheet (sheet name), and cell the career advice content should go. 


### Deployment


## Deployment

1. Deploy trigger
2. Add TRIGGER_SECRET_KEY
3. Deploy DigitalOcean
4. Add URL to Google Apps Script.  

## Conclusion

This post was was an introduction to the world of workflow automation.  I   