---
title:  "Automation 🤝 Mobile Games"
date:   2019-01-28 18:00:00
categories: javascript, automation
---

Sup. Lately I've been playing a mobile game called [Star Ocean: Anamnesis](https://starocean.square-enix-games.com/home/). It's a mobile [gacha game](https://en.wikipedia.org/wiki/Gacha_game). It's the kind of game where you farm a ton of resources to improve your characters and items, which is pretty enjoyable and zen overall. The problem, also, is that these games require a metric fuckton of farming, and therefore, time investment. This isn't inherently bad, but sometimes they can load a ton of it onto you at once, which you're then expected to spread out amongst several weeks.

Pshaw, lets automate this.



So, where do you get started with automating a game? They're pretty dynamic, and you don't really have the ability to hook into most of them. Well, one method I've seen used in other, similar blog posts is read the screen state and building from there. You'll see blog posts [along this line](https://code.tutsplus.com/tutorials/how-to-build-a-python-bot-that-can-play-web-games--active-11117) where they grab the screen, parse the state they care about, and do that every so often (depending on the game / action level / etc). That's what we're going to do.

If you're interested in the actual project itself, the link is posted at the end.

### Pre-requisites:

- an emulator, such as Nox, that will let us play games on our desktop
- a programming language that lets us get native window position / pixel colors (we'll be using nodejs)
- a way to push commands to the game (we'll be using `nox_adb` for this)

So, the first and most important thing you need to do is figure out how you're going to attach to and grab the screen state. I made two small utilities that do this by hooking into win32 using C# via edge-js (what a mouthful):

- [https://github.com/seiyria/winpos](https://github.com/seiyria/winpos) (get window positions by name)
- [https://github.com/seiyria/pixcolor](https://github.com/seiyria/pixcolor) (get pixel color at x,y)

This will let us get started! Hooray!

### Setting up your project

You'll either want to make, or have a state machine library. For my purposes, it was easier to just fake it. We need to define states for each of the screens and states we care about. Lets take a look at some screenshots:

<blockquote class="imgur-embed-pub" lang="en" data-id="a/z9rSUtg"><a href="//imgur.com/z9rSUtg">Star Ocean Anamnesis</a></blockquote><script async src="//s.imgur.com/min/embed.js" charset="utf-8"></script>

There are quite a few screenshots here, mostly covering all of the states I care about for my project. Namely:

- the "main" screen
- the achievements screen
- the "gift box" screen
- the event screen (the stuff we're farming)
- the party join interfaces
- the combat interface
- the reward interface

This is a lot of screens, and a lot to juggle. But we're getting ahead of ourselves. Lets define some states:
```js
const WINDOW_STATES = {
  UNKNOWN: 0,
  BRIDGE: 50
};
```

We just need an enum to represent all of our possible states (in the order of the priority you want them checked). This list is not fully exhaustive, of course. Next, we need to figure out how to uniquely identify these screens. My strategy is to find a unique pixel on each screen/state that will only be there. This is easier than you think! There are so many gradients / color differences that it's easy to hook onto. So, for the "bridge" screen, my hook is the "Edit Favorite" bit on the left side. There is nothing like that anywhere else in the app, especially in that position, which makes it a prime candidate. We need to do this for each screen, but first, lets make sure our program recognizes it.

I track all of this information in a hash that maps the screen ID to the position, and the color at that position, like so:
```js
const WINDOW_INFORMATION = {
  WINDOW_STATES.BRIDGE: { hex: '159DF1', pos: { x: 210, y: 795 } }
};
```

(_note: This code was changed slightly, it should be using the notation that allows you to interpolate an object key when declaring it, ie `[WINDOW_STATES.BRIDGE]` but Jekyll doesn't like that so much. The correct version is in the repository!_)

For the "main" screen, there are several different states represented: whether you have a gift, whether you have an achievement, and of course there is the "not that" state, which we can infer by the absence of the prior states. So, that one image is very telling and very useful to us! As for how I got those particular colors, I ran the program and made it tell me what color is there, then I recorded it so I would know for the future what color it is - I didn't use an eyedropper or anything. There are plenty of ways to accomplish this, though.

So, now we have some states and somewhat-reliable detection _of_ those states. We can add some state-machine-esque transitions now in our polling loop (to be shown in a bit). For now, lets focus on transitioning between the main screen and another screen. Here is how I have it set up:
```js
const WINDOW_TRANSITIONS = {
  WINDOW_STATES.BRIDGE: {
    onEnter: () => {},
    onLeave: () => {},
    onRepeat: () => {
      tryTransitionState(WINDOW_STATES.BRIDGE, WINDOW_STATES.EVENT_SCREEN);
    }
  }
};
```
(_note: This code was changed slightly, it should be using the notation that allows you to interpolate an object key when declaring it, ie `[WINDOW_STATES.BRIDGE]` but Jekyll doesn't like that so much. The correct version is in the repository!_)

What I've is create a simple enter/exit/loop mechanism for each state. Since menu times and performance are different per person, computer, etc, I generally delegate all of my logic to the `onRepeat` loop, which will fire once every time you poll and find that particular state. So, when the program detects we're in the bridge state, it will repeatedly try to transition state between the bridge and the event screen (by clicking the big "Events" button). Essentially, the loop is something like this:
```js
// poll the nox instance
const poll = (noxIdx, lastState = WINDOW_STATES.UNKNOWN) => {
  const noxVmInfo = noxInstances[noxIdx];
  const { left, top } = noxVmInfo;

  const state = getState(left, top);
  const oldTransitions = WINDOW_TRANSITIONS[lastState];
  const curTransitions = WINDOW_TRANSITIONS[state];

  // we only change state if it's a new state
  if(state !== lastState && state !== WINDOW_STATES.UNKNOWN) {
    if(oldTransitions && oldTransitions.onLeave) {
      oldTransitions.onLeave();
    }

    if(curTransitions && curTransitions.onEnter) {
      curTransitions.onEnter();
    }

  } else if(state === lastState) {
    if(curTransitions && curTransitions.onRepeat) {
      curTransitions.onRepeat();
    }
    
  }

  // loop and do it again
  setTimeout(() => poll(noxIdx, state), OPTIONS.POLL_RATE);
};
```

Here, we check _each and every state_, check the pixel defined in each state, and then use that to determine what state we're in. If we find a new state, we transition out of the old one, into the new one. Otherwise, we call `onRepeat`.

We have to add all of the information for each state, transitions, and so on. There's also some other logic, like translating a screen coordinate to the actual VM coordinate, and sending the tap commands (via `nox_adb`) to the VM. These details are abstracted from this post, but can be found in the project itself.

So, we have states, we have transitions, and we can click the screen. That's all good, and that's what we need to pull this all together. At this point, we're in the home stretch. You can add additional features that pertain to game mechanics (like, click the screen to attack, or perform combo/rush attacks, whatever your game does).

Here's a screenshot of the application running side by side with the game itself. You can see the state transitions on the right side:

![Game and State](https://i.imgur.com/PaXX4fx.png)

That's pretty much all there is to it. Grab some pixels and turn it into a lean, mean, farming machine! You can find my [full project on GitHub](https://github.com/seiyria/soa-autofarm). It's by no means complete, but it does the job for now.
