---
layout: post
title:  "Performance In The Large"
subtitle: "Taking out the garbage"
date:   2016-9-1 14:17:27 -0500
categories: programming
---

Most of my blog posts so far have been with the performance characteristics of toy examples.  While fun and informative, these aren't
the real hard problems programmers face. If you really need to crunch some numbers in an array fast, you can methodically experiment
and find your way to a fast solution.  What is hard is performance of overall, larger systems.  You have to worry about the way
your functions interact with each how, how they interact with the hardware, how they interact with the memory model of your runtime, and
the operating system.  Performance issues can appears out of these interactions that don't exist in any single function.  Figuring out how
to make big systems perform well while still being understandable is the real hard problem in programming.  

I won't be solving that problem, I will just be exploring it, and showing some of these kinds of problems and how you can deal with them
in various languages. I will still use a toy example, but it will be a much bigger toy this time!  We will be creating mockup for a game,
vaguely replicating the wonderful [Minecraft](https://minecraft.net/en/)


## The Naive Approach, in C#

[Link To Gist](link)

Above is a link to how many developers might naively being to implement a game like Minecraft. (note, that experience AAA game devs, their eyes would 
bleed at this) It is 3d, so of course you have the obligatory Vector class, with obligatory operator overloading so you can do simple 
operations on those vectors with very obvious code and very little typing. The game world consists of Chunks, that are loaded as you
approach close enough to them, and unloaded as you get too far away. Each Chunk has a number of entities, that move around randomly
at various speeds each game tick.  The player moves forward and every few ticks he passes into a new Chunk, which causes 1 Chunk to be
loaded and another to be unloaded. Each chunk also has a number of blocks.  Everything in the game, Chunks, blocks, entities have positions
represented by a Vector class. Each tick of the game, the player moves forward a little bit. The chunks are iterated over and told to
update their entities, then checked to see if they have gone out of range, and removed if so.  If a chunk is removed, a new one is added to
replace it.

Simple enough, and the approach above would work fine for many games, and does work fine for many games! It isn't completely dumb, it uses
array backed List to keep track of things, because arrays are fast, and it preallocates them to the proper size when it can to avoid 
wasting memory and cpu on growing the array. Modern computers are fast, so this shouldn't be a problem!

But the problem is that while computers are fast, so too have our expectations grown.  A screen used to have 64,000 pixels max, now they
have 2 million at a minimum. A game world that you could explore for hours used to be impressive, but now players expect endless worlds
larger than planets, with detail down to blades of grass. And all of that has to happen at 90FPS on two monitors at once because VR! So, while our game 
is simple in principle, we load up and move around a *lot* of simple things. 65k blocks per chunk, 100 chunks at time, plus 1,000 entities 
per chunk all moving around every game tick.

### Naive Approach Performance

You can see in the code we have a rudimentary frame rate lock, at 60fps, and on a modern Core I7 cpu we aren't hitting that frame rate ever! 
Some other troubling stats are clear:

- 72% of CPU time spent in garbage collection!!!
- 98% CPU Utilization
- 700 megabytes of allocations per second
- 5.8 seconds to load the world.
- 39.4 ms per frame

I'm not even rendering yet! Or playing sounds, or doing networking. This is the kind of performance that causes people to say "Garbage collection is terrible
and slow!", which causes people to respond "No you just aren't using it right!", which then leads to "If I have to think about memory management anyway, what 
is the point of garbage collection!" and so on.

The root problem here, as is often the case, is too many allocations, which would be a problem even if there wasn't any garbage collector, though perhaps
not quite as bad.  I am casually creating new Vectors all the time for just a short while and then tossing them away. With blocks I am creating longer 
lived objects and then regularly tossing them to the garbage collector as well.  There are many things I can do to improve this, but C# has one feature
which is an 'easy fix', and that is structs, which are a value type.  They are not allocated on the heap, and they are passed by value.  You can't
just turn all of your classes into structs, as larger classes being copied around by value would be wasteful, but small ones you can.  In this case, 
Vector is a perfect candidate, at only 12 bytes.  

### Change class Vector to struct Vector

Just six characters and look at the difference:

- 45 of CPU time spent in garbage collection
- 50% CPU utilization
- 200 megabytes of allocations per second
- 4.6 seconds to load the world.
- 4.9ms per frame

Suddenly things have gone from a hopeless situation where there is negative time for the frame rate budget for rendering, to having 
11 milliseconds and 50% of the CPU to spare. But the struct is just a partial fix.  Next I will refactor the code a bit for even better performance.

### Refactor

[Link to Gist](link)

After a more serious refactoring:

- 0.16% of CPU time spent in garbage collection
- 24% CPU Utilization
- 1 megabyte of allocations per second
- 4.6 seconds to load the world.
- 4.2ms per frame
- 100ms to load the world

A huge difference! Very little time is spent on GC now, world load times are now instant. Notice something interesting though, if one
had been looking *only* at the time per frame, they would have thought this refactor had made very little difference. But in reality the total CPU
use cut in half. The fact that the code sleeps to lock the frame rate at 60fps was hiding the cost of the GC, but as rendering and networking
get added to this framework, CPU resources would have run out quickly.

### What Changed?

Lots of little things, using arrays instead of List when it doesn't cause any extra work saves a tiny bit of overhead.  Benefitting from
some array bounds elisions by structuring loops just right in some places.  Reducing GC pressure and improving runtime by not using *foreach* on Lists, and
other minor tweaks. But the main thing, was rethinking how the data is organized.  Previously, each chunk had 65k Block objects.  By thinking about the data,
one can figure out how many possible block types there are.  They will likely have some bound. In this case 256 was chosen to replicate Minecraft. You could 
easily bump that up to a short or int and still realize the bulk of this improvement. So instead of each chunk allocating and storing a complete Block
object 65k times, it just stores an index into a global array of Blocks. This is similar to how Minecraft actually does things.

This sort of thing is a very basic example of [Data Oriented Programming](https://dataorientedprogramming.wordpress.com/tag/mike-acton/). Think a little
bit more about what is actually happening with your data in memory, and less about what an idiomatic OOP design should be. 

One could go further with this, for instance pulling the Vector class out and replacing it with an array of positions, or even separate arrays of
x,y,and z values, depending on how the data is accessed could have big speed benefits due to cache locality.  But that sort of madness is beyond the scope
of this blog post.



