---
title: Maze Generator
author: edransch
layout: post
permalink: /2012/05/maze-generator/
categories:
  - Maze
tags:
  - Maze
---
I made a thing! ( Check it out [here][1] )  
  
A few years ago I decided it would be a fun project to build a maze generator. My initial attempt was a failure, as is often the case. My second attempt turned out much better. It was written in Python using PyGame for the rendering and controls. It was my first experience using Python to build something, and it was a lot of fun.  
  
This is my third attempt at the project. My main motivation behind restarting was to be able to easily share the work with others. This is the kind of thing that&#8217;s (arguably) fun to play around with for a few minutes at most. It&#8217;s hardly something that&#8217;s worth installing PyGame just to play around with.  
  
So here we are, with a basic JavaScript & Canvas implementation of my Maze Generator and Solver. I&#8217;ve implemented two Maze Generation algorithms so far, more to come! Most of the algorithms I&#8217;ll be adding in the future are explained nicely here: <http://weblog.jamisbuck.org/2011/2/7/maze-generation-algorithm-recap> (Which I stumbled upon thanks to [Blake Winton][2]). So far I&#8217;ve got two generator algorithms:  


### Depth First Search 

This one is a classic. It&#8217;s about as simple as it gets to implement. I find it a little bit boring to watch, but the mazes it produces are pretty cool, tending towards longer, winding passages. 

### Prim&#8217;s 

I like watching this algorithm in action. The maze it produces tends to have shorter passages. I&#8217;ve found that the path to from start to finish is usually very straight, diagonally through the maze.

### More to Come! 

I&#8217;ll be implementing more maze algorithms as I have time. Please check out the source code on [github][3] if you&#8217;re interested

 [1]: http://www.erickdransch.com/maze
 [2]: http://bwinton.latte.ca/
 [3]: https://github.com/EkkiD/maze
