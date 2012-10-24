---
layout: post
title: "iPad mini and Fusion Drive"
date: 2012-10-24 12:38
comments: true
categories: [iPad mini, Fusion Drive, Core Storage]
---

Apple announced what was probably the worst kept secret yesterday: the iPad mini. Pretty much what everyone expected and people with more access than me covered it [here](http://www.theverge.com/2012/10/23/3540572/apple-new-ipad-mini-hands-on), [here](http://gizmodo.com/ipad-mini/), and [here](http://reviews.cnet.com/ipad-mini/) for example.

Of all the things announced, the one I found the most interesting was the [Fusion Drive](http://www.apple.com/imac/performance/#fusion). 

<!-- more -->

On the hardware side, it's a combination SSD and HD. That's nothing really new, as hybrid SSD/HD's have been around for a little while. Interestingly, most hybrid drives have a small SSD (~8GB) to use purely as a cache, Apple's Fusion Drive apparently uses a 128GB SSD. [ArsTechnica](http://arstechnica.com/) has a nice analysis, where they speculate that it's an implementation of [automatic tiering](http://arstechnica.com/information-technology/2012/10/apple-fusion-drive-wait-what-how-does-this-work/). When Phil Schiller first mentioned it in the presentation, I was thinking of [Hierarchical Storage Management](http://en.wikipedia.org/wiki/Hierarchical_storage_management). Which I remember from my days of playing on Vaxen.

Clearly, there's a software component to Fusion Drive that's implemented in [Mountain Lion](http://www.apple.com/osx/), and ArsTechnica references something called "Core Storage". The only other reference I could find to "Core Storage" was [here](http://blog.fosketts.net/2011/08/04/mac-osx-lion-corestorage-volume-manager/) and a one-liner in the OSX Lion release note under [File Vault](http://developer.apple.com/library/mac/#releasenotes/MacOSX/WhatsNewInOSX/Articles/MacOSX10_7.html).

This seems like a fascinating solution, and I'm going to do more research to understand how it all works.
