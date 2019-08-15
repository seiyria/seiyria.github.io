---
title:  "Setting Up Auto-deploy With Dokku"
date:   2019-08-15 12:00:00
categories: dokku, autodeploy
---

As a continuation to [my previous post](https://seiyria.com/server,/dokku/2017/10/12/dokku-scaleway.html) on setting up a VPS and moving away from a PaaS (like heroku), this small post goes into detail how you can get back one of the greatest features of the average PaaS: autodeploy!

This post will assume you have Dokku and letsencrypt installed already.

So, I was looking to move away from another PaaS that I was one (my last one!) from an old project. A huge pain point of using a private server is that Dokku doesn't have any default auto-deploy support, meaning you have to pre-compile anything before you deploy. Well, every so often I look around and today I stumbled upon [Wharf](https://github.com/palfrey/wharf). What it is, is a simple way to manage Dokku apps on your server with a simplistic UI. It's pretty ugly, but it does the basics of what you need - add/remove env variables, force a deploy, and look at deploy logs.

Lets look at how to set up Wharf. At the time of writing, it's pretty simple: 

Install some pre-requisite dokku plugins:

- install [dokku-redis](https://github.com/dokku/dokku-redis)
- install [dokku-postgres](https://github.com/dokku/dokku-postgres)

Set up the Wharf app:
- create a new app `dokku apps:create wharf`
- `mkdir /var/lib/dokku/data/storage/wharf-ssh/`
- `chown dokku:dokku /var/lib/dokku/data/storage/wharf-ssh/`
- `dokku storage:mount wharf /var/lib/dokku/data/storage/wharf-ssh/:/root/.ssh`
- link redis `dokku redis:create wharf && dokku redis:link wharf wharf`
- link postgres `dokku postgres:create wharf && dokku postgres:link wharf wharf`
- set an admin password `dokku config:set wharf ADMIN_PASSWORD=somesecretpassword`

Deploy Wharf to your server:
- `git clone https://github.com/palfrey/wharf.git`
- `cd wharf`
- `git remote add dokku dokku@my-server.tld:wharf` (replace with your actual server domain/ip)
- `git push dokku`

Once Wharf is deployed, you can go to `http://my-server.tld` (or whatever your server domain/ip is) to log into Wharf for the first time. I'll be using `http://my-server.tld` as an example for the next section as well, so be sure to replace it with your server or ip!

Next, you can set the app to auto-deploy from github: 

* First, you need to set the `GITHUB_URL` env variable in dokku so Wharf knows where to pull your app from: `dokku config:set myapp GITHUB_URL=https://github.com/myuser/myrepo.git`. 
* Set up a `GITHUB_SECRET`. This is used to prevent bad payloads from hihacking your server - you can set it to anything, like so: `dokku config:set myapp GITHUB_SECRET=thisisasupersecret`
* Second, you need to [set up a GitHub webhook](https://developer.github.com/webhooks/creating/#setting-up-a-webhook). You should point the URL to `http://my-server.tld/webhook` (wherever your Wharf instance is, and append `/webhook`) and it needs to be set to `application/json` for the content type. You also need to match the secret to the `GITHUB_SECRET` you chose above.

Now the app will auto deploy! If you don't need to pre-compile your app, this will be sufficient for you. However, if you're using typescript and don't want the overhead of using ts-node in production, you have a little more work to do. So, just a few more steps to go for that:

First, create an [app.json](http://dokku.viewdocs.io/dokku/advanced-usage/deployment-tasks/) in the root of your repository. It should look something like this:

```
{
  "scripts": {
    "dokku": {
      "predeploy": "npm run build:server && npm run setup"
    }
  }
}
```

In my case, `npm run build:server` will call `tsc` and do some other things. `npm run setup` will seed the database and do whatever I need to do. The reason why this is done is so Wharf will compile and then send the updated server code to your Dokku app. 

That should be all you need to do to get auto deploy to Dokku working! 🎉


