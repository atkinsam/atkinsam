---
layout: post
title: Fractal Animation Using OpenCL
---

<img src="/images/animation.jpg" style="width: 100%"/>

In this post I explain how I created animated videos of fractal 
transformations, specifically transformations of the Julia Set. 
The post explores three different implementations of the program:

* Serial computation in Python
* Parallel frame computation in C++
* Parallel pixel computation in C++ using the OpenCL framework

Each implementation performs the same overall task of creating transformation
animations, yet each does so in a very different way. 

In the first half of this post I cover the basic operations for the 
task, independent of implementation. Later, I explain each implementation
and how each affects the run time of the program.

## Complex Quadratic Polynomial Representation

Computing Julia Sets is simple using the complex quadratic 
polynomial representation. The equation 

$$ f_c(z) = z^2 +c  $$

simplifies the chaotic Julia Set into a simple polynomial which is easy
to work with in a program. I set each element of a matrix to a complex 
$$ z $$, relative to the position of the element in the matrix. I also
define complex $$ c $$. Variable $$ c $$ changes the form of the Julia set,
which is explained later in the post.
For each element 
in a matrix with a now-known $$ z $$ and $$ c $$, the program applies the
above equation until reaching a sufficient limit on the magnitude of $$ z $$
(or reaching the color white).

The following algorithm applies the copmlex quadratic polynomial 
representation of the Julia Set in python and is the one I used for every 
version of the program I wrote. 

<script src="https://gist.github.com/atkinsam/4e08805a266700464245c581ff22e8e9.js"></script>

The function above generates a `size x size` matrix in which each element
contains a value between 1 and 255. This value represents the color assigned
at that element. 

## Color Mapping

In a grayscale image, black is assigned to 255 and white is 
assigned to 0. In multicolored frames, I applied a custom colormap to the 
matrix. This would map every grayscale value in the matrix to an RGB color,
depending on an input image containing a gradient of colors. Here is a 
visualization: 

<img src="/images/colormap.jpg" style="width: 100%"/>

<img src="/images/apply_colormap.jpg" style="width: 100%"/>

## Animation

Creating animations of the fractal images simply requires changing the 
parameters given to the Julia Set algorithm explained earlier in the post. 
There are six parameters and each are explained below.

__`size`__ defines the dimensions of the matrix/image. If `size = 50`, the 
dimensions are set to 50 rows and 50 columns.

__`zoom`__ defines the boundaries of complex $$ z $$ in the earlier algorithm. As 
the boundaries get closer (`zoom` gets smaller), the matrix will be filled with a 
smaller section of $$ z $$ values, thus _zooming in_ on the image. 

__`center_re`__ and __`center_im`__ define the center point of the $$ z $$ values. 
In the very center of each matrix, complex $$ z $$ is set to `center_re` 
$$ + $$ $$ i $$ $$ \times $$ `center_im`. These values are used in conjunction
with `zoom` to define the $$ z $$ values across the matrix. In the context
of the rendered frames in animation, these two parameters shift the image
left/right and up/down, depending on the sign and magnitude of each variable.

__`c_re`__ and __`c_im`__ are user-set and directly affect what the Julia Set
will look like. Incrementing/decrementing `c_re` or `c_im` (or both) defines 
how the fractal transforms throughout the frames of an animation. 

