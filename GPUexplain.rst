﻿.. highlight:: python
   :linenothreshold: 25

3D Graphics Explanation
=======================

This is a short introduction to 3D graphics from the perspective of pi3d,
there will be gaps and possibly misapprehensions but it should give you a
reasonable feeling for how things work! Also I intentionally
skip over many of the more involved aspects such as rendering to off-screen
buffers, using masks, etc.

Beneath the python module classes and functions provided by pi3d there are
three "steps" necessary for control of the GPU. Two or three of these
require external libraries (shared objects in linux, dlls in windows) to be
imported [#]_:

  1. on the Raspberry Pi ``libbcm_host.so`` is used to create and manage a display
  surface to draw on. On linux (desktop and laptop) the surface is provided
  by the x11 server [#]_ and windows and Android use pygame (which uses SDL2 [#]_)

  2. ``libELG`` is used to set up the interface between the machine or operating
  system window system and

  3. ``libGLESv2`` provides access to the OpenGL language functions developed
  to standardise utilisation of graphics cards. Mobile devices, including
  the Raspberry pi use a slightly cut-down version called OpenGL ES, specifically
  version 2.0.

From OpenGL ESv2.0 onwards the fundamental graphics donkey work is done by
'shaders' that are defined by the developer and compiled as the program
runs rather than being 'built into' the GPU. This opens up a fantastic
range of possibilities but there are some fundamental limits that may not
be immediately apparent.

Communication between python and the GPU
----------------------------------------

There are two parts to a shader: the vertex shader and the fragment (essentially
pixel) shader which are written in a C like language (GLSL). I will give
some more detail to what each actually does later but one crucial thing to
appreciate is that information is passed from the CPU program (in
our case python pi3d ones) to the shaders and the vertex shader can pass
information on to the fragment shader, however the only output is pixels [#]_.
It is fundamental to the efficiency and speed of the GPU that the shaders
operate on only one vertex or pixel. i.e. the vertex shader can't access
information about the location of adjacent vertices and the fragment shader
can't determine the colour, say, of adjacent pixels. This allows the processing
to be run in parallel (massively parallel, some GPU have thousands of
processing cores) but means that some operation such as blurring or edge
detection have to be done as a double pass process.

Information needed to render the scene is passed to the shader in four
distinct blocks:

1.  *An 'element array'* that will be drawn by the call to the
    drawElements function. This function can be used to draw polygons (limited to
    triangles in OpenGL ES2.0), lines or points, and the type of drawing will
    determine how the entries in the array are interpreted. Essentially
    each element will contain reference indices to one or more vertices. In the
    simple square example below this is the ``triangle indices`` array.

2.  *An 'attribute array'* of vertex information, again the type
    of drawing determining how much information needs to be passed. For the
    most general 3D drawing in pi3d the array contains vertex x,y,z values,
    normal vectors and texture coordinates.

3.  *'uniform' variables*. This includes things
    that apply to all the vertices being drawn, such as the transformation matrix
    (for the shape to which the vertices belong), the projection matrix to
    represent camera location and field of view, the location and colour of
    light sources, fog properties, material shades and transparency,
    variables to control pattern repeats or for moving patterns etc.

    A very significant part of the uniform variables are images or texture
    samplers to 'clothe' the object or to provide information on bumps or
    reflections.

4.  *The program* for the GPU to run, comprising the vertex
    and fragment shader.

In pi3d these four categories of information are held in various objects:
The element and attribute arrays are part of the Buffer and the Shader class
contains the shader programs. However the uniform variables are held in
Buffer, Shape, Camera, Light and Texture objects as seemed logical and
appropriate. General window information, EGL and OpenGL functionality are
held in pi3d globals or the Display object.

NB other 3D graphics frameworks pass essentially identical information to
the GPU but use different terminology. So in threejs there are: Scene,
PerspectiveCamera, WebGLRenderer, Mesh, Geometry, Material etc.

Sequence of events
------------------

3D objects are defined for use in graphics programs starting with a
list of points or vertices in space each one needing x, y, z coordinates.
Although not generally essential, in pi3d each vertex has a normal vector
defined as well. This is effectively an arrow at right angles to the surface
at that point and it also needs three values to define its magnitude in
the x, y, z directions. The normal vector can be used by the shader to
work out how light would illuminate a surface or how reflections would
appear. If the normals at each corner of a triangular face are all pointing
in the same direction then the fragment shader will treat the surface as
flat, but if they are in different directions the surface will appear to
blend smoothly from one direction to another. 3D models created in
applications such as blender normally have an option to set faces to look
either angular or smoothed by calculating different types of normal vectors.
Each vertex also has two texture coordinates. These are often
termed the u, v position from a two dimensional texture that is to be mapped
to that vertex. Again the fragment shader can interpolate points on a surface
between vertices and look up what part of a texture to render at each pixel.
The crucial piece of information needed by the shader is to define which
vertices to use for the corners of each triangle or element. So if I use as an example
a very simple one sided square this could be defined by the attribute array::

  """   vertices     |    normals     |  texture
                     |                |  coords
       x    y    z   |  x    y    z   |  u    v
  """
  attribute_array = numpy.array(
    [[0.0, 0.0, 0.0,   0.0, 0.0,-1.0,   0.0, 0.0],   # 0
     [0.0, 1.0, 0.0,   0.0, 0.0,-1.0,   0.0, 1.0],   # 1
     [1.0, 1.0, 0.0,   0.0, 0.0,-1.0,   1.0, 1.0],   # 2
     [1.0, 0.0, 0.0,   0.0, 0.0,-1.0,   1.0, 0.0]])  # 3

and the element array of
triangle indices::

  element_array = numpy.array(
    [[0, 1, 2],
     [0, 2, 3]])

Note the order of corners
is important. Each triangle 'faces' towards a view where the sequence
is clock-wise. Normally the backs of faces are not rendered by the GPU.

The GPU uses coordinate directions x increases from left to right, y
increases from bottom to top, z increases going into the screen.

.. image:: images/simple_quad.png
   :align: left

The GPU has been designed to be fantastically efficient at performing
vector and matrix arithmetic. So rather than the CPU calculating where
the vertices have moved and how these positions can be represented
on the 2D computer screen it simply calculates a transformation matrix
to represent this and passes that to the GPU. In pi3d we pass two matrices,
one representing the object translation, rotation and scale and an additional
one including the camera movement and perspective calculations [#]_. In the
vertex shader these matrices are used to convert the raw vertex positions
to screen locations and to work out where the light should come from in
order to work out shadows.
  
Image files are converted into texture arrays that are accessed
very efficiently by the GPU.

When pi3d.Buffer.draw() method is called for a 3D object the python side
of the program sets the shader and necessary uniform variables to draw the
given object. It then works out the 4x4 matrix combining translation, rotation,
scale for the object and an additional matrix incorporating the camera
movement and lens settings. The camera has two basic modes for handling
perspective, the default is 'normal' where things further away are represented
as smaller on the screen and the this is defined by a viewing angle between
the top edge of the screen and bottom edge. If the camera is set to
orthographic mode then objects do not get smaller in the distance and one
unit of object dimension corresponds to a pixel on the screen. An orthographic
camera can be used to do fast 2D drawing.

The glDrawElements function is then called  which sets the vertex shader
to work out the locations of each vertex, normal, lighting, texture in
terms of screen coordinates. The vertex shader then passes the relevant
information to the fragment shader which  calculates what colour and alpha
value to use for each pixel. The fragment shader takes into account the
depth value of each pixel and doesn't draw anything that is behind something
it has already drawn. This means that it is more efficient to draw opaque
objects from near to far but if something is partially transparent then
is must be drawn **after** anything further away that should 'show through'.

pi3d uses a double buffer system where everything is drawn onto an off-screen
buffer which, when complete at the end of the frame loop, is swapped
'instantaneously' to visible. This makes the animation much smoother

.. [#] The attempt to work out on what platform pi3d is running and what
   libraries to import is done in /pi3d/constants/__init__.py and the
   Initialization is done in /pi3d/utils/DisplayOpenGL.py
.. [#] X11 is the standard windowing and user-input system used on Linux
   systems
.. [#] Simple DirectMedia Layer https://www.libsdl.org/index.php
.. [#] It is possible to get 'output' from GPUs using sophisticated techniques
   that allow the parallel processing capabilities to be used elsewhere, but
   this is not trivial!
.. [#] There are actually three 4x4 matrices, the last of which is used
   for creating a "distance map" for calculating shadow casting (see the
   CastShadows.py and TigerShadow.py demos) Quite a technical procedure.
