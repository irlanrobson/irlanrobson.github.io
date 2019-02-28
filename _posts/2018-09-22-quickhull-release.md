---
layout: post
title: Quickhull Release
tags: []
---

If we are writing a modern rigid body physics engine in C++, we should carry Quickhull in our backpack. That means: either use 
[qhull](http://www.qhull.org) or implement one from scratch.

It takes some time to learn how to use the qhull library. The implementation itself seems very reliable. The data structures exposed to the user, however, are somewhat difficult to digest. Perhaps because it was written many years ago. The other problem might be that is not very fast for run-time execution. However, qhull is a must for educational purposes as it contains a lot of computational geometry gems and related algorithms. For example, convex hull simplification.

Some time ago I decided to implement my own version of Quickhull. The mission objective was to design and implement a convex hull builder that could create convex hulls at run-time. Clean interface as a secondary objective. Quickhull is a very well studied and understood algorithm. These tasks weren't very hard to accomplish.

The source code for my version is [here](https://github.com/irlanrobson/bounce). This code fixes all the geometrical and topological errors that can appear during the convex hull construction. Therefore is reliable. It's fast: single memory allocation and pooling. The hull is a half-edge mesh. Everyone has heard of it. Such topics were revisited
[here](http://box2d.org/files/GDC2014/DirkGregorius_ImplementingQuickHull.pdf).

I encourage one to rewrite an existing algorithm putting some of its personal design decisions into it in order to make it more fast, reliable, and user friendly. The results are most of the time rewarding.

This post was somewhat a delayed announcement in form of a blog post.
