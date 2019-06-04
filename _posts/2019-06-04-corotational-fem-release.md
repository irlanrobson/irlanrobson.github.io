---
layout: post
title: Corotational FEM Release
tags: []
---

After publishing a mass-spring based [cloth](https://github.com/irlanrobson/bounce/tree/master/include/bounce/cloth) last year, I started implementing a finite-element based soft body. Fortunately last month I published the source code 
for the soft body as a feature for [Bounce](https://github.com/irlanrobson/Bounce). 

The soft body simulation uses the corotational linear finite-element method explained in [1]. For moving forward with the soft body dynamics we use a first order backward implicit Euler integrator followed by a simplified constraint solver that can robustly handle 
vertex contacts. This ensures unconditional stability. The solver can perform fairly well for small time steps and good tetrahedrons.

The simulator supports element-based elastic and plastic behaviour. This is pretty usefull for environment deformation. It also supports kinematically driven nodes which can be useful for simulating deformable sliding doors and elevators. Static and kinematic nodes is possibly by using the constraint satisfaction technique described in [3]. The solver works by establishing target velocities at the beginning 
of a Conjugate Gradient (CG) solver and maintaining the velocities using a filtering procedure to cancel any delta velocity that would get added to the target velocity during the CG. 

For simplicity an user can set body material parameters through the body definition in my implementation. However, with some modifications one can easely set per element parameters. 
This can be useful if you are simulating melting behaviour, for example, where only some element material properties changes according to thermal laws. The elastic material 
parameters supported in our model are Young modulus, and Poisson's ratio. These values can be easely found in the material science literature and on the Internet.

While this is super cool, one could ask: Since you already have an unconditionally stable mass-spring system, 
why even care about building another soft body simulator? 

Of course one could create a particle for each mesh vertex, a spring for each tetrahedral edge, and cross his fingers to hope the simulation runs as expected. Or even use some kind of pressure law 
to inflate a cloth. However, mass-spring systems cannot capture volumetric effects. 
Their behaviour depends on how the mesh is structured and how the stiffness and damping values are assigned to each spring. Anyone experienced with game physics knows that 
those values are not easy to tune. The other subproblem is that creating many springs could
possibly slow down the simulation significantly. Remember that performance is the main criteria for games. Nonetheless, it seems that there are libraries that are using mass-spring systems for simulating soft bodies. 
Check it out the great [VEGA FEM](http://run.usc.edu/vega/) for example.

For those who don't know, finite-element based simulation would only be used for off-line simulations in the past. Then in 2004 [1] published his paper 
about making FEM suitable for interactive simulations. Further [2] explained in detail how they made their fully-featured simulation package fast. Under the name of Digital Molecular Matter (DMM), 
their package was used in the great *Star Wars: The Force Unleashed* by LucasArts. The engine was so nice that it could possible be a substitute for a typical rigid body engine without problems.

Finally back to the engine. Here's a video demonstrating the simulation of a pinned elastoplastic beam being deformed running at interactive rates:

<iframe width="560" height="315" src="https://www.youtube.com/embed/xxGV3_pnmJQ" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## References

[1] Matthias Muller, Markus Gross. [Interactive Virtual Materials](http://matthias-mueller-fischer.ch/publications/GI2004.pdf).

[2] Eric G. Parker, James F. O'Brien. [Real-Time Deformation and Fracture in a Game Environment](http://graphics.berkeley.edu/papers/Parker-RTD-2009-08/Parker-RTD-2009-08.pdf).

[3] David Baraff, Andrew Witkin. [Large Steps in Cloth Simulation](https://www.cs.cmu.edu/~baraff/papers/sig98.pdf).