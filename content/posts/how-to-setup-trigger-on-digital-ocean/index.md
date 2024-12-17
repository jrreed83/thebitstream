---
title: How to use Trigger.dev
author: Joey Reed
date: 2024-12-15
draft: true
summary:     
tags: ["automation"]
ShowToc: true
---

Working at a small company is great, but I don't get much time to develop effective marketing strategies.

I keep reading about how generative AI is transforming the business landscape.  So a few months ago I started investigating how small businesses use this technology to help day-to-day business operations.  

One topic that kept coming up were "no-code" automation tools.  I had never heard of this before.  They give non-programmers the power to conveniently build powerful software systems by combining popular APIs, services, and tools together in a simple drag-and-drop interface.  

The 3 most popular no-code platforms on the market today are  

* [make.com](https://www.make.com/en)
* [zapier](https://zapier.com/)
* [n8n](https://n8n.io/)

So far, I've used make.com to build out a simple content management/generation system with Google Sheets and the OpenAI API.  Overall, the process was pretty easy.  But I like to program, so I looked around for some alternatives.          

##  Trigger.dev

[Trigger.dev](https://trigger.dev/) is a newer automation tool 

### Triggers in Google Sheets

### Digital Ocean

```shell
$ doctl serverless init --language js doctl-trigger-dot-dev
A local functions project directory 'doctl-trigger-dot-dev' was created for you.
You may deploy it by running the command shown on the next line:
  doctl serverless deploy doctl-trigger-dot-dev
```

```shell
$ doctl serverless connect
```

```shell
$ doctl serverless deploy doctl-trigger-dot-dev
Deploying '/home/joeyreed/Documents/GITHUB/doctl-trigger-dot-dev'
  to namespace 'fn-9972eb5c-4690-46d5-822b-5515212c189c'
  on host 'https://faas-sfo3-7872a1dd.doserverless.co'
Deployment status recorded in 'doctl-trigger-dot-dev/.deployed'

Deployed functions ('doctl sls fn get <funcName> --url' for URL):
  - sample/hello
```

```shell
$ doctl serverless function list
Latest Update     Latest Version    Runtime Kind    Function Name
12/16 05:45:31    0.0.1             nodejs:14       sample/hello
```

```sh
$ doctl serverless function invoke sample/hello
{
  "body": "Hello stranger!"
}
```

```sh
doctl-trigger-dot-dev/packages/sample/hello$ npx trigger.dev@latest init
```

```sh
doctl-trigger-dot-dev/packages/sample/hello$ npm install -D vite
```

![digital-ocean](./figures/digital-ocean-function.png)



[node instructions](https://docs.digitalocean.com/products/functions/reference/runtimes/node-js/)


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