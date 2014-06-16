---
layout: post
title: "Klay: A post-mortem"
---

Klay lets you create 3D sculptures in seconds using your iPad. I finished it yesterday after years of maybe-sometimes-getting-some-work-done.

Here is a post-mortem of the fun bits of the app.

# Motivation

3D modeling & animation is tedious and unpleasant. Programming is also tedious and unpleasant, but not nearly as much, and you get to use your brain (when you are not busy gluing shit together, which might become your whole job if you are not careful).

When I was a teen, I was really into 3D modeling and animation. I created some stuff, but I was mostly showing signs of real OCD by compulsively learning tools. (Maya, XSI, Lightwave, Premiere, Photoshop, After Effects... It was a huge waste of time because I "learned" more than I was creating). [Takeo Igarashi](http://www-ui.is.s.u-tokyo.ac.jp/~takeo/teddy/teddy.htm) is a brilliant researcher, and my tool-learning compulsion let me to find his early work: T.E.D.D.Y. -- A buggy Java program with which you could make 3D shapes from 2D strokes (simple closed curves, please). It was many years later that my subconscious dug out TEDDY and thought "This would be a cool iPad app." ...It took me about 2.5 years to get a finished app. In my defense, most of that time was spent not working on Klay.

At its core, Klay is a re-implementation of T.E.D.D.Y. Ignoring all mesh-editing features. On top of that, there is social stuff -- email, facebook -- and a very pleasant touch focused user experience. It is very simple -- just create meshes, with undo and redo. This makes it very easy for kids to learn it. They love it.

So... moving on:

# Meshgen

2D Meshes are represented using [Half-Edges](http://en.wikipedia.org/wiki/Doubly_connected_edge_list). 3D meshes are vanilla arrays-of-vertices-and-indices. We generate 3D meshes from the 2D mesh and a height map. There was never a need to be fancy with 3D.

Klay's mesh generation is this pipeline:

[2D points] -> Sanitize -> Constrained Delaunay (aka CDT) -> Add internal points -> Re-CDT -> Get 3d mesh

Old video, but pretty close to the final thing:

<iframe width="560" height="315" src="//www.youtube.com/embed/RL9wgP4O-_4" frameborder="0" allowfullscreen></iframe>

Let me get to it point by point.

1. Sanitize

    A "stroke" (that's what a drawing intending to create a 3D mesh is called) is an array of points in 2D. Sanitizing makes sure that there are no "fatal" self-intersections, and it does its best to spit out a set of 2d points that will result in a 3D mesh. That why I used the word fatal -- It is possible to self-intersect and still get a mesh.

2. CDT

    The Constrained Delaunay Triangulation is a [Delaunay Triangulation](http://en.wikipedia.org/wiki/Delaunay_triangulation) where some edges are specified to be included even if they don't meet the Delaunay criterion -- No point in the triangulation is inside the circumcircle of any triangle.

    <img src="http://127.0.0.1:4000/img/klaypm_1.png" style="width: 600px;"/>

    When applied to the sanitized points, we get a zig-zaggy-triangulation.

3. Add internal points

    We classify the triangles in three categories. "Joint" triangles are triangles that have all edges inside. "Sleeve" triangles have one edge facing the outside and two edges inside. "Terminal" triangles have two edges facing outside. I deviate from the TEDDY paper by only subdividing Sleeve triangles. It works wells enough and it means less code; I'll say more about this later.

    The inside edges of the Sleeve triangles are subdivided by inserting N interior points along each edge (N is larger if the edge is longer). We also calculate the height of each point -- The points are collinear, so we can map them onto the X axis and get the height from an ellipse equation.

    The resulting triangulation is not always pretty, but the shading style doesn't require a pretty triangulation. If I were to add realistic shading (or add a feature to export meshes), then I would prettify the 3D mesh *after* the fact, when we have more information and can make better decisions. For now, the triangulation is "pretty enough".

4. Re-CDT

    So now we have the original sanitized stroke, the new interior points from the Sleeve triangles and a height map. The next step, which is also a deviation from the TEDDY paper, which uses a different method of triangulation, is to re-run the CDT function, with the same edge constraints but with new interior points.

    <img src="http://127.0.0.1:4000/img/klaypm_2.png" style="width: 600px;"/>

    Cool note: this is a decent UV-map of half of the finished 3D mesh ;)

4. Get 3D Mesh

    It is very simple:
    1. Duplicate all interior points
    2. Edge points get z=0
    3. Interior points get the corresponding value from the pre-computed height map. The duplicates get the same value, multiplied by -1.
    4. Generate an array with all vertices, and an index array that we can use with GL to draw.

# Style

From early-on, I wanted a non-photorealistic style. I ended up with a special toon shader:

<img src="http://127.0.0.1:4000/img/klaypm_3.png" style="width: 600px;"/>

The shader is an old-school Phong shader with thresholds set to decrease lightness. The black outline is a check for 0 screen-space normals with an epsilon value for thickness.

# Multiple platforms. Optimizations. Orthogonal directions.

Everything that the hardware doesn't implement is written from scratch (except for all the iOS stuff like facebook sharing and email :) )

This was always a hobby project. I kept it portable and running across a number of platforms at all times. Not all platforms worked all the time, but there were always a couple. The game runs on Linux, OSX, Windows and of course, iOS -- The main target. I ported the code to the Android NDK and got to render a triangle, but I never continued. Android is too much risk for me. I saw a guy distribute a very simple game written in Unity that would crash on my (mainstream) Nexus 7. Android is too fragmented to maintain a hand-made game.

I see development as a hill-climbing problem, at the maximum there is a final product. I learned a lot, but the lack of a goal made me go in orthogonal directions. For instance, I spent a lot of time optimizing my linear algebra library so that some operation or other would only take one or two CPU instructions. It was fun, but it was x86... My final target was ARM, where none of my optimizations were enabled and ultimately, linear algebra is *not* the bottleneck in any part of the program.

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

After I decided to go indie, I sat down and crunched through the last 10%. It was a month of steady work. Not much fun. I had to make a Menu system, create backgrounds, integrate the app with Facebook and make sure that everything worked in different Apple devices (I have an old iPad 2 and a new, sexy, iPad Air.) There was also some bug fixing :).

<img src="http://127.0.0.1:4000/img/klaypm_4.png" style="width: 600px;"/>

# The way forward

Klay should be out soon, and hopefully it will sell well! No matter how it does, I will keep on the path I set on to two months ago.

It's back to creative time. It is not clear what I am going to do, but some things are certain: It is going to be a game, it is going to be exclusively for VR, and it will revolve around the manipulation of gravity, space and light.
