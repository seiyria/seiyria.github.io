---
title:  "Making a Reddit Reply Bot"
date:   2019-04-25 15:00:00
categories: javascript, reddit
---

There's a lot of documentation on writing a reddit bot in Python, but I had a lot of trouble finding even basic documentation for Node - even some of the libraries that are listed on reddits official wiki are dead or 5 years old (read: don't support new reddit very well). So, I wanted to write about a simple and common use-case: _replying to a user who tags you_.



### Creating a Reddit Application

First, head on over to [https://www.reddit.com/prefs/apps](https://www.reddit.com/prefs/apps) and hit "create app" - you need to do this so reddit isn't using your personal user account. You should also sign up for a new reddit account for your bot (especially if it can be summoned). Make sure you add your main account and bot account as developers on this application.

When creating an app, you need to fill in the field similarly to this: 

<blockquote class="imgur-embed-pub" lang="en" data-id="vclhFhT"><a href="//imgur.com/vclhFhT"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

Once in, you'll see a screen like this: 

<blockquote class="imgur-embed-pub" lang="en" data-id="IKpfGVt"><a href="//imgur.com/IKpfGVt"></a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

Take note of this, because you'll need this information in just a second.

### Starting the Node Project

For something like this, I found it very easy to use [`snoostorm`](https://www.npmjs.com/package/snoostorm) (a wrapper around [`snoowrap`](https://www.npmjs.com/package/snoowrap)). This library makes it exceptionally simple to get comments as they come in.

You first need to make a `snoowrap` object, then use that to make a `CommentStream` object. To do that, you'll need your reddit bot username, password, application secret, and application id. You must specify a unique user-agent, so call it something like `my-node-js-bot`. So, configure it like so (mine is configured based on the picture above):

```
const Snoowrap = require('snoowrap');
const { CommentStream } = require('snoostorm');

const client = new Snoowrap({
    userAgent: 'my-node-js-bot',
    clientId: 'qR6rJnQ7sEJZDw',
    clientSecret: 'OCoo9pYnlC2K6fxQQxbcIPQ5MA4',
    username: 'myusernamebutactuallybot',
    password: 'mypasswordbutactuallybot'
});
```

With this client object, you can finally start listening for new comments! Head over to [/r/testingground4bots](https://www.reddit.com/r/testingground4bots) and hop into a thread or make your own. Then, add some code to start watching for comments:

```
// pollTime is 10000 because reddit is very strict on posting too frequently
// at first, you'll only be able to post once every 10 minutes, so make sure you get it right!
const comments = new CommentStream(client, { subreddit: 'testingground4bots', limit: 10, pollTime: 10000 });

comments.on('item', (item) => {
    console.log(item);
});
```

Start the bot up, and you'll see a flood of comments in your terminal. You may be wondering why that is - you haven't even seen any new ones come in yet! Well, the `client` will always give you the first X entries (in this case, 10) when you start your bot up, and then it will track from there.

We can fix that pretty easily:

```
// reddits api doesn't use millis
const BOT_START = Date.now() / 1000;
  
const comments = new CommentStream(client, { subreddit: 'testingground4bots', limit: 10, pollTime: 10000 });

comments.on('item', (item) => {
    if(item.created_utc < BOT_START) return;
    
    console.log(item);
});
```

Great, now you only see the newest comments as they come in. Hopefully you have an established enough reddit account to make some posts on this subreddit. If you do so, you'll see your terminal fill with them fairly quickly after posting.

### Making It Interact

So far, you've got a bot and it reads comments - that's a great start! But you want it to interact with your audience, right? So, how about a good ol' hello world? Check it out:

```
const BOT_START = Date.now() / 1000;
  
const comments = new CommentStream(client, { subreddit: 'testingground4bots', limit: 10, pollTime: 10000 });

comments.on('item', (item) => {
    if(item.created_utc < BOT_START) return;
    
    item.reply('hello world!');
});
```

There, any time a comment comes in, the bot will reply to it with "hello world!" Wait... that might talk a bit often, won't it? It might get _just a little bit annoying._ Reddit recommends replying specifically when your bot is mentioned, so, there's a fairly easy way to do that:

```
const BOT_START = Date.now() / 1000;

const canSummon = (msg) => {
    return msg && msg.toLowerCase().includes('/u/myusernamebutactuallybot');
};
  
const comments = new CommentStream(client, { subreddit: 'testingground4bots', limit: 10, pollTime: 10000 });

comments.on('item', (item) => {
    if(item.created_utc < BOT_START) return;
    if(!canSummon(item.body)) return;
    
    item.reply('hello world!');
});
```

There! So, what this does is make sure that the comment your bot finds _actually_ refers to the bot itself. This `canSummon` function will do basic checking to make sure your bot doesn't erroneously spam a bunch of peoples comments. Make a comment now that says `/u/myusernamebutactuallybot` (rather, you should check for _your own bots name_), and you should see a reply shortly afterwards that says "hello world!" in reply. 

That's all you gotta do! 🎉
