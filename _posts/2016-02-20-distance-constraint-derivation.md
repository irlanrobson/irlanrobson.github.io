---
layout: post
title: Distance Constraint Derivation
tags:
- Math
---

Sometimes it's desired to have a distance constraint. That constraint assumes a choosen separation distance for two points fixed in two bodies. I've written a derivation for it in 3D and shared
[here](/assets/distance_constraint.pdf). An implementation for it should be pretty simple since we have minimally one Jacobian and solve a 1D linear system. [Here](/assets/distance_joint.zip) is an example which you can use in your own projects.

### Appendix

The boring part is that it is possible to have a non-defined gradient when this constraint is satisfied. However, the problem is solved by skipping the constraint solving when that happens. Another workaround I can recommend is to define 3 distance constraints and solve them as a 3-by-3 linear system. This is sometimes called a 
block solver. You can find one [here](https://github.com/irlanrobson/bounce_lite/blob/master/src/bounce_lite/dynamics/joints/sphere_joint.cpp). Separating the points is done by chosing a point outside the volume of the shape of one of the bodies. Here, fortunately, there are neither square roots nor divide-by-zero checks which generally yields in greater performance.

Note that an inertia tensor 
![I](https://latex.codecogs.com/gif.latex?I) is a symmetric 3-by-3 matrix. So do its inverse. Therefore, 
![I = I^T](https://latex.codecogs.com/gif.latex?I%20%3D%20I%5ET). This simplifies the effective mass for this constraint (and can be applied to many other constraints) to

![JM^{-1}J^T = m_1 + m_2 + (I^{-1}_1 A_1)^T A_1 + (I^{-1}_2 A_2)^T A_2](https://latex.codecogs.com/gif.latex?JM%5E%7B-1%7DJ%5ET%20%3D%20m_1%20&plus;%20m_2%20&plus;%20%28I%5E%7B-1%7D_1%20A_1%29%5ET%20A_1%20&plus;%20%28I%5E%7B-1%7D_2%20A_2%29%5ET%20A_2)

Another way to compute the velocity constraint vector is to build the gradients of the position constraint with respect to the position vector directly (Jacobian) in a piece of paper. Remember that if 

![C(x(t)))](https://latex.codecogs.com/gif.latex?C%28x%28t%29%29%29) then by the chain rule we have:

![frac{ partial C } { partial t } = frac{ partial C } { partial x } frac{ partial x } { partial t } = Jv](https://latex.codecogs.com/gif.latex?%5Cfrac%7B%20%5Cpartial%20C%20%7D%20%7B%20%5Cpartial%20t%20%7D%20%3D%20%5Cfrac%7B%20%5Cpartial%20C%20%7D%20%7B%20%5Cpartial%20x%20%7D%20%5Cfrac%7B%20%5Cpartial%20x%20%7D%20%7B%20%5Cpartial%20t%20%7D%20%3D%20Jv)

Talking about the Jacobian, we can differentiate the velocity constraint to get the acceleration constraint. For that we can use a product rule-chain rule combo. By the product rule,


![frac{d}{dt} (frac{partial C}{partial x}frac{partial x}{partial t}) = frac{d}{dt}(frac{partial C}{partial x}) frac{dx}{dt} + frac{partial C}{partial x} frac{d}{dt}(frac{dx}{dt}) = frac{d}{dt}(J) v + J a](https://latex.codecogs.com/gif.latex?%5Cfrac%7Bd%7D%7Bdt%7D%20%28%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20x%7D%5Cfrac%7B%5Cpartial%20x%7D%7B%5Cpartial%20t%7D%29%20%3D%20%5Cfrac%7Bd%7D%7Bdt%7D%28%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20x%7D%29%20%5Cfrac%7Bdx%7D%7Bdt%7D%20&plus;%20%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20x%7D%20%5Cfrac%7Bd%7D%7Bdt%7D%28%5Cfrac%7Bdx%7D%7Bdt%7D%29%20%3D%20%5Cfrac%7Bd%7D%7Bdt%7D%28J%29%20v%20&plus;%20J%20a)

where

![a = frac{dv}{dt}](https://latex.codecogs.com/gif.latex?a%20%3D%20%5Cfrac%7Bdv%7D%7Bdt%7D). By the chain rule,

![frac{d}{dt}(frac{partial C}{partial x}) = frac{partial }{partial x}(frac{partial C}{partial x})frac{dx}{dt} = frac{partial }{partial x}(J)v](https://latex.codecogs.com/gif.latex?%5Cfrac%7Bd%7D%7Bdt%7D%28%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20x%7D%29%20%3D%20%5Cfrac%7B%5Cpartial%20%7D%7B%5Cpartial%20x%7D%28%5Cfrac%7B%5Cpartial%20C%7D%7B%5Cpartial%20x%7D%29%5Cfrac%7Bdx%7D%7Bdt%7D%20%3D%20%5Cfrac%7B%5Cpartial%20%7D%7B%5Cpartial%20x%7D%28J%29v)

Usually it's not required doing such a workload but I think it is a good vector calculus exercise.

### Example

Let us consider the linear motion of two points 
![x_1](https://latex.codecogs.com/gif.latex?x_1) and 
![x_2](https://latex.codecogs.com/gif.latex?x_2). To keep things down to earth we use block matrices again. Let's define

![x in mathbb { R }^ { 6 times 1 } = begin{bmatrix} x_1 &x_2 end{bmatrix}^T](https://latex.codecogs.com/gif.latex?x%20%5Cin%20%5Cmathbb%20%7B%20R%20%7D%5E%20%7B%206%20%5Ctimes%201%20%7D%20%3D%20%5Cbegin%7Bbmatrix%7D%20x_1%20%26x_2%20%5Cend%7Bbmatrix%7D%5ET)


![v in mathbb { R }^ { 6 times 1 } = begin{bmatrix} v_1 &v_2 end{bmatrix}^T](https://latex.codecogs.com/gif.latex?v%20%5Cin%20%5Cmathbb%20%7B%20R%20%7D%5E%20%7B%206%20%5Ctimes%201%20%7D%20%3D%20%5Cbegin%7Bbmatrix%7D%20v_1%20%26v_2%20%5Cend%7Bbmatrix%7D%5ET)

That is, the position and the velocity vector, respecively. Then, the vector stacking the 3 distance constraints that keep the points together at anytime is


![C(x) : mathbb { R }^ { 6 times 1 } rightarrow mathbb { R }^ { 3 times 1 }](https://latex.codecogs.com/gif.latex?C%28x%29%20%3A%20%5Cmathbb%20%7B%20R%20%7D%5E%20%7B%206%20%5Ctimes%201%20%7D%20%5Crightarrow%20%5Cmathbb%20%7B%20R%20%7D%5E%20%7B%203%20%5Ctimes%201%20%7D)


![C(x) = x_2 - x_1](https://latex.codecogs.com/gif.latex?C%28x%29%20%3D%20x_2%20-%20x_1)

The Jacobian is then

![frac{ partial C } { partial x } in mathbb { R }^ { 3 times 6 } = begin{bmatrix} frac{ partial C_1 } { partial x_1 } &frac{ partial C_1 } { partial x_2 } end{bmatrix}^T = begin{bmatrix} -I &I end{bmatrix}](https://latex.codecogs.com/gif.latex?%5Cfrac%7B%20%5Cpartial%20C%20%7D%20%7B%20%5Cpartial%20x%20%7D%20%5Cin%20%5Cmathbb%20%7B%20R%20%7D%5E%20%7B%203%20%5Ctimes%206%20%7D%20%3D%20%5Cbegin%7Bbmatrix%7D%20%5Cfrac%7B%20%5Cpartial%20C_1%20%7D%20%7B%20%5Cpartial%20x_1%20%7D%20%26%5Cfrac%7B%20%5Cpartial%20C_1%20%7D%20%7B%20%5Cpartial%20x_2%20%7D%20%5Cend%7Bbmatrix%7D%5ET%20%3D%20%5Cbegin%7Bbmatrix%7D%20-I%20%26I%20%5Cend%7Bbmatrix%7D)

where 

![I](https://latex.codecogs.com/gif.latex?I) is the identity matrix. Note that the Jacobian is just a vector of gradients in block form. From those we can compute the velocity constraint vector easely.

![frac{ partial C } { partial t } = begin{bmatrix} -I &I end{bmatrix} v = -v_1 + v_2 = v_2 - v_1](https://latex.codecogs.com/gif.latex?%5Cfrac%7B%20%5Cpartial%20C%20%7D%20%7B%20%5Cpartial%20t%20%7D%20%3D%20%5Cbegin%7Bbmatrix%7D%20-I%20%26I%20%5Cend%7Bbmatrix%7D%20v%20%3D%20-v_1%20&plus;%20v_2%20%3D%20v_2%20-%20v_1)

We can see now that building the Jacobian by inspection is more painless, and predict that it can be specially in the presence of angular motion.