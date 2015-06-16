---
title:  "Code Quality and You"
date:   2015-05-04 14:15:00
categories: javascript, quality
---

Welcome! Today I'd like to talk about another subject which can't be emphasized enough: **Code Quality**. This entails a lot of tools and patterns that ultimately come together to make your game more solid and programmer friendly. Even if you're working alone on a project, these tools can save you some precious debugging time by pointing out simple errors, if not more complex ones. I'll be using my [current project, c](https://github.com/seiyria/c) as an example where possible.



A few notes before we get started:

* Some of the following tools are specific to the JavaScript ecosystem.
* Some of the following tools are only free for open source projects, so bear in mind that you might be missing out on the awesome! 

## Style Checking
Some of the easiest tools you can set up for your project are [JSHint](http://jshint.com/about/) and [JSCS](http://jscs.info/). These tools provide _basic_ static analysis capabilities (basically, checking code correctness at compile-time instead of runtime) and are highly customizable. JSCS even provides a [`preset`](http://jscs.info/overview.html#presets) option to allow you to better adopt the habits of the pros developing more widely-used software.

### JSHint
JSHint provides some static analysis capabilities, and is more geared towards semantics instead of style (in recent releases). Some things it can do for you:
* Warn when you have an undefined variable that you're trying to use
* Warn when you're not using a variable
* Enforce quotation standards (comparing `'` and `"`)
* Enforce usage of curly braces where applicable (not using them was what caused the [Apple SSL security exploit](https://nakedsecurity.sophos.com/2014/02/24/anatomy-of-a-goto-fail-apples-ssl-bug-explained-plus-an-unofficial-patch/))
* Enforce strict type comparisons

Each of these things are very good, but everyone has their own opinions. The take-away here is that you should be using something to keep your codebase consistent and free of lint!

An easy way to integrate JSHint into your project is to use it with [Gulp](https://github.com/spalger/gulp-jshint) or [Grunt](https://github.com/gruntjs/grunt-contrib-jshint). The added bonus here is that mistakes will fail a build if you use it with Continuous Integration (to be discussed further down in the article). If you don't want to specify any options, JSHint will use sensible defaults.

If you're interested, you can see [my `.jshintrc` here](https://github.com/seiyria/c/blob/master/.jshintrc).

*note: I strongly prefer JSHint to [JSLint](http://www.jslint.com/) because JSLint is far too pedantic, even for me. It's another valid tool you could use, however*

### JSCS
JSCS is a tool dedicated to enforcing very specific style constraints. This can prevent you from having a code standards guide, as it is simply enforced by the tool. Some might consider this pedantic, but I think it's a great tool to keep code consistent without much effort, as errors simply fail the build!

JSCS has a [very in-depth list of options](http://jscs.info/rules.html), and the best part is that they all have inverses. If you want to require semi-colons, you can tell it that. If you want to enforce a no-semi-colon rule, you can do that too. Here are some of the styles I use JSCS for:
* No empty code blocks
* Require `var func = function(){}` instead of `function func()`
* Require `else` to be on the same line as the closing `}` for an `if` statement
* Require dot notation for accessing variables (`obj.var` instead of `obj['var']`)
* Require spaces after arguments / parameters in a function call, and in a `for` loop
* Disallow trailing commas in arrays and objects (this actually [breaks old versions of Internet Explorer](https://hirtopolis.wordpress.com/2011/02/11/jslint-and-the-trailing-comma-of-death/))

The easiest way to integrate this into your workflow is to use [Gulp](https://github.com/jscs-dev/gulp-jscs) or [Grunt](https://github.com/jscs-dev/grunt-jscs). Like JSHint, a failure will fail your build.

The best part is that if your code doesn't match, it can fix it for you! If you specify the `fix` option, you'll never be warned of these problems, instead they'll just be fixed for you.

If you're interested, you can see [my `.jscsrc` here](https://github.com/seiyria/c/blob/master/.jscsrc).

## Dependency Checking
Dependency checking is a simple way to ensure that your dependencies are up to date. If all of your dependencies follow [Semantic Versioning](http://semver.org/), this system becomes quite useful to you. Currently, I use [Gemnasium](https://gemnasium.com/seiyria/c) and [bitHound](https://www.bithound.io/github/seiyria/c/master/dependencies/npm) to watch dependencies for c. Gemnasium *only* watches dependencies, but it's still very nice to know when something becomes outdated. I'll talk more about bitHound later, as it does more than dependency version checking.

Gemnasium is (I think) free to use for open source projects, you just have to sign in with your GitHub account.

## Continuous Integration
Continuous Integration, or CI, is a tool you would typically use to verify the integrity of individual commits, pull requests, or builds. I'll cover a few tools that can do this for you, and they're both free for open source projects!

### Travis CI
For GitHub, if you're using [Travis CI](https://travis-ci.org/), you get some very nice integration:

![Travis CI](http://i.imgur.com/2ZIFeAk.png)

The nice thing about CI is that it shows [historical data](https://travis-ci.org/seiyria/c/builds), so you can see every build that ever occurred for your repository. Travis is *very* easy to set up. For example, here's [my `.travis.yml` file](https://github.com/seiyria/c/blob/master/.travis.yml):

{% highlight yaml %}
language: node_js
node_js:
  - "0.12"
  - "iojs"

before_script:
  - npm install -g gulp
{% endhighlight %}

Simply: I build against the latest version of node and iojs, and I have to make sure to install gulp before running any tests or they won't even run. By default, Travis runs the command `npm test`. Of course, without a system like gulp in place, Travis would not be that useful.

### Hound CI
[Hound CI](https://houndci.com) is another form of CI, and it's a bit more opinionated than Travis. You can use them both in conjunction, too. Hound will be much more in your face, and only triggers on PRs. The coolest part is that it only looks at the code that was added, so you don't get any unnecessary messages. It has a default JSHint profile that it runs, but you can specify that it [use a specific one, too](https://github.com/seiyria/c/blob/master/.hound.yml#L2). 

In the odd event that your contributor misses both your local JSHint settings (if you have it), and Travis (if you have it), you'll see some [messages like this](https://github.com/seiyria/c/pull/4). This was before I told Hound to use *my* styles, so the messages were superfluous.

Hound requires no extra setup to use. You just have to auth with your GitHub account. 

## Static Analysis
Static Analysis tools provide a bit more insight into your code. You'll see various metrics like lines of code (more = higher maintainability cost), complexity (too much going on = higher maintainability cost) and a few others. I have a few tools hooked up to c right now.

### bitHound
At first glance, bitHound might be a bit overwhelming. It can do quite a bit: analysis, dependency checking, technical debt warnings, etc. It really is quite simple to use though. [Here is my bitHound dashboard for c](https://www.bithound.io/github/seiyria/c/master). The first thing you'll notice is the chart -- that's the overall score, over time. I haven't been able to break past 97 yet, but I'm sure it will get there in time. bitHound is very nice because it shows you all of your over-time metrics in a simple, digestible fashion. Nicely enough, it also lets you change what you want to look at. Here's the full history of c, as told by bitHound:

![bitHound over time](http://i.imgur.com/0l1BEU3.png)

_(the faded items are on different branches)_

Yikes! What happened in the middle there?
Let's take a look at the [commit where my score dipped quite a bit](https://www.bithound.io/github/seiyria/c/commit/24714963dd138375a3ebb805a1d60e24ee2054b7). Since my overall score is an average of all my other file scores, having one file tank is a pretty serious blow to a small project. Specifically, in this commit, my main feature added was saving and loading. [Here's the problem file](https://www.bithound.io/github/seiyria/c/blob/24714963dd138375a3ebb805a1d60e24ee2054b7/src/js/gamestate.js). If you expand, you see that..  wait, that's a pretty simple function, isn't it?

To most, yes, this function is pretty simple, but it also has a glaring amount of repetition and some unnecessary nesting. As far as I can tell, complexity is a combination of: 
* number of lines in a function
* number of branches in a function
* number of assignments in a function
* levels of nesting in a function

And having too many of these makes your function inherently more complex. The fix required [a little bit of refactoring](https://github.com/seiyria/c/commit/665b40e1e347a60a0e6d054e232f042bb377021b), but in the end it took a bunch of loose variables and tied them together in a more sensible way.

bitHound takes very little effort to set up, and has few options (it's another tool you simply have to sign up with GitHub to use). Namely, you can tell it [what to ignore and what to test](https://gist.github.com/dansilivestru/b3f127f9736ab7cb36bb). I [use it to ignore content files, my gulpfile, and files that I didn't write](https://github.com/seiyria/c/blob/master/.bithoundrc).

### Code Climate
Code Climate, at first glance, is [a bit less complex](https://codeclimate.com/github/seiyria/c). All it really does is static analysis, which is fine; it looks at things different from bitHound and there's no harm in having multiple services run the same types of analysis -- each tool does it differently. With a greater variety of tools, you can either use them all in tandem, or figure out which ones you like best and cease using the others.

[Here's my dashboard in Code Climate](https://codeclimate.com/github/seiyria/c). If I had Unit Tests (described more below), I could hook those in here, too. Otherwise, Code Climate seems to pull open source projects daily and analyze them. Nicely enough, you can get a [full file listing at a glance](https://codeclimate.com/github/seiyria/c/code) and see if there are any major issues. There's also a general [issues dashboard](https://codeclimate.com/github/seiyria/c/issues/categories/complexity) where they mention everything they see that could be improved with your codebase.

Overall, this is a bit more of a shallow view in comparison to the previous one, but they're two tools that do similar things, just a tinge of difference between them.

This tool requires very little effort to set up, just sign up with GitHub.

### Codacy
I also checked out [Codacy](https://www.codacy.com/app/seiyria/c/dashboard). I can't talk much about it yet, since it doesn't yet support ECMAScript 6 (ES6). I've added it in the hope that soon it will support ES6 and the analysis will actually help me. [I let them know, and hopefully it will come soon](https://twitter.com/codacy/status/593424520297455616)! If you're not using ES6, though, it should be another valuable tool in the toolbox.

For fun, [here's what they think my problems are](https://www.codacy.com/app/seiyria/c/issues). They just don't like template strings, it seems!

Another simple tool, all you have to do is sign up with GitHub! Although, I don't know the specifics here as I wasn't able to utilize it very well.

## Unit/Integration Testing
Unfortunately not yet present in c, but still a very valuable thing to do. Unit testing adds yet another layer of confidence on top of your code so you can refactor relentlessly and ensure your code still works (provided your code coverage is high enough). You can use tools like [Jasmine](http://jasmine.github.io/), [Mocha](http://mochajs.org/), and [Karma](http://karma-runner.github.io/0.12/index.html) to accomplish these goals. 

Presumably, when something gets introduced into c, this space will be updated.

## Badges
You might be thinking "man, the more services I use, the more I have to check if I want to see progress on my project." Nope! All of the tools above provide badges that you can embed into your README, so any time someone looks at it, they see your project status. On GitHub, this means they show up [right on the front page](https://github.com/seiyria/c). Here are what the badges correspond to:

* bitHound [![bitHound Score](https://www.bithound.io/github/seiyria/c/badges/score.svg)](https://www.bithound.io/github/seiyria/c)
* Codacy [![Codacy Badge](https://www.codacy.com/project/badge/9f26b0ef8b1748e59a737f035bdf52c6)](https://www.codacy.com/app/seiyria/c)
* Code Climate [![Code Climate](https://codeclimate.com/github/seiyria/c/badges/gpa.svg)](https://codeclimate.com/github/seiyria/c)
* Gemnasium [![Dependency Status](https://gemnasium.com/seiyria/c.svg)](https://gemnasium.com/seiyria/c)
* Travis CI [![Build Status](https://travis-ci.org/seiyria/c.svg)](https://travis-ci.org/seiyria/c)

Go forth and make your code better!

