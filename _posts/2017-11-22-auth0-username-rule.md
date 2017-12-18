---
title:  "Creating an Auth0 rule to attach username to your id token"
date:   2017-11-22 10:00:00
categories: auth0
---

It's no secret that I use [Auth0](http://auth0.com) for nearly all of my projects. Generally, I'll just use the base offering, take the token it gives me, and move on. However, this time, I wanted to actually try to more fully utilize some of the features Auth0 provides. I didn't want to keep having to do a back-and-forth "sign up" flow, where I give my server a token, the server validates it, then checks if I've signed up before. If not, it would send a message back saying "hey, you need a username" and that dance would continue until the user is fully registered. 



This time, I'm going to use the username provided by Auth0, and show you how to add it to the `id_token` payload so you don't have to make an extra API call to get it.

It's pretty simple. In your Auth0 tenant, head over to Rules. Hit the big "Create Rule" button, and select the blank template.

The code for this rule is as follows:

```
function (user, context, callback) {
  context.idToken['http://yournamespace/username'] = user.username;
  callback(null, user, context);
}
```

You have to namespace your claim to add it to the `id_token` unfortunately, but on the positive side the namespace is never "called" (no http requests, or anything of the sort), so it can be whatever you want. Using your project name as a namespace probably isn't a bad idea. You can find more about Auth0 rules [here](https://auth0.com/docs/rules/current).

Now, when you get a token from Auth0, you can head over to [jwt.io](http://jwt.io) and paste it in - you'll see the payload has changed slightly:

```
{
  "http://yournamespace/username": "test1",
  "iss": "https://multinc-role.auth0.com/",
  "sub": "auth0|5a14eca678914308349b8088",
  "aud": "E27Q5Hzsqzv0TkjlydTMgoVIUfQh_T-u",
  "iat": 1511358908,
  "exp": 1511394908,
  "at_hash": "nqMikpWrdjLUunIHJiaqqg",
  "nonce": "DBSxH~.cLzysAi94.pb63b2AsAFK2vOk"
}
```

