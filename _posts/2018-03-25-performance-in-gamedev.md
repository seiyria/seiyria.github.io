---
title:  "Performance in Game Dev"
date:   2018-03-25 12:00:00
categories: gamedev, performance
---

When you're making a game, performance is _important_. This goes without saying. However, to what extent is it important? 



If you're making a small tetris clone, you can probably get away with some pretty gross algorithms and every computer, phone, and toaster these days can play it happily. As you'd expect, the need for correctness in your algorithms and data structure choice is correlated directly to how complex of a game you have.

So I've been working on a multiplayer RPG ([Land of the Rair](https://rair.land)) and, you guessed it, I did some pretty bad things. For my game's small maps (40x40, even up to 150x150) with ~500 npcs on the map, performance was pretty alright. It had it's blemishes but nothing's perfect, right? Well, the latest map we're working on is intended to be a bit bigger: 200x200 with up to 1000 (maybe 1500) npcs. So, bigger space, and more monsters on top of that.

The lesson is very simple: **Do not iterate over every creature in your world, ever, unless you absolutely have to**. Especially if you make a helper method that makes getting them all pretty easy. In my architecture, the world contains spawners, which manage and contain monsters. So, I have a helper that would get all monsters. On top of that, I have targetting functions (called _every_ tick) that check _every_ creature in the map to see if it can be targetted by the current creature. It used to not be so bad, so I forgot about it. I also have a function that every player skill in the game calls when it tries to match a target (when you click on one) to the actual npc ref. Again, it went unnoticed.

I was thinking about this and it became extremely noticeable when I had a function that checked every creature on the map for the 1x1 radius around the target. In a 200x200 map, when I only need a 3x3 chunk, that's extremely wasteful. So, I asked around and found out that apparently quadtrees are made for this sort of thing. I swear, my CS education has only ever really helped me in game dev! So I went searching for a library that came close enough to what I wanted. A lot of the JS quadtree implementations leave something to be desired (whether it be docs, examples) but eventually I came across [RBush](https://github.com/mourner/rbush) (which is an R-Tree, not a quadtree).

Popped it in, rewrote some code, and now everything is running way better. I don't have actual numbers, but players reported 2x faster speed (at least, it was that noticeable) with the first update - this was just one change. Today, I did a deeper dive and found some other functions that iterate over everything, fixed them, and I'm looking forward to hearing about the result.

Anyway, if you're making a multiplayer game that depends on anything spatial, you should probably consider using a data structure that facilitiates spatial searching. In every case I'm sure it will be faster than an O(N) loop hundreds of times a second. It's incredibly obvious in hindsight and should go without saying, but here's to the people like me that need a reminder.
