---
title:  "Common Pitfalls in JS-based Games"
date:   2015-04-23 13:07:00
categories: javascript, incremental
---

Welcome! You might be reading this out of curiosity, or because you want to improve your programming capabilities to stop people from exploiting your JS games. Given that the first thing I do when I open a new incremental is open the terminal and start messing around with your games, I figured it's about time to write something about what I see and how I break your games. Consequently, I'll describe ways you can protect your games from the basic code manipulations I perform. Some might say "you're just ruining the game for yourself!" while I'm going to turn around and say "I don't care" -- that's not the point of this! 

_NB: This will only apply to vanilla JS applications, which I see more commonly. Frameworks like AngularJS and such are out of scope for this post. Advanced techniques such as using a debugger, while slightly more on topic, will also be disregarded for now._

Lets talk about me for a second: I'm [seiyria](http://seiyria.com) and I'm a professional mostly-JavaScript software developer. That is to say, I really like JavaScript -- the ecosystem, the language, and the reach of JavaScript (did you know, there's hardware _powered by JavaScript_? Awesome!). I've been programming for 10+ years now, sometimes I make applications, sometimes I make games -- a little bit of everything.

Okay, enough about me, lets go into the things you'll care about.

First, I'd like to talk about some basic client-side things that can be exploited. Later, I'll get into protecting your server (as much as you can, anyway). It's good to note that your game, if it's client side only, can never be fully protected. You can take measures to do so, but there are more techniques not listed here that can be performed.

### "Clean Code"

Clean code, or non-obfuscated code, is something that makes it very easy to logically see how your code works. This means that you're basically uploading your code as you wrote it with no manipulation. It's also very easily preventable. It won't stop someone who's determined to push through but 50% of the time I'll look at it and go "eh, too much work for me to care."

Essentially, what you should do here is just run your code through [UglifyJS](https://github.com/mishoo/UglifyJS) once before pushing it to a server. There are three benefits: A smaller JS footprint, less HTTP requests (assuming you concatenate them all into one file), and you've performed a small amount of mitigation, yay!

### Public Functions

If all of your functions are public, ie, they look like this:

{% highlight js %}
function myPublicFunction() {
  // perform awesomeness
}
{% endhighlight %}

or this:

{% highlight js %}
var myPublicGame = {
  doSomethingAwesome: function() {
    // awesome level > 9000
  }
};
{% endhighlight %}

Then we have a problem. Lets suppose your "do something awesome" functions are game-critical functions like "level up" or "add resources" -- it's very trivial for me to open the console and go `myPublicFunction()` or `myPublicGame.doSomethingAwesome()`. What's even better, is if your function accepts an argument for how much I get to increase my resources or level by. My favorite.

What can you do to fix this? Two simple ways; the first requires no real planning on your part, just put this around all of your code:

{% highlight js %}
(function() {
  //your code here
})();
{% endhighlight %}

What does that do? Well, it puts all of your code in an isolated scope that can't be accessed globally. This means that all of your code is inaccessible via the terminal, for the most part. The second way requires a bit more thought, but it lends itself better to an application design standpoint. JavaScript has a class-like syntax. I say class-like, because JS does not have traditional classes ([not yet, anyway](https://babeljs.io/docs/learn-es6/#classes)). Lets look at that syntax here:

{% highlight js %}
var MyClass = function() {
  var myPrivateVariable = 4;
  
  var self = this;
  self.myPublicVariable = 10;
};

var myClassInst = new MyClass();

console.log(myClassInst.myPrivateVariable); // => undefined
console.log(myClassInst.myPublicVariable); // => 10
{% endhighlight %}

As you see here, you can fake private variables by limiting the scope they're accessible from. Why does this matter? Well, lets make it a more practical example:

{% highlight js %}
var MyGame = function() {
  var growthRate = 100;
  var myCurrency = 0;
  
  var grow = function() {
    myCurrency += growthRate;
  };
  
  setInterval(grow, 1000); // grow more every second of my life!
};
{% endhighlight %}

If you've been following along, you'll notice that I can no longer go into the console and type `MyGame.growthRate = 10000000000` or `for(var i=0; i<10000; i++) { MyGame.grow(); }` -- it's all private! There are a few more approaches here that could be taken, such as using `RequireJS` or other tools to manage your files, but lets keep it simple for now.

Some games that have exploits like this available:
* [BlackMarket](http://totominc.github.io/blackmarket/) -- `devMode(2)` gives you 100 quintillon money. More succinctly, they expose `money` and `prestige` as global objects, which can be freely manipulated. There's also a `cheats.js` file. If you're using grunt or gulp or some build system, this should be excluded from your distribution build for sure.
* [Meme Clicker](http://sixbytesunder.com/memeclicker/) -- `app.memes = 1000000000000000000`
* [A Dark Room](http://adarkroom.doublespeakgames.com/) -- literally everything is exposed. The game is more complex than "get currency, spend currency" though, so I'll leave this one as an exercise to the reader.
* Many more games; I'm not going to go make a huge list, I'm just providing some small examples.

### LocalStorage modification

Maybe a little overkill, but if your game stores things in localStorage, it's probably vulnerable. Most games I see just store a simple hash object with some data, or store a bunch of keys with data. Suppose you've protected your game via the above measures and now you want to make sure everything is good to go. Lets look at Meme Clickers data ([here is an example](http://hastebin.com/hekobosiwu.cpp)). All I have to do is modify the `memes` attribute to be whatever I want, reload the page, and it'll be peachy - the game won't even know I messed with it; for all it knows, that's a valid state. Similar case with Blackmarket -- [check this out](http://hastebin.com/keselinowo.cpp).

"Alright," you say, "what can I do about that?" Simple. Store a hash of all the data. MD5, SHA-1/2, anything. If you hash all of the data you're saving, store the hash with it, and then load the game, all you have to do is verify the hash upon loading the game. If the hash is invalid, the save is invalid, and should be treated as a fresh start.

### Server Exploits

Okay, so this is what prompted this article. Recently there was a game introduced called [IncrementalGame](http://incrementalgame.com/). It's pretty meta, and it's also backed by a server. That last bit is what makes it a much more fun target than other games, since other people can see what I'm doing, too. [Yesterday, I posted a simple exploit](http://www.reddit.com/r/incremental_games/comments/33fylj/incrementalgamecom_shall_we_play_a_game/cqksb2g) that allowed anyone to massively increase the votes behind any game listed there. I simply dug around in the code until I came across something that looked like it did something, watched my Network tab in my dev tools, and figured out how the game worked. Here are some things to note about having a game with a server:

* Always validate the data coming in. Yesterday, I was able to send negative votes to any set of games simply by using my exploit, but changing the sign on the `10` to be `-10`. In some games, it may make sense to allow positive and negative inputs, but this one is not one of those cases. The resolution here, always validate the data coming in from the client.
* If your API is entirely internal, giving back error messages like "vote size > 20; truncating to 20" just makes it so I know that I can't send a value greater than 20, which means I can send a larger request. The resolution here, don't send error messages that don't need to be sent.
* If your server processes a lot of data, it's much easier to DoS it. In this case, IncrementalGame.com took an array of votes and processed every one of them. Supposing that I put 10000 individual votes into the array being sent to the server, I can make the server choke when it has to process all of that data repeatedly. The resolution here, simplify the data going to your server.
* If your game needs to enforce a rate limit on how much you can interact with it, say, you can only vote 1.3x / second, then you better not be attempting to enforce that on the client. As shown previously, I can simply make an array of 10000 votes and send that to the server, raw, without clicking any buttons on the page. If disabling buttons on the page is the only thing stopping people from spamming the server, that can only end poorly. The resolution here, make sure your server is effectively rate-limiting your players from spamming it too much. **This is not to say that you should only validate on your server, but your server should be authoritative! You should still validate on the client side.**

### Conclusion

In short, it's very easy to make a game that's exploitable. Hopefully the techniques listed above not only help you grow as developers, but make your game and have it played the way you intended.

If not, I'll be there to break it.

_Want me to take a look at your game / app? Send me a message!_
