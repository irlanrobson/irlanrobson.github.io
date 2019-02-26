---
layout: post
title: Bounce Lite Update
tags: []
---

I've added two joint types to Bounce Lite. Ball-in-socket, and revolute with angle limits. Bounce Lite is available [here](https://github.com/irlanrobson/Bounce).  Here's a video showing how five revolute joints can be used to simulate a small ragdoll:

<iframe width="560" height="315" src="https://www.youtube.com/embed/Xfu-hYiAhFk" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

As it can be seen on the video, as the iteration count increases the human legs keep from falling into the ground after spending some time planning on it. The sequential impulse solver convergence is augmented when the density of the body parts is balanced or as the solver iteration count increases.

Angle limits were added to avoid a leg and an arm from rotating more than 45 degrees. This, however, isn't a productive solution because most of the body joints of an human are best approximated by spherical joints and conical limits. (Therefore, I look forward to add spherical joints and conical limits in Bounce.)

Here is another video that shows a "mouse joint" which can be used, for example, along with ray casting to pick body shapes:

<iframe width="560" height="315" src="https://www.youtube.com/embed/CX7ml-p-bx4" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

If you haven't read my post on how to derive the Jacobian for the constraint above then you can find it [here](/2015/08/22/mouse-constraint-derivation.html). It's basically a sphere constraint assuming the mouse point as a point with infinite mass, and is actually a good exercise to understand the basics of linear equality constraints.

In addition to the previous changes, I've updated all Bounce Lite source code as last week I changed my old wordpress blog name consequently invalidating the old source code zlib licence headers.

You have reached the end of this post. Here, I showed how I would simulate a small ragdoll system using Bounce in the abscence of spherical joints. I showed as well what a "mouse constraint" is. That's all.