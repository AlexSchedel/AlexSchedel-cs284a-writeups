---
title: "Project 1: Rasterizer"
has_children: false
nav_order: 1
---

# Project 1: Rasterizer

## Project Overview

In this project, I implemented and explored various rasterization and sampling schemes for rendering two dimensional images. 
This encompasses very simple approaches, like basic nearest neighbor point sampling, to intermediate approaches such as supersampling, 
all the way to more advanced techniques such as trilinear texture sampling. One of my basic goals with this project is to visually 
demonstrate the differences in these approaches while also making note of the computational requirements of each technique. Additionally, 
although these techniques are all used within the context of this project to render two dimensional images, they could easily be extended 
or extrapolated to other tasks as well.

Additionally, as is the case with any academic project, great value is also yielded by the implementation of algorithms. Making note of 
various formulae and sets of equations on paper is one thing, but actually implementing them often requires digging into and understanding 
them at a much deeper level. 

## Task 1 - Basic Triangle Rasterization

To rasterize triangles, I made use of the 3 lines algorithm from the lecture slides. I first determine the coordinates of the bounding box 
of the triangle, that being the smallest rectangle that will contain all points in the triangle. From there I iterate through the image, 
sampling at every x + 0.5 and y + 0.5 coordinate, to make sure my sample comes from the middle of each box and that I sample every box that 
is inside the triangle. I have a helper function which implements the point in triangle test from lecture. I run this three times, once for 
each side of the triangle. If a given sample passes or fails all three tests, I draw it directly onto the screen. This results in a fully 
rendered triangle.

In terms of efficiency, this algorithm is linear with respect to the size of the bounding box of the triangle. Before I run any loops, I 
first find the minimum and maximum x and y coordinates across all three points. This defines my bounding box. I then iterate over all the 
samples in this box with a double for loop. Here is an illustration of the output, with the pixel selecter zoomed in on a triangle to
demonstrate the sort of artifacts this process produces.

![Task 1](./images/t11.png)

## Task 2 - Supersampling

### Additional Data Structures

Unlike the previous task, my implementation of supersampling requires an additional data structure, that being a sample buffer. The 
sampling buffer is a 1D array which stores sampled pixels before they are drawn on the screen, allowing me to perform additional 
operations on the points before rendering. In supersampling in particular, this provides a temporary location to store svg pixels 
sampled at a higher rate which are then down sampled to the appropriate resolution. As an additional implementation note, because this 
array is 1D and needs to store 3D data (x and y corresponding to 2D location of each sample and the third dimension “s” corresponding 
to the number of the supersample within the x, y pixel) a clever indexing scheme is required to organize all the data. I settled on an 
indexing scheme of:
`sample_buffer[((y * width + x) * sample_rate) + s]`
Although this is not explicitly “row major order” it does organize everything in such a way so that all supersampled pixels can be 
neatly stored in the sample buffer.

### Algorithm

My supersampling algorithm is an extension of the previously described Task 1 algorithm. I make use of the three line test to determine 
whether or not a point is within the triangle as before, the main additions to the algorithm are which points are sampled and how they 
are processed once sampled.

The Task 1 algorithm made use of a double for loop to iterate through all the sample points. This algorithm makes use of a quadruple 
for loop or, perhaps more accurately, two double for loops. The first still iterates through all the potential x, y samples, while the
second is responsible for iterating through all the supersample points i, j within each x, y coordinate. The coordinates of these points are 
determined by repeatedly offsetting each x, y coordinate by multiples of `1 / sqrt(sample_rate)` until the next x, y coordinate would be 
reached. Each of these points then undergoes the three line test and is placed in the sample buffer with appropriate color. This has 
the net effect of antialiasing the triangles, by blending their original colors with an amount of the background color proportional to 
how many super sample points are inside or outside of the triangle.

Once in the sample buffer, the points require further processing. My algorithm works by summing the RGB values of all super sample 
points for a given x, y coordinate and then dividing by the total number of samples in order to average out the color. This has the 
effect of down sampling from a higher resolution to a lower one. One final note is that points and lines need to be handled specially, 
otherwise they would just appear to get lighter and later as the sampling rate increases due to not being super sampled and therefore 
having their colors incorrectly averaged with default background color pixels. To account for this, I simply changed `fill_pixel` to 
artificially “supersample” each point and line by placing `sample_rate` number of pixels of the sample color in the sample buffer. 
This means that when the color of a point or line is averaged with the other points, the average will always be equal to the original 
color of a point or line.

Example 1                           |  Example 2                             |  Example 2
:----------------------------------:|:--------------------------------------:|:--------------------------------------:
![Task 2 SR=1](./images/t21.png)  |  ![Task 2 SR=4](./images/t22.png)    |  ![Task 2 SR=16](./images/t23.png)

