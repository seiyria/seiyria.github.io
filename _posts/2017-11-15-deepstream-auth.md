---
title:  "Setting up a self-hosted Deepstream auth service"
date:   2017-11-15 10:00:00
categories: deepstream
---

Lately, I've been playing with [deepstream.io](https://deepstreamhub.com/open-source/?io)@3.1.1 making a realtime game engine (read: wrapper) on top of it. This has provided me with an innumerable amount of challenges. One of them, today, was trying to set up an authentication service for my self hosted Deepstream instance. Reason being, I'm transitioning away from DSHub, because my needs will not scale with their plans. So, I took the plunge and worked on setting up the basics to get my small project working again.

So, to do that, I needed token auth (at least, something to test with) and anonymous auth.

Deepstream provides identifiers for each client and provider that connects, which you can get via `data.id` after a successful login. For some reason, I was not getting my own back from my auth service. As it turns out, you need to set a `username` AND send the `id` back in the `clientData` section. The  [current docs](https://deepstreamhub.com/tutorials/guides/http-webhook-auth/#set-up-a-simple-http-authentication-server) don't explain this super well. Here is my http service, as a result.

Here is my resulting service, maybe it can help someone else struggling with this problem:

```js
const express = require('express');
const bodyParser = require('body-parser');
const app = express();

const uuid = require('uuid/v4');

app.use(bodyParser.json());

const ONLY_VALID_TOKEN = 'adKTi3fEg99ZeOsgRIaVJr8D9fZq0XNT';

app.post('/local-deepstream-auth', (req, res) => {
  const { authData } = req.body;

  const generatedId = uuid();

  if(authData && authData.token) {
    if (authData.token === ONLY_VALID_TOKEN) {
      res.json({
        username: generatedId,
        clientData: { id: generatedId },
        serverData: { hasAuthority: true }
      });

      return;
    }

    res.status(403).send('Invalid token');
    return;
  }

  res.json({
    username: generatedId,
    clientData: { id: generatedId },
    serverData: {}
  });
});

app.listen(3871, () => console.log('Local Deepstream Auth running on 3871.'));
```
