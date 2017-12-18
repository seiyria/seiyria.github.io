---
title:  "Moving To Scaleway (From Any NodeJS PaaS) "
date:   2017-10-12 15:00:00
categories: server, dokku
---

Or, really, any PaaS, but my examples will use JS.

There are a lot of NodeJS PaaS' that I've worked with previously - I really wanted to take the headache out of managing my own infrastructure. They provide headaches of their own, though. What if:

* For some reason, you start getting 502s on all of your websocket connections for no reason?
* Your deploys get stuck?
* Anything that's out of your control happens?



Sure, you can contact support and hope it gets fixed. And to their credit, some of these providers have fantastic and nearly immediate support. But after you put up with these issues periodically for a while, you get fed up. I got fed up.

A few months ago, someone put a bug in my ear for [Scaleway](https://www.scaleway.com/). I dismissed it because I didn't want to manage my own infrastructure. Nevertheless, I kept my ears open at projects like [Dokku](https://github.com/dokku/dokku) for when things would inevitably get out of control. Today, I decided to take a leap and finally move away from my provider and host my project myself. It only took an hour.

Before I talk about how I did it, let me give you a table comparing specs that shows why Scaleway was such a good choice (I wanted LetsEncrypt support, and MongoDB was a bonus):

Provider | Cost (per month) | CPU | RAM | LetsEncrypt | Included MongoDB
-------- | ---- | --- | --- | --- | ---
[Scaleway](https://www.scaleway.com/pricing/) | €2.99 | 2 CPU | 2GB RAM | no | no
Scaleway | €11.99 | 4 CPU | 8GB RAM | no | no
[Heroku](https://www.heroku.com/pricing) | $7 / Dyno (Hobby) | ??? | 512MB RAM | no | no
Heroku | $25 / Dyno (Standard) | ??? | 512MB RAM | no | no
[Clever Cloud](https://www.clever-cloud.com/pricing) | €4.50 | 1 CPU | 256MB RAM | no | no
Clever Cloud | €14.40 | 1 CPU | 1GB RAM | no | no
[evennode](https://www.evennode.com/pricing) | €6 | ??? | 512MB RAM | yes | yes
evennode | €13 | ??? | 768MB RAM | yes | yes

I mean, from this, I was willing to accept lower performance to have an included MongoDB and have LetsEncrypt taken care of. It made sense in my mind. However, looking strictly at the numbers, Scaleways lowest tier provides an immense amount of value. 

So, lets talk about how easy it was to get my project off of a provider, and onto Dokku.

After purchasing a Scaleway server, you can create a server and spec it out however you want. For me, the defaults were more than sufficient, and I ended up going with a tier inbetween the one I posted above (4 cores, 4GB RAM), so I named my server and within a minute or two it was ready. You'll want to be familiar with the command line, and ideally have an SSH key ready to give to Scaleway so you can actually get into your server, too.

Once you're into your server, it's time to set up Dokku. It's [two commands](http://dokku.viewdocs.io/dokku/): 

```
wget https://raw.githubusercontent.com/dokku/dokku/v0.10.5/bootstrap.sh
sudo DOKKU_TAG=v0.10.5 bash bootstrap.sh
```

Easy peasy. After it's done, head over to the IP for your server and click confirm to finish setup.

Next, lets add your project. Mine is called `landoftherair` and I have a server for it at `server.rair.land`, so keep that in mind when you see these commands:

```
dokku apps:create landoftherair
dokku config:set landoftherair MONGODB_URI mongodb://myserver.mlab.com # Hosted on mLab
dokku config:set landoftherair ... # the rest of your environment variables
```

Now lets get the app running. What I did next was went into my Google Domains DNS panel and created an `A` record pointing to my server IP for `server` (this ultimately makes `server.rair.land` point to my scaleway server).

Next, I went into my project and created a dokku remote, pushed to it, and let it deploy the app:

```
git remote add dokku dokku@server.rair.land:landoftherair
git push dokku master
```

Afterwards, I set up LetsEncrypt with a few more commands:
```
sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
dokku config:set --no-restart landoftherair DOKKU_LETSENCRYPT_EMAIL=kyle@seiyria.com
dokku domains:add landoftherair server.rair.land
dokku letsencrypt landoftherair
dokku letsencrypt:cron-job --add
```

And that's it. Now my project is running on a custom server running Dokku, with a renewable LetsEncrypt SSL cert. 

