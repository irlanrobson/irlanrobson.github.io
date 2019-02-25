---
layout: post
title: MATLAB R2015a Experiences
tags: []
---

I look forward to get a MATLAB license after using the R2015a trial for a week. Scalar and vector functions, gradient fields, and level curves are very easy to be evaluated in my experience. Example:

Let's plot the linearization of the elliptic paraboloid 

![](https://latex.codecogs.com/gif.latex?z=f(x,&space;y))![](https://latex.codecogs.com/gif.latex?f(x,&space;y)=x^2&space;+&space;y^2)

at 

![](https://latex.codecogs.com/gif.latex?P=(2,&space;2,&space;8))

To define this function in MATLAB ("MAT" comes from MATrix not "MATH"!) such 

![](https://latex.codecogs.com/gif.latex?-10&space;\leq&space;x&space;\leq&space;10,&space;y&space;=&space;x^T) we type 

[x, y] = meshgrid(-10.0: 1.0: 10.0) to get our working domain. The first argument is the minimum value for x, the third is the step size and the last is the maximum value. If we┬áommit the middle parameter then the step size becomes 1 by default. For the ones confused about this script output: this grid is a collection of 2D vector pairs. After this script is executed the 20x20 matrix y will be equal the transpose of 20x20 matrix x.

The version of MATLAB which I'm using doesn't allow us to define a function. However we can bypass this by evaluating our function at x and y using 
z = x.^2 + y.^2. Then we can plot (x, y, z) by typing mesh(x, y, z). We now have a nice graph:

{:refdef: style="text-align: center;"}
![z = x^2 + y^2](/assets/function_z.png)
{: refdef}

What about two plots on the same image? Fortunately this is done by using the hold on and hold off commands.

Example:

To plot level curves we type contour(x, y, z). So, by typing hold on; mesh(x, y, z); contour(x, y, z); hold off; we have the graph below.

{:refdef: style="text-align: center;"}
![Level Curves](/assets/level_curves.png)
{: refdef}

For the function initially given 

![](https://latex.codecogs.com/gif.latex?\frac{\partial&space;z}{\partial&space;x}=2x,&space;\frac{\partial&space;z}{\partial&space;y}=2y) 

For ![](https://latex.codecogs.com/gif.latex?P) one has

![](https://latex.codecogs.com/gif.latex?\frac{\partial&space;z}{\partial&space;x}=4,&space;\frac{\partial&space;z}{\partial&space;y}=4)

The tangent plane at 
![](https://latex.codecogs.com/gif.latex?P) is then

![](https://latex.codecogs.com/gif.latex?z&space;-&space;8&space;=&space;4&space;(x&space;-&space;2)&space;+&space;4&space;(y&space;-&space;2))

or

![](https://latex.codecogs.com/gif.latex?z&space;=&space;4x&space;+&space;4y&space;-&space;8)

To plot the Jacobian

![](https://latex.codecogs.com/gif.latex?J=\begin{pmatrix}\frac{\partial&space;z}{\partial&space;x}&\frac{\partial&space;z}{\partial&space;y}\end{pmatrix}=\nabla&space;f(x,&space;y))

evaluated at [x, y] then we execute [u, v] = gradient(z). This will make essentially the same as  meshgrid but now with u and v holding the computed (2D) gradient vectors. We can now plot these vectors on the x-y plane using quiver(x, y, u, v). Finally, we can plot the curve levels, gradient vectors, and the function, on the same image by typing hold on; mesh(x, y, z); contour(x, y, z); quiver(x, y, dx, dy); hold off; which generates a nice graph in my opinion:

{:refdef: style="text-align: center;"}
![](/assets/gradient.png)
{: refdef}

That's all. This post was basically a brief review on using MATLAB to plot some calculus functions. When I get the full version and if I the time allows me, I promess to bring more MATLAB graphs. 

Cheers to everybody!