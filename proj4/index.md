---
title: "Project 4: ClothSim"
has_children: false
nav_order: 1
---

# Project 4: ClothSim

## Overview

## Part 1: Masses and Springs

To begin the project, I first used `PointMass` and `Spring` structs to create a simple cloth mesh.

### Implementation

The first step in building the grid is creating all the `PointMass`es which will become the start and end points for each `Spring`. There should be `width * height` points and they should be evenly spread across the shape of the cloth. To figure out where to place these points, I first divided the number of points in each direction by the total width and height of those directions. This gave me a scalar quantity which I could later use to multiply the `x`, `y` indices of each point to find the actual position. Next, I needed to determine if each point was to be pinned or not. To accomplish this, I iterated through all the points in the set of pinned points and checked if my newly created point shared coordinates with any of them. If both the start and end points of the pin were in the set, it would become a pinned point. Finally, my implementation has support for both horizontal and vertical meshes. To achieve this was relatively straightforward as I could just swap the places of the coordinates in my vector.

The second main step was to build all of the springs. To do this I iterated through all the points I created, and checked if their neighbors existed with a helper function I wrote. If those neighbors did exist, I would use them and the original point to create a spring with a given constraint. The three possible constraints are:

`STRUCTURAL` - These springs do not deform and make sure that the cloth maintains its internal structure even as it is moved about.

`SHEARING` - These springs do not get longer, but they will move relative to other springs. Like the structural springs, these help the cloth maintain its shape, however they allow movement.

`BENDING` - These springs can deform to get longer, allowing the cloth to “stretch” out as one would expect it to under real world forces.

A problem I faced during this implementation was differentiating between pointers and structs. In particular, the vectors which represent the points and the springs contain the literal structs, not pointers to those structs. This means that in order to modify them, one must first access each item and then return a pointer to it, even when using foreach loops. I forgot to do this at first, meaning that although my calculations on the individual points and springs were correct, I was making changes to a local copy rather than the actual mesh elements, resulting in no change overall.

### Results

Here is the full created spring mesh from various angles.

From Front         |  Spring Closeup             |  Oblique Angle
:------------:|:--------------:|:---------------------:
![Part 1](./images/t11.PNG) | ![Part 1](./images/t12.PNG) | ![Part1](./images/t13.PNG)

Here we can see the writeframe with various springs removed to better understand it’s internal structure

Without `SHEARING`          |  With only `SHEARING`           |  With all `Spring`s
:------------:|:--------------:|:---------------------:
![Part 1](./images/t14.PNG) | ![Part 1](./images/t15.PNG) | ![Part1](./images/t16.PNG)

## Part 2: Simulation via Numerical Integration

With the `Spring` mesh constructed, I moved on to implementing the actual physics that would allow me to simulate realistic cloth motion.

### Implementation

This simulation involves three main components: computing the total force acting on each point mass, using Verlet integration to compute new point mass positions, and finally constraining position updates.

I first compute the forces acting on each point. To do this, we can first look to our middle school science textbooks to find a very relevant equation: $F = ma$. I take all of the `external_accelarations`, multiply them each by the mass, and tally all the resulting forces into a single `force` variable. I then go through every `PointMass` and add this force to it. The next task is to update the `Spring`s appropriately. For this I subtract the positions of the end of each `Spring` to get a direction vector, From that vector I calculate the force on each `Spring` by Hooke’s law (this one was in a high school science textbook for me if I recall correctly). That force calculated, I then normalize the direction vector so that I know which way the `Spring` has to be adjusted. Finally, assuming that each specific type of `Spring` is enabled in the simulation, I go into it’s respective `PointMass`es and add in the Hooke force multiplied by the direction to one point, as well as an equal and opposite force (yay back to middle school science) on the other `Spring`.

Next I calculate the new positions of each `PointMass` based on all of the forces I just added in. This part of the code is much simpler, as basically it just requires implementing equation $x_{t + dt} + (1 - d) * (x_t - x_{t-dt}) + a_t * dt^2$.

