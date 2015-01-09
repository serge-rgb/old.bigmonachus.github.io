---
layout: post
categories: [devlog]
title: "Klay: A post-mortem"
---

Klay lets you create 3D sculptures in seconds using your iPad. I finished it yesterday after years of maybe-sometimes-getting-some-work-done.

Old video, but pretty close to the final thing:

<iframe width="560" height="315" src="//www.youtube.com/embed/RL9wgP4O-_4" frameborder="0" allowfullscreen></iframe>

Here is a post-mortem of the fun bits of the app.

# Motivation

3D modeling & animation is tedious and unpleasant. Programming is also tedious and unpleasant, but not nearly as much, and you get to use your brain!.

I used to be very much into 3D modeling and animation. I did some stuff with Maya, XSI, Lightwave and so on. Takeo Igarashi](http://www-ui.is.s.u-tokyo.ac.jp/~takeo) is a brilliant researcher, and my tool-learning compulsion let me to find his early work: T.E.D.D.Y. -- A Java program with which you could make 3D shapes from 2D strokes (simple closed curves, please). It was many years later that my subconscious dug out TEDDY and thought "This would be a cool iPad app."

# What is it?

At its core, Klay is a re-implementation of T.E.D.D.Y. without mesh-editing features. On top of that, it has social stuff -- email, facebook -- and a very pleasant touch focused user experience. It is a simple app -- just create blobs on top of other blobs, with undo and redo. This makes it very easy for kids to learn it. They love it.

# Meshgen

2D Meshes are represented using [Half-Edges](http://en.wikipedia.org/wiki/Doubly_connected_edge_list). 3D meshes are vanilla arrays-of-vertices-and-indices.

After the user finishes a gesture, the algorithm receives a set of 2D-points in screen-space. If everything goes alright, the pipeline outputs a 3D mesh in world-space.

_This is the pipeling of Klay's mesh generation:_

1. Sanitize

    A "stroke" is an array of points in 2D. Sanitizing makes sure that there are no fatal self-intersections, and it does its best to spit out a set of 2d points that will result in a 3D mesh. It is possible to self-intersect and still get a mesh. The process of sanitizing is to make an effort to output a closed simple curve from what is _almost_ a closed simple curve.

2. CDT

    The Constrained Delaunay Triangulation is a [Delaunay Triangulation](http://en.wikipedia.org/wiki/Delaunay_triangulation) where some edges are specified to be included even if they don't meet the Delaunay criterion.
When applied to the sanitized points, we get a zig-zaggy-triangulation.

    <img src="http://bigmonachus.org/img/klaypm_1.png" style="width: 600px;"/>

3. Add internal points

    Having a silluhette curve, we can determine if points are inside or outside the mesh.

    We classify the triangles into three categories. _Joint_ triangles are triangles that have all edges inside of the mesh. _Sleeve_ triangles have one edge facing the outside and two edges inside. _Terminal_ triangles have two edges facing outside.

    The inside edges of the _Sleeve_ triangles are subdivided by inserting N interior points along each edge (N  is proportional to the length of the edge). These N new points are colinear, so we project them to the X-axis and use an ellipse equation, determined by the length of the edge, to assign heights to each new point.

    We keep the heights in an array, and use the 2D points to create a new triangulation. The Delaunay triangulation is constrained to not modify silluhette edges.


4. Re-CDT

    The resulting triangulation is not always pretty, but it works for our shading style.

    <img src="http://bigmonachus.org/img/klaypm_2.png" style="width: 600px;"/>

    Cool note: this is a decent UV-map of half of the finished 3D mesh ;)

    As you can see, triangles vary a lot in size, and that is a Bad Thing. However, if in the future I were to use a different shading style that required prettier meshes, it would be much better to fix them in the 3D stage, not the triangulation stage.

4. Get 3D Mesh


    A 3D mesh is obtained from the 2D mesh by adding a z-component to each vertex.

    The procecdure is this:

    1. Duplicate all interior points
    2. Edge points get z=0
    3. The original interior points get their corresponding z-value from the pre-computed height map from step 3. The duplicated interior points are treated the same, but their z-values go in the opposite direction.
    4. Using the connectivity information from the 2D points, we put the 3D points into a vertices and indices structure that is easy to feed to OpenGL

# Style

From early-on, I wanted a non-photorealistic style. I ended up with a special toon shader:

<img src="http://bigmonachus.org/img/klaypm_3.png" style="width: 600px;"/>

The shader is an old-school Phong shader with thresholds set to decrease lightness. The black outline is a check for 0 screen-space normals with an epsilon value for thickness.

# Multiple platforms. Optimizations. Orthogonal directions.

Everything that the hardware doesn't implement is written from scratch (except for all the iOS stuff like facebook sharing and email :) )

This was always a hobby project. I kept it portable and running across a number of platforms at all times. Not all platforms worked all the time, but there were always a couple. The game runs on Linux, OSX, Windows and the target platform: iOS. I did a little bit of work with the Android NDK but I never got past the "draw a triangle" stage.

I see development as a hill-climbing problem, at the maximum there is a final product. I learned a lot, but the lack of a goal made me go in not optimal directions. For instance, I spent a lot of time optimizing my linear algebra library so that additions and multiplications would take one or two SIMD instructions. It was fun, but it was x86... My final target was ARM, where none of my optimizations were enabled. Ultimately, linear algebra is *not* the bottleneck in any part of the program.

Some optimizations were good.

I designed the renderer to avoid OpenGL state changes, which is a good thing to do on all platforms.

I "discovered" a trick to avoid dynamic branches in shaders that allowed the game to run at 60hz on the iPad 2 (it really made the difference). It is obviously one of the first optimization tricks you find if you google hard enough.

The trick is to turn this:

    if (x < 0.5) {
        color = this;
    } else {
        color = that;
    }

.. into this:

    float is_less = float(x < 0.5);
    color = is_less * this + (1 - is_less) * that;

I could have used [@aras_p](http://twitter.com/aras_p)'s shader optimizer, but optimizing was fun, and for most of the development, I just had fun. Because that is what it was, a hobby project.

Keeping it running on multiple platforms was good because it made me find bugs -- Especially with driver issues.

# Focusing

After I decided to ship this as a product, I sat down and crunched through the last 10%. It was a month of steady hard work. Not much fun. I had to make a Menu system, create backgrounds, integrate the app with Facebook and make sure that everything worked with different Apple devices (I have an old iPad 2 and a new, sexy, iPad Air.) There was also some bug fixing :).

<img src="http://bigmonachus.org/img/klaypm_4.png" style="width: 600px;"/>

