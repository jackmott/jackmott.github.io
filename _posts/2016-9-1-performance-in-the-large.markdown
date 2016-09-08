---
layout: post
title:  "Taking out the garbage"
subtitle: "The cost of allocations"
date:   2016-9-1 14:17:27 -0500
categories: programming
---

One of the never ending arguments about languages is the pros and cons of garbage collection. Some people hate it because they think it is slow,
others insist it is fast, some hate that it takes away control from them, others love it for that same reason. I'm going to explore this a bit 
and show some pros and cons that arise, and how you can deal with them in C#, Java, and C++.  I will be creating basic framework for a game 
in the style of [Minecraft](https://minecraft.net/en/). Don't get too excited, there won't be any rendering or anything playable. Also, please
don't take any of these experiments to represent evidence of innate performance qualities of any of these languages. In all three cases, I am
aware of ways to optimize the code further, this is just meant to illustrate the relative costs of allocation and how you can start
reducing those costs in each language. If I get emails about fairness I'm going to refer you to this paragraph.

## The Naive Approach, in C Sharp

[Link To Gist](https://gist.github.com/jackmott/6d0ba936b24595402b49b6e76137c788)

Above is a link to how many developers might naively being to implement a game like Minecraft. (note, that experienced AAA game devs, their eyes would 
bleed at this) It is 3d, so of course you have the obligatory Vector class, with obligatory operator overloading so you can do simple 
operations on those vectors with very obvious code and very little typing. The game world consists of Chunks, that are loaded as you
approach close enough to them, and unloaded as you get too far away. Each Chunk has a number of Entities, that move around at various speeds each game tick. 
The player moves forward and every few ticks she passes into a new Chunk, which causes 1 Chunk to be
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

### Naive Approach 

You can see in the code we have a rudimentary frame rate lock, at 60fps, and on a modern Core I7 cpu we aren't hitting that frame rate ever! 
Some other troubling stats (collected with Perfmon) are clear:

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

[Link to Gist](https://gist.github.com/jackmott/fa4ca37bd372c1fe99f514bcf92519a0)

After a more serious refactoring:

- 0.1% of CPU time spent in garbage collection
- 1 MB/s allocations
- 19ms to load the world
- 0.98ms per tick

A huge difference! Very little time is spent on GC now, world load times are now instant and game ticks now process in under a millisecond.  At this
point almost nothing happens in a game tic except updating positions of entities, and occasional unloading of one chunk to be replaced by another.

### What Changed?

Lots of little things, using arrays instead of List when it doesn't cause any extra work saves a tiny bit of overhead.  Benefitting from
some array bounds elisions by structuring loops just right in some places.  Reducing GC pressure and improving runtime by not using *foreach* on Lists, and
other minor tweaks. But the main thing, was rethinking how the data is organized.  Previously, each chunk had 65k Block objects.  By thinking about the data,
one can figure out how many possible block types there are.  They will likely have some bound. In this case 256 was chosen to replicate Minecraft. You could 
easily bump that up to a short or int and still realize the bulk of this improvement. So instead of each chunk allocating and storing a complete Block
object 65k times, it just stores an index into a global array of Blocks. This is similar to how Minecraft actually does things.  This optimization
only makes sense if blocks are static things, most of the time, as they are in Minecraft.  You can break blocks, and place blocks, but they rarely
have state associated with them that changes.  This trick will not work with entities as currently designed, as they are moving around, their health is changing,
and so on.

This sort of thing is a very basic example of [Data Oriented Programming](https://dataorientedprogramming.wordpress.com/tag/mike-acton/). Think a little
bit more about what is actually happening with your data in memory, and less about what an idiomatic OOP design should be. Note that had you proceeded with 
the original design further into the development cycle, refactoring to make these changes could end up very very painful. Now that memory isn't being shuffled around 
like mad, there is plenty of CPU available to replace the mockup code with some real 'AI'.

One could go further with this, for instance pulling the Vector class out and replacing it with an array of positions, or even separate arrays of
x,y,and z values, depending on how the data is accessed could have big speed benefits due to cache locality and allow you to utilize SIMD instructions.  But that sort of madness is beyond the scope
of this blog post.


## The Naive Approach, Java

[Link to Gist](https://gist.github.com/jackmott/ddc406ecaf1cd9b86bb5a3dc2581cf28)

Recreating the same naive program in Java, I get the following stats (collected with Mission Control):

- 17% of CPU time spent in garbage collection
- 156 MB/s allocations
- 4.1s to load the world
- 4.9ms per tick


Java has no value types yet, so we can't apply the trick of making Vector a struct, but the memory use and GC time is already much better
than the .NET case where we made Vector a struct.  Part of the reason for this is that the JVM does escape analysis, so some of the wasteful
Vector allocations that are only being used within a function can be allocated on the stack automatically. Interestingly, escape analysis is
a feature coming soon to .NET, and value types are coming soon to Java. 

### Java refactored

[Link to Gist](https://gist.github.com/jackmott/21d8765608767f972b0f8b69a344d4cf)

But what if I refactor to avoid the wasteful allocations in the first place?

- .04% of CPU time spent in garbage collection
- 1.3 MB/s allocations
- 50ms to load the world
- 1.18ms per tick

The stats are now very much inline with the refactored .NET code.  While slightly worse, don't read too much into that, I'm not as experienced
at Java and probably have more obvious small mistakes. What should be noted here, is that avoiding wasteful allocations is important, no matter
the platform. One obvious way to further improve the Java code would be to eliminate the Vector class entirely, and just use float x,y,z in place
every where we use it.  This is a bit painful, but gets rid of the reliance on escape analysis and saves some object overhead. This would
be equivalent to converting the class to a value type, if/when Java has those. Another option is to use a pool of Vector objects, which you reuse.


## C++ Extremely naively

The first experiment I ran with C++ was to strictly copy the behavior of C# / Java, and create the same objects, on the heap, every time.
This was very unnatural, as creating a *new Vector* and then 2 lines of code later calling *delete* on it kind of alerts you to the 
absurdity of the situation. But for completeness I wrote that code and: 

- 0% of CPU time spent in garbage collection (but lots spent allocating!)
- 1 second to load the world
- 17.39ms per tick

While the game world loads a lot faster, the per tick performance is still really bad. Even worse than the naive
Java implementation, probably because there is no escape analysis saving us from allocating Vectors all over the place. This code is 
actually kind of absurdly naive though, it takes extra typing annoyance to make code this bad and it is pretty unlikely even 
someone not keen on performance would do this.  However this is how I was taught to do things with C++ in school, so you never
know.

### Minor refactor - no more heap allocating Vectors

[Link To Gist](https://gist.github.com/jackmott/a14abc480429a0aa494275b8e30e3511)

This is vaguely equivalent to making the Vector class a struct in C#.  I'm also passing Vector by value, and never allocating it on the heap. The
code gets smaller and simpler, I don't have to worry about memory management as much.

- 0% of CPU time spent in garbage collection (but some spent allocating)
- 737ms to load the world
- 1.4ms per tick


### Major refactor

[Link To Gist](https://gist.github.com/jackmott/f2fb4d967003f0ca18494ca2cb1e8fe0)

Applying the same tricks as we did in the other languages, so that we aren't allocating so many blocks:

- 0% of CPU time spent in garbage collection (a bit spent allocating)
- 13ms to load the world
- 1.17ms per tick



## Performance Comparisons:

A couple of quick graphs showing some performance comparisons. Again, I reiterate not to read these as proof of the performance superiority of any memory 
management approach. I assure you that any expert in any of these languages could optimize them further than they are here.

### The Naive Approaches (With Vector Fixes in C# and C++)

![Naive](/images/mem-slow.png "Naive")

Note the Y Axis here is Log Time. Notice how in all 3 languages performance starts to degrade at the same time, about 1,000 ticks in, probably reflecting when the heap begins to fill up,
and allocations get expensive for C++, and GC has to kick in more for .NET and Java. While one could bicker about which language is doing best here for eternity,
the fact is all three implementations are completely unacceptable.


### The Refactored Approaches

![Faster](/images/mem-fast.png "Faster")

Notice all 3 languages perform well here, but the garbage collected languages do still have GC pauses, which are a big problem in gaming.  Most game
developers would get even more clever, using object pooling and other techniques to try to get allocations down to zero within the main game loop, if possible.


## Conclusions

The main take away here is think about how you work with memory, no matter what language you use.  C# offers some nice tools in value types to make
this a bit easier.  Java on the other hand uses escape analysis to attempt to "auto struct" things for you. Both of these approaches have pros and cons.
It is less important to worry about which is best, and more important just to understand how your language works, so the code you type will leverage
it's strengths, and avoid it's weaknesses.  C++ doesn't totally solve this problem either, allocating too much is one of the primary causes of performance
problems in C++ code as well.  Manage your memory well.