Unfortunately, despite the ostensibly simple nature of this section of the code, this is where my most frustrating bug occurred. Notice that the new position of the vector is dependent on the old position. I, in an effort to be far more concise and clever than I apparently have any business being, originally set the `lastPosition` variable to the value in `position` for each vector before calculating its new value. This resulted in the cloth descending generally as it should have, but very very slowly. Originally I thought that the cloth was supposed to descend super slowly until I fixed this bug and first saw it flop down over the course of a second or so as intended.

The final part of this implementation involves constraining the `Spring`s so that they do not deform too much in the simulation. Towards this end, I made it so that `Spring`s would not extend more than 10% of their original length. As is the case in the first part of this part, this involved subtracting the ends of the `Spring` to get a direction vector and then doing some LInEaR ALgeBrA on it. I took the norm of the direction vector to see if a `Spring` is potentially overstretching. If it is, I then compute a correction vector and add it into the `PointMass`es. There is an additional complexity here however. If one of the `PointMasses` is pinned, I add the correction to only the unpinned `PointMass`, if they are both unpinned, I spit the correction in half and add to each `PointMass`. There is no need to account for when both `PointMass`es are pinned, because in this case, the `Spring` would not be moving at all and thus not need to be corrected. Implementing this step also proved to be useful for debugging, as before I did, my mesh would basically just explode. Once this was implemented, it would flop and flail around, but because it would not overly deform, would remain in one place. 

### Results

Here are a few screenshots showing the cloth as it falls with the default parameters.

Before Fall         | Starting Fall  | Middle Fall      | Rest State 
:------------:|:--------------:|:---------------------:|:-------:
![Part 2](./images/t21.PNG) | ![Part 2](./images/t22.PNG) | ![Part 2](./images/t23.PNG) | ![Part 2](./images/t24.PNG)

We can now modify some of the parameters of the simulation to better understand how they affect the motion of the cloth.

To start, we can try using different values of `ks`, which represents the spring constant. Here is the cloth with three different `ks` values paused at a similar point in the simulation.