I have provided a few examples of arguments and their respective 
Julia Set renderings at the bottom of this post. The color maps used 
in the examples can be found
[here](https://github.com/atkinsam/opencl-fractal-animation/tree/master/colormaps).

Creating sequential frames of fractal images required defining two additional
constants at the beginning of the program: `c_re_step` and `c_im_step`. The
values of these variables are added respectively to `c_re` and `c_im` for 
each frame of the video.

Once each frame is computed, a color map is applied and the image is exported 
to storage. I found that the fastest way to export the image was using a raw
image format like ppm, as no encoding is required. At the end of the 
program, I use ffmpeg to take sequential 
images and create an MP4 video. The exact command I used is below, but  
this changes depending on where files are exported and how they are
named.

    $ ffmpeg -f image2 -r 60 -i tmp/F%04d.ppm -vcodec mpeg4 -q:v 1 -y out.mp4

This command takes all images in the ./tmp/ directory named F0001.ppm, F0002.ppm,
etc. and encodes them sequentially into an mpeg4 video at 60 frames per 
second with video quality set to the maximum.

## Programming models

As explained earlier, I implemented the program using three different models:

* Serial computation in Python
* Parallel frame computation in C++
* Parallel pixel computation in C++ using the OpenCL framework

These were implemented to speed up the process of creating the videos. When 
some high-resolution videos were taking upwards of an hour to compute, 
I decided to re-write the program. Summaries of each model are below:

## Serial

The serial model is simple. It requires a 
for-loop which iterates through each frame number and computes a Julia Set
for that frame using the appropriate adjusted `c_re` and `c_im` values. A 
diagram of its execution is below.

<img src="/images/serial.jpg" style="width: 100%"/>

The downside to the serial implementation is speed. The program uses one 
process and zero child threads. It computes each pixel of each frame sequentially,
applies a color map to the entire frame, exports the frame to the disk, 
and then moves on to the next frame when finished. There is no parallelization
whatsoever, and the program can take hours to output a video depending on the 
size and parameters.

## Parallel frames

The parallel frames model is similar to the serial model, with one significant
difference. The entire sequence of frames is divided up into _n_ groups, 
where _n_ is a user-set constant of the number of child processes or
threads to which the load is distributed. The diagram below shows how the 
execution of this model happens. 

<img src="/images/parallel.jpg" style="width: 100%"/>

The downside to the parallel frames implementation is that it still carries
a similar weakness to the serial model. The video is created on a per-frame
basis, so in a process's/thread's given sequence of frames, no two can be
computed at the same time. It also means that a non-optimal amount of 
context switches take place, as each frame is written to the disk in between
the termination of its computation and the start of the next frame's 
computation.

## Parallel pixels (OpenCL)

The parallel pixels model uses the OpenCL framework to compute sequential
Julia Sets. The source code for the entire program can be found [here](https://github.com/atkinsam/opencl-fractal-animation) 
on my Github. Instead of parallelizing the execution on the frame level, it
instead treats every element of the Julia set (every pixel of the image) as
a separate operation. In OpenCL, this kind of elementary computation can 
be performed in something called a kernel. A kernel can be queued onto an 
OpenCL-compatible device like a multicore processor or modern GPU. To 
differentiate between frames, I used an NDRange (N-Dimensional Range) kernel, 
shown in the diagram below (from 
[OpenCL 1.2 Specification](https://www.khronos.org/registry/cl/specs/opencl-1.2.pdf)).

<img src="/images/ndrange.jpg" style="width: 100%"/>

In the case of Julia Set computation, the NDRange kernel execution model is
easy to visualize. Each element of the set can be computed
independently once complex $$ z $$ is known based on the position of the 
element. In this case, each work-group shown in the diagram only contains 
one work-item. Both $$ G_y $$ and $$ G_x $$ are set to `size`, and each 
element of the Julia Set is computed in a single work-item. 

Using this model, I created NDRange kernels for each frame of the animation and
then enqueued them for execution on the OpenCL device. Before enqueueing them,
OpenCL buffers are created for the range of $$ z $$ values as well as the 
color map used for the set. This speeds up execution because the OpenCL device 
can read/write from its own memory faster than the memory of the host system. 
Instead of applying the color map after an entire frame is computed,
it is applied during the computation of each element of the Julia Set. These 
operations can be seen in the main kernel code of my program:

<script src="https://gist.github.com/atkinsam/3c5adae01d8c4847261b9d55e27783b8.js"></script>

Note that in OpenCL's kernel language, complex numbers are not built-in. 
Type `Complex` is defined earlier in the kernel 
[source code](https://github.com/atkinsam/opencl-fractal-animation/blob/master/src/kernel.cl)
as a `double2`, a 2-dimensional vector double defined in the language. 
Both `c_add` and `c_multiply` are implementations of complex number arithmetic.

I also 
chose to not write to the disk after each frame's computation to reduce the
number of context switches in the operating system in this model. Instead, 
every frame is  kept in OpenCL buffers until all frames are finished. 
Afterward, the data is moved to host memory and subsequently written to the 
host disk in raw image format. 


## Speed Comparison 

In order to verify that there is a difference in speed for each of the 
programming models I used, I tested each of them using several different 
animations at different image resultions. The render times come from my
Thinkpad laptop equipped with an Intel i7-4600U CPU @ 2.10GHz and no
dedicated GPU. This plot represents the render time of a 300-frame animation
at different resolutions.

<img src="/images/chart.jpg" style="width: 100%"/>

Notice the logarithmic scale of the y-axis. Render times were so poor 
on the serial model that I had to change the axis scale in order to even
show the OpenCL parallel pixels running times. At the 500x500 pixel resolution,
the serial program rendered a video in an average 1044.09 seconds (over 17
minutes). At the same resolution, the parallel frames program took an average
of 140.39 seconds (2 minutes and 20 seconds) and the parallel pixels program
using the OpenCL framework rendered the same video in only 7.32 seconds.

## Conclusion 

When I started working on making fractal animations using 
Python, I had no idea my program was so slow. I accepted that it could take
hours to render some videos. Over the course of this experiment I realized
what kind of difference parallelization can make in computation. 
I also learned that there _is_ a right and wrong way to implement parallelization.
The parallel frame and parallel pixel models both use parallel programming,
but one is significatly faster than the other. Between context switches, buffer
versus host memory, and choosing an elementary operation, it is clear that
the implementation used to execute an algorithm can make an enormous difference
in runtime.
I hope to apply this in more practical uses in the future. 

If you would like to check out the source code for my OpenCL program, you can
find it on my [Github](https://github.com/atkinsam). If you find a flaw in my 
program, feel free to open 
an issue (or fork the repo and help me fix it!). 

If you would like to make your own animations, I have provided a few
examples of Julia Set frames to help you get started. 

## Examples

<img src="/images/examples.jpg" style="width: 100%"/>

__A__
* `size` = 400
* `center_re` = 0.0
* `center_im` = 0.0
* `zoom` = 1.0
* `c_re` = 0.0
* `c_im` = 0.64
* color map: brg.jpg 

__B__
* `size` = 400
* `center_re` = 0.49 
* `center_im` = 0.1
* `zoom` = 0.19 
* `c_re` = 0.26
* `c_im` = 0.0.001
* color map: gnuplut.jpg 

__C__
* `size` = 400
* `center_re` = 0.46
* `center_im` = 0.0
* `zoom` = 0.24
* `c_re` = 0.26
* `c_im` = 0.0
* color map: ocean.jpg 

__D__
* `size` = 400
* `center_re` = 0.2
* `center_im` = 0.4
* `zoom` = 0.2
* `c_re` = 0.0
* `c_im` = 0.642
* color map: gist_rainbow.jpg 

