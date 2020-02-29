---
title:  "Tabletop Simulator Debugging Errors"
date:   2020-02-29 9:00:00
categories: tabletopsimulator
---
A quick little post, but something that is strangely hard to search for, so here's hoping this becomes a good search result for this error.



On Tabletop Simulator, it's possible to see a bunch of weird errors with mods. You might see things like `[XmlLayout] Ignoring duplicate id value 'X. Id values must be unique <XX>`. 

![error picture](https://i.imgur.com/OETcwZJ.png)

This is probably one of the most common errors I see, and until today, I had no idea why I was seeing them at all. As it turns out, if you have logging enabled in Tabletop Simulator (`-log` launch option), you'll see errors in mods. It seems like most people have logging turned _off_, so unless you're working on your own mod, you may as well turn it off.