`ks` = 5        | `ks’ = 5000             |  `ks` = 50000 . 
:------------:|:--------------:|:---------------------:
![Part 2](./images/t25.PNG) | ![Part 2](./images/t26.PNG) | ![Part 2](./images/t27.PNG) 

Notice that lower values of `ks` make the cloth more “jiggly” and behave more erratically, while high values make it appear stiffer. We can understand this phenomenon if we think about how `ks` is used in the simulation. `ks` is used in Hooke's law to generate a force described by the equation $k_s * (\lVert p_a - p_b\rVert - l)$. A larger value of `ks` will make for a larger force. This force is applied to the `PointMass`es within each `Spring` in order to stabilize it. As the force gets larger and larger, it will overpower other forces acting on the `PointMasses`, making them more and more rigid.

Next we can change the `density` value. This produces results most notable when looking at the rest state of the cloth.


`density` = 1         | `density` = 15             | `density` = 150      | `density` = 1500 
:------------:|:--------------:|:---------------------:|:-------:
![Part 2](./images/t28.PNG) | ![Part 2](./images/t29.PNG) | ![Part 2](./images/t210.PNG) | ![Part 2](./images/t211.PNG)

`density` generally corresponds to the weight of the cloth, so we see that as we make the `density` greater and greater it makes the cloth dip more and more in the final rest state as we would expect to see in real life.

Finally, we can take a look at `damping`.

`damping` = 0%        | `damping` = 0.1%             | `density` = 0.2%  
:------------:|:--------------:|:---------------------:
![Part 2](./images/t212.PNG) | ![Part 2](./images/t213.PNG) | ![Part 2](./images/t214.PNG) 

`damping` is a constant which resists movement. This means if it is set to a low value, the cloth will move for a long time before settling, while larger values will cause it to move slowly. In the pictures above, I took screenshots of the cloth and the maximum extent of its backwards bending (meaning how far the cloth “overshot” where gravity ultimately wants it to go). Notice that in the undamped version, the cloth flies completely backwards, in the second one, it bends backwards a bit, and in the final picture it generally comes to a standstill. Even larger values of `damping` than this just make the cloth fall more slowly.

Finally, here is an example of shaded and pinned cloth under the simulation.

![Part 2](./images/t215.PNG) 

## Part 3: Handling Collisions with Other Objects

The cloth can now interact with gravity, but it is not yet able to interact with other objects in a scene, so for the next part of this project I implemented collisions with two primitives, namely spheres and planes.

### Implementation

To handle sphere collisions, I went back to the vector math I used in part 2. I used the origin of the sphere of a given `PointMass` to find a direction vector. I then use the norm of this vector to determine if the `PointMass` is inside or on the sphere. If it is not, there is nothing to do. If it is, I calculate a correction vector by normalizing the direction vector and multiplying it by the distance that the `PointMass` is inside the sphere. An issue I encountered in this section is that I initially miscalculated the correction vector by a small amount which caused the cloth to jiggle off any sphere it touched near instantly, fixing this allowed to to drape over spheres as expected.

For plane collisions, I find the positions of both the current and last position of each `PointMass` and check if they are on opposite sides of the plane. This can be done with vector math and the line test used all the way back in project 1 to render simple triangles. If the positions are on the same side of the plane, there is nothing to do, if they are not, I create a correction vector from the normal of the plane multiplied by a small constant and apply it to the position of the `PointMass`.

### Results

Here is a cloth draping a top a sphere with various `ks` values.

`ks` = 500        | `ks` = 5000           | `ks` = 50000  
:------------:|:--------------:|:---------------------:
![Part 3](./images/t32.PNG) | ![Part 3](./images/t31.PNG) | ![Part 3](./images/t33.PNG) 

Notice that, with lower values of `ks`, the cloth hugs the sphere more tightly. As is the case in part 2, this is because high values of `ks` keep the springs from deforming relative to their endpoints.

Here is an image of the cloth resting on the plane. I made the cloth purple because [purple is my favorite color](https://youtu.be/y2gA_kWieYE?t=3). 

![Part 3](./images/t34.PNG) 

## Part 4: Handling Self-Collisions

Though now able to interact with basic primitive geometry, there is one thing that the cloth can still not interact with: itself. In this part of the project I implemented self collisions to deal with this.

### Implementation

The implementation of this behavior required three functions. First, I created a hash function which would create an int corresponding to a subbox of the space that each `PointMass` was in. Then I used this hashing function to create a spatial map, which effectively mapped those points into boxes. Finally, I implemented a function that did the work of actually adjusting the coordinates of colliding points.

Although the spec suggested using `fmod` to implement this, I found no need. Instead I simply floor divided the individual coordinates of each `PointMass`’s `position` vector, and then multiplied them together in such a way as to make each possible combination of `x`, `y`, and `z` values produce a unique hash. To do this I actually took inspiration from the supersampling part of project 1, as placing supersampled pixels into the 1D frame buffer required a similar mapping.

I then implemented a function which would actually build the spatial map. The works by using the aforementioned hash function to hash all of the `PointMasses` into a particular bucket, which then goes into a map. This part of the project provided the most headaches for me. How map syntax in C++ works eludes me to this day. I had a lot of trouble putting things into the map and getting them out, in particular I ran into a large amount of type conversion errors, and eventually resorted to simply trying various syntax conventions until something finally worked. Additionally, as was the case in project 3-2, there were some heap shenanigans to deal with as the vector itself had to be allocated on the heap to make sure that things did not get garbage collected upon the return of the function call.

I then implemented a function to handle the actual collision, similar to how I had done that for the sphere and plane primitives. To do this, I interacted through every `PointMass` within the same box as the given one and found their direction vectors. If the second point was too close (with `2 * thickness`), I created a correction vector to repulse the original `PointMass`. I then summed up and averaged all the correction vectors, as there can be many given there are a lot of points in the mesh, and finally applied it to my `PointMass`.

Finally, I added logic into the simulate function to make sure that these new functions were actually called when I ran the program. This resulted in one last headache, as I initially forgot to actually call my function that created the map, giving me segfaults when I then tried to access it in later functions.

### Results

Here is a demonstration of what the cloth looks like as it falls onto 
itself, ultimately settling on the ground in a rest state:

Starting to Fall        | Currently Falling          | At Rest State 
:------------:|:--------------:|:---------------------:
![Part 4](./images/t41.PNG) | ![Part 4](./images/t42.PNG) | ![Part 4](./images/t43.PNG) 

We can also examine what changing the simulation parameters does to the cloth. Here is the cloth with different `density` values.

`density` = 1        | `density` = 15          | `density` = 150  
:------------:|:--------------:|:---------------------:
![Part 4](./images/t44.PNG) | ![Part 4](./images/t45.PNG) | ![Part 4](./images/t46.PNG)

Notice that lower `density` values allow the cloth to fold more cleanly over itself, while larger ones cause it to crumple up more closely.

Here are various `ks` values on the cloth. The screenshots are taken shortly after the cloth hits the surface and as it is settling into its rest state. 

`ks` = 5        | `ks` = 500          | `ks` = 5000    |   `ks` = 50000  
:------------:|:--------------:|:---------------------:|:------------------------:
![Part 4](./images/t47.PNG) | ![Part 4](./images/t48.PNG) | ![Part 4](./images/t49.PNG)  | ![Part 4](./images/t410.PNG) 

Recall that high `ks` values push `Spring`s apart from each other. We can see this in the images above. Lower `ks` values result in the cloth bunching up more randomly and more tightly, while higher values cause it to look more smooth.

## Part 5: Shaders

For this part of the project I implemented various shaders for my cloth simulation using GLSL. This language uses C like syntax to specify how the vertices of a given texture will change as well as how those vertices will be shaded. The reason I used an extended language instead of just defining my shaders in C++ is that this language is optimized to be parallelized and split across multiple GPUs, ensure that my renders will be able to operate in real time with the added benefit of ensuring my CPU will not spontaneously combust when I try to run them.

GLSL breaks shaders down into a two component pipeline of vertex and fragment shaders. Vertex shaders describe how individual vertices of a mesh will be displaced by the shader. This is of particular relevance in implementing bump mapping and displacement shading.

After the vertex shader comes the fragment shader. The vertex shader sends variables to a specific output to be used by the fragment shade. The fragment shader shades all the individual fragments however they should be. This could be using Blinn-Phong, diffuse, or a custom texture.

### Blinn-Phong Shading

One of the first shading algorithms I implemented was Blinn-Phong. This is a good starting point as it is a fairly simple shading model that does not involve modifying the vertex positions. Blinn-Phong as who component light sources that get combined together in the final render. The first is ambient lighting, meaning it shows the lighting of the scene. The second is specular reflection, meaning the shiny and off center reflections of light that allow the object to look photoreal.

We can take a look at the individual components of the shader as well as how they come together to create our final render.

Only Diffuse         | Only Specular             |  Diffuse + Specular
:------------:|:--------------:|:---------------------:
![Part 5](./images/t51.PNG) | ![Part 5](./images/t52.PNG) | ![Part 5](./images/t53.PNG)

### Texture Shading

In addition to lighting simulations, we can also map textures to the cloth. Here is a nice picture of me that I took in Peru this January a few days before having to flee the country due to political violence stretched over the cloth texture in two positions

Mid Drop        | Over Sphere        
:------------:|:--------------:
![Part 5](./images/t54.PNG) | ![Part 5](./images/t55.PNG) 

### Bump / Displacement Mapping

From there we can move on to textures which also involve changing the actual position of the vertices as well as applying shading. Here is a bump mapped texture at two different sphere resolutions. 

Resolution = 16       | Resolution = 16      
:------------:|:--------------:
![Part 5](./images/t56.PNG) | ![Part 5](./images/t57.PNG) 

And here is a displacement mapping at two different sphere resolutions.

Resolution = 128      | Resolution = 128       
:------------:|:--------------:
![Part 5](./images/t58.PNG) | ![Part 5](./images/t59.PNG) 

COMPARE

### Mirror Shader

Finally, we can also implement a mirror shader.

Mid Drop        | Over Sphere        
:------------:|:--------------:
![Part 5](./images/t510.PNG) | ![Part 5](./images/t511.PNG) 

Write up link: https://alexschedel.github.io/AlexSchedel-cs284a-writeups/
