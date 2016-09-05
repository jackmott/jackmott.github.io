---
layout: post
title:  "Performance In The Large"
subtitle: "Taking out the garbage"
date:   2017-9-1 14:17:27 -0500
categories: programming
---

Most of my blog posts so far have been with the performance characteristics small number crunching examples.  While fun and informative, these aren't
the real hard problems programmers face. If you really need to crunch some numbers in an array fast, you can methodically experiment
and find your way to a fast solution.  What is hard is performance of overall, larger systems.  You have to worry about the way
your functions interact with each other, how they interact with the hardware, how they interact with the memory model of your runtime, and
the operating system.  Performance issues can appear out of these interactions that don't exist in any single function.  Figuring out how
to make big systems perform well while still being understandable is the real hard problem in programming.  

I won't be solving that problem, I will just be exploring it, and showing some of these kinds of problems and how you can deal with them
in various languages. I will still use larger toy example this time. I will be creating basic framework for a game in the style of [Minecraft](https://minecraft.net/en/)
Don't get too excited, there won't be any rendering or anything playable.

## The Naive Approach, in C#

[Link To Gist](link)

Above is a link to how many developers might naively being to implement a game like Minecraft. (note, that experienced AAA game devs, their eyes would 
bleed at this) It is 3d, so of course you have the obligatory Vector class, with obligatory operator overloading so you can do simple 
operations on those vectors with very obvious code and very little typing. The game world consists of Chunks, that are loaded as you
approach close enough to them, and unloaded as you get too far away. Each Chunk has a number of Entities, that move around randomly
at various speeds each game tick.  The player moves forward and every few ticks she passes into a new Chunk, which causes 1 Chunk to be
loaded and another to be unloaded. Each Chunk also has a number of Blocks.  Everything in the game, Chunks, Blocks, Entities have positions
represented by a Vector class. Each tick of the game, as the player moves forward a little bit, the chunks are iterated over and told to
update their entities, then checked to see if they have gone out of range, and removed if so.  If a chunk is removed, a new one is added to
replace it.

Simple enough, and the approach above would and does work fine for many games. It isn't completely dumb, it uses an
array backed List to keep track of things, because arrays are fast, and it pre-allocates them to the proper size when it can to avoid 
wasting memory and cpu on growing the array. Modern computers are fast, so this shouldn't be a problem!

But the problem is that while computers are fast, so too have our expectations grown.  A screen used to have 64,000 pixels max, now they
have 2 million at a minimum. A game world that you could explore for hours used to be impressive, but now players expect endless worlds
larger than planets, with detail down to blades of grass. And all of that has to happen at 90FPS on two monitors at once because VR! So, while our game 
is simple in principle, we load up and move around a *lot* of simple things. 65k blocks per chunk, 100 chunks at time, plus 1,000 entities 
per chunk all moving around every game tick.

### Naive Approach Performance

You can see in the code we have a rudimentary frame rate lock, at 60fps, and on a modern Core I7 cpu we aren't hitting that frame rate ever! 
Some other troubling stats are clear:

- 80% of CPU time spent in garbage collection!!!
- 600 MB/s allocations
- 5.8 seconds to load the world.
- 31.4 ms per tick

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

- 45% of CPU time spent in garbage collection
- 200 MB/s allocations
- 4.4 seconds to load the world.
- 4.1ms per tick


Suddenly things have gone from a hopeless situation where there is negative time for the frame rate budget for rendering, to having 
11 milliseconds and 50% of the CPU to spare. But the struct is just a partial fix.  Next I will refactor the code a bit for even better performance.

### Refactor

[Link to Gist](link)

After a more serious refactoring:

- 0.1% of CPU time spent in garbage collection
- 1 MB/s allocations
- 100ms to load the world
- 0.3ms per tick


A huge difference! Very little time is spent on GC now, world load times are now instant and game ticks now process in under a millisecond.  At this
point almost nothing happens in a game tic except updating positions of entities, and occasional unloading of one chunk to be replaced by another.

### What Changed?

Lots of little things, using arrays instead of List when it doesn't cause any extra work saves a tiny bit of overhead.  Benefitting from
some array bounds elisions by structuring loops just right in some places.  Reducing GC pressure and improving runtime by not using *foreach* on Lists, and
other minor tweaks. But the main thing, was rethinking how the data is organized.  Previously, each chunk had 65k Block objects.  By thinking about the data,
one can figure out how many possible block types there are.  They will likely have some bound. In this case 256 was chosen to replicate Minecraft. You could 
easily bump that up to a short or int and still realize the bulk of this improvement. So instead of each chunk allocating and storing a complete Block
object 65k times, it just stores an index into a global array of Blocks. This is similar to how Minecraft actually does things.  This optimization
only makes sense if blocks are static things, most of the time, as they are in minecraft.  You can break blocks, and place blocks, but they rarely
have state associated with them that changes.  This trick will not work with entities as currently designed, as they are moving around, their health is changing,
and so on.

This sort of thing is a very basic example of [Data Oriented Programming](https://dataorientedprogramming.wordpress.com/tag/mike-acton/). Think a little
bit more about what is actually happening with your data in memory, and less about what an idiomatic OOP design should be. Note that had you proceeded with 
the original time further into the development cycle, refactoring to make these changes could end up very very painful.  Also note that with
this refactored design, almost nothing is happening most of the time, except changing the positions of the entities, and occasionally swapping out one chunk for 
another.  That is exactly the goal, have almost nothing happen. Now that memory isn't being shuffled around like mad, there is plenty of CPU available to replace
the mockup code with some real 'AI'.

One could go further with this, for instance pulling the Vector class out and replacing it with an array of positions, or even separate arrays of
x,y,and z values, depending on how the data is accessed could have big speed benefits due to cache locality and allow you to utilize SIMD instructions.  But that sort of madness is beyond the scope
of this blog post.


## The Naive Approach, Java

[Link to Gist](link)

Recreating the same naive program in Java, I get the following stats:

- 17% of CPU time spent in garbage collection
- 156 MB/s allocations
- 4.1s to load the world
- 4.9ms per tick


Java has no value types yet, so we can't apply the trick of making Vector a struct, but the memory use and GC time is already a bit better
than the .NET case where we made Vector a struct.  Part of the reason for this is that the JVM does escape analysis, so some of the wasteful
Vector allocations that are only being used within a functions can be allocated on the stack automatically. Interestingly, escape analysis is
a feature coming soon to .NET, and value types are coming soon to Java.  Also notice while the memory stats here are impressive, the time it takes
to complete one tick of the game is longer the .NET case with structs. 

### Java refactored

[Link to Gist](link)

But what if I refactor to avoid the wasteful allocations in the first place?

- .04% of CPU time spent in garbage collection
- 1.3 MB/s allocations
- 125ms to load the world
- 1.18ms per tick

The stats are now very much inline with the refactored .NET code.  While slightly worse, don't read too much into that, I'm not as experienced
at Java and probably have more obvious small mistakes. What should be noted here, is that avoiding wasteful allocations is important, no matter
the platform.


## C++ Extremely naively

[Link to Gist](link)

This is extremely naive in part because almost nobody would be heap allocating the vectors the way I am here, it just isn't a natural
thing to do in C++, but just to explore the cost of allocation, even with no GC, I'll do it.  It is also extremely naive because I'm
not very experienced at C++. So who knows what horrible choices I am making!

- 0% of CPU time spent in garbage collection (but lots spent allocating!)
- 1 second to load the world
- 17.39ms per tick

While the game world loads a lot faster, the per tick performance is still really bad. Even worse than the naive
Java implementation, probably because there is no escape analysis saving us from allocating Vectors all over the place. This code is 
actually kind of absurdly naive though, it takes extra typing annoyance to make code this bad and it is pretty unlikely even 
someone not keen on performance would do this. The fact that you have to type "new Vector" them immediately "delete Vector" tends
to help clue you in to the absurdity of what you are doing.  However this is how I was taught to do things with C++ in school, so you never
know.

### Minor refactor - no more heap allocating Vectors

This is vaguely equivalent to making the Vector class a struct in C#.  I'm also passing Vector by value, and never allocating it on the heap. The
code gets smaller and simpler, I don't have to worry about memory management as much.

-0% of CPU time spent in garbage collection (but some spent allocating)
-737ms to load the world
-1.7ms per tick

Interesting here is that while C++ is using less CPU, it also takes more time per tick.  Remember that every game tick, the thread sleeps if it has finished in 
time for the 60fps budget.  The JVM can spend some of that time doing GC.  So the C++ code here has more room to grow. But there is still a lot of 
wasteful memory allocation here, with no JVM or .NET tricks to fix it.

### Major refactor

-0% of CPU time spent in garbage collection (a bit spent allocating)
-163ms to load the world
-1.7ms per tick

