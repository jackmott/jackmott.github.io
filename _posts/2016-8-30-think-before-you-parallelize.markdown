---
layout: post
title:  "Think Before You Parallelize"
subtitle: "A tour of parallel abstractions"
date:   2016-8-30 14:17:27 -0500
categories: programming
---

In 2005 Intel release the Pentium D, which began the era of multi-core desktop CPUs.  Today, even our phones have multiple cores. Making use
of all of those cores it not always easy to do, but modern languages and libraries have come a long way to help programmers take advantage.
All kinds of utility functions and concurrent abstraction have been developed in an attempt to make using all our cores more accessible 
and simple. Sometimes these abstractions have a lot of overhead though, and sometimes it doesn't even make sense to parallelize an operation 
in the first place.

## Don't Parallelize when it is already Parallelized

Suppose you are working on a big number crunching function for a high traffic website that is a bit of a performance bottleneck. You get the idea to
parallelize it and it tests much faster on your dev machine with 4 cores. You expect great things on the 24 core production server. However
once you deploy you find that performance in production is actually slightly worse!  What you forgot was that the web server was already parallelizing 
things at a higher level, using all 24 production cores to handle multiple requests simultaneously.  When your paralellized function
fires up, all the other cores are busy with other requests.  So you take the hit of whatever overhead was required to parallelize the function
with no benefit. 

On the other hand, if your website was say, a low traffic internal website with only a few dozen hits per day, then the plan to parallelize would
likely pay off, as there will always be spare cores to crunch the numbers fast. You have to consider the overall CPU utilization of your webserver,
and how your parellelized function will interact with the other jobs going on. Will it thrash the L1 cache and slow other things down? Test and
measure.

Another scenario, say you are working on 3D game, you have some trick physics math where you need to crunch numbers, maybe adding realistic building
physics to Minecraft.  But separate threads are already handling procedural generation of new chunks, rendering, networking, and player input. If
these are keeping most of the system's cores busy, then parallelizing your physics code isn't going to help overall. On the other hand if those
other threads are not doing a lot of work, cores may indeed be free for you to crunch some physics.

So think about the system your code is running in, if things are getting parallelized at a higher level, it may not do any good to do it
again at a lower level. Instead, focus on algorithms that run as efficiently as possible on a single core.

## Consider your target hardware

Many developers have very nice machines, probably with a minimum of 8 logical cores these days, possibly more.  But consider the entire scope
of where your code might run.  Will it run on a low cost virtualized web app fabric in the cloud?  This may only have 1 or 2 virtual cores for
you to work with.  Will it run on old desktops or cheap phones, that maybe only have 2 cores?  An algorithm that gets sped up on your 8 core system
at home may not fair so well on systems with only 2 or 3.

## Case Study Easy Parallel Loops 

It is common in a given programming language to have compiler hints or library functions for doing easy parallel loops when it is appropriate. What happens
behind the scenes can be very different depending on the abstractions each language or library uses.  In some cases a number of threads may be created to 
operate on chunks of the loop, or ThreadPools may be used to reduce the overhead of creating Threads. It is important to have a rough understanding of how 
the abstractions available to you work, so you can make educated guesses about when it might be useful to use them, how to tune them, and how to measure them.
At minimum you should consider the following issues.

If the computational overhead of creating or managing the thread is greater than the benefit you get, you can end up with a slower results
than you would doing a single threaded implementation. I will compare some toy workloads with some common parallel loop abstractions 
in C#, F#, C++ and Java. 

### CSharp

``` c#   
    
    public double ImperativeSquareSum()
    {
        var localArray = rawArray;               
        double result = 0.0;
        for (int i = 0; i < localArray.Length; i++)
        {
           result += //Do Work
        }
        return result;
    }
         
    public double LinqParallelSquareSum()
    {
        var localArray = rawArray;
        return localArray.AsParallel().Sum(/* Do Work */);
    }
    
    public double ParallelForSquareSum()
    {
        var localArray = rawArray;
        object lockObject = new object();
        double result = 0.0;
        Parallel.For(0, localArray.Length,() => 0.0,
            (i, loopState, partialResult) => { /*Do Work*/ },            
            (localPartialSum) => { lock (lockObject) { result += localPartialSum }});            
        return result;
    }
```

#### 1 million doubles - (result += x*x)

Method |    Median | Bytes Allocated/Op |
------------------------- |---------- |------------------- |
      Imperative |  1.1138 ms |          29,480.65 |            
    LinqParallel |  3.3174 ms |         117,802.56 | 
     ParallelFor |  1.9985 ms |          59,264.27 |

<br/>     
The easiest way to parallelize work like this in C# is with [PLINQ](https://msdn.microsoft.com/en-us/library/dd460688(v=vs.110).aspx).
Just type your collection name, then *.AsParallel()* and fire away with Linq queries. Unfortunately in this case it does no good, and
neither does the *Parallel.For* function.  The workload of just squaring doubles and adding them up isn't enough to get a net benefit
here. You would need to roll your own function using ThreadPools or perhaps Threads directly to see a speedup.

#### 1 million doubles - (result += Math.sin(x))

Method |     Median |  Bytes Allocated/Op |
---------------------- |----------- |------------------- |
      Imperative | 37.1130 ms |         840,522.92 | 
    LinqParallel |  9.8497 ms |         225,694.67 | 
     ParallelFor |  8.5615 ms |         166,386.40 |

<br/>
With the bigger workload there is now a large improvement by parallelizing. It takes a CPU about 2 orders of magnitude more cycles to perform a sin operation
that it does an add or multiply. Because of this, per-element overhead cost becomes a much smaller percentage of 
overall runtime, and we get the ~4x speedup we expect from 4 physical cores. It also reduces the relative cost of the simple Linq approach compared to the
more complex *Parallel.For* abstraction. Consider how big the workload is to help decide if the simple Linq approach is worth the cost.

### FSharp

F# has a number of easy to use 3rd party libraries for this purpose. All can be used from C# as well. 
A quick rundown of them here:

```ocaml

    (* Nessos Streams ParStream *)
    array
    |> ParStream.ofArray                    
    |> ParStream.fold (fun acc x -> acc + x*x)  (+) (fun () -> 0.0) 

    (* FSharp.Collections.ParallelSeq *)
    array
    |> PSeq.reduce (fun acc x -> acc+x*x)

    (* SIMDArray (uses AVX2 SIMD as well) *)
    array
    |> Array.SIMDParallel.fold (fun acc x -> acc + x*x)
                               (fun acc x -> acc + x*x)  
                               (+) (+) 0.0 

```

### 1 million doubles (result += x*x)

Method |     Time |  
---------------------- |----------- | 
 .NET / F# Parallel SIMDArray | 0.26ms | 
 .NET / F# Nessos Streams | 1.05ms | 
 .NET / F# ParallelSeq | 3.1ms |  

 <br/>

 [SIMDArray](https://github.com/jackmott/SIMDArray) is 'cheating' here as it also does SIMD operations, but I include it because I wrote it, so I do what I want.
 All of these out perform core library functions above.


### 1 million doubles (result += Math.Sin(x))

Method |     Time |  
---------------------- |----------- |
.NET / F# Nessos Streams | 6.7ms | 
.NET / F# ParallelSeq | 9.9ms |  

<br/>

The Sin operation can't be SIMDified here so SIMDArray is out. Nessos streams again proves to be better than the core library functions. 


### C++

Now the same experiment in C++. Most C++ compilers can auto parallelize loops, which you can control via compiler flags or inline hints in your code.  For instance
with Visual Studio's C++ compiler you can just put *#pragma loop(hint_parallel(8))* on top of a loop, and it will parallelize it if it can.  Unfortunately our
toy example is (intentionally) a tiny bit too complex for that.  Since we are summing up results, this creates a data dependency. Fortunately we can use [OpenMP](http://openmp.org/wp/),
which is available in Microsoft Visual C++, GCC, Clang, and other popular C++ compilers:

```c
    double result = 0;
    #pragma omp parallel for reduction(+ : result)
    for(int i = 0; i < COUNT; i++) 	
    {
       result += /*Do Work*/;
    }
```

This is equivalent to the Parallel.For loop used above in C#, where you identify that you will be aggregating data.  This is actually less typing
and easier to read too, even if the syntax is odd. How does it perform?

#### 1 million doubles - (result += x*x)  No SIMD

Method |     Median |  
---------------------- |----------- |
      ForLoop | 1.031 ms | 
    ParallelizedForLoop |  0.375 ms | 

<br/> 
We can see that OpenMP is managing a more efficient abstraction than .NET for this case, managing almost almost a 3x speedup where .NET was actually a bit slower.
Newer OpenMP implementations available on other compiles can also be directed to do SIMD vectorization in the loop for even more speed increase. That does not seem to be available
in MS Visual C++, and the usual automatic vectorization seems to not happen within the omp loop. Automatic vectorization can be done on the single thread
for loop but it was turned off for these C++ tests. *The C++ compilers does do older SSE instructions, as is the case with .NET and Java as well, but they only use
a single lane.  MSVC++ will use all lanes if you specify /fp:fast but only in the non OMP loop*

#### 1 million doubles - (result += sin(x))  No SIMD

Method |     Median |  
---------------------- |----------- |
      ForLoop | 10.625 ms | 
    ParallelizedForLoop |  2.44 ms | 

<br/>

This time a little more than a 3x speedup, and as you can see the results are overall faster than .NET as well.

### Java

Java's streams library which performed excellently in a [previous blog post](https://jackmott.github.io/programming/2016/07/22/making-obvious-fast.html) can be
used here again. You simply have to tell it you want a parallel stream:

```java

//Regular stream
sum = Arrays.stream(array).reduce(0,(acc,x) -> /*Do Work*/);

//ParallelStream
sum = Arrays.stream(array).parallel().reduce(0,(acc,x) -> /*Do Work*/);

```

<br/>

#### 1 million doubles (result += x*x)

Method |     Median |  
---------------------- |----------- |
      Stream | 1.03 ms | 
    Parallel Stream |  0.375 ms | 

<br/>

#### 1 million doubles (result += Math.sin(x))

Method |     Median |  
---------------------- |----------- |
      Stream | 34.5 ms | 
    Parallel Stream |  7.8 ms | 

<br/>

Java performs right on par with C++ in the first example, but falls behind when using Math.sin().  It appears that this is
not due to the parallel streams, but due to Java using a more accurate sin implementation, rather than calling the
x86 instruction directly.  This difference may not exist on other hardware. I do not like it when a langauge tells me I
can't touch the hardware if I want. A Math.NativeSin() would be nice. The streams library overall though has proven to be
excellent, matching C++ in both scalar and parallel varieties.

### Javascript

pfffftttt (yeah I know about Web Workers)

### Rust

Rust provides no easy loop parallelizing abstractions out of the box, you have to roll your own. OpenMP style
features [may be in the works for Rust](https://github.com/rust-lang/rfcs/issues/859) though, and 3rd party libraries are available.
So let's take a look at a nice one called [Rayon](https://github.com/nikomatsakis/rayon) which adds a "par_iter" providing similar
functions as the regular iter, but in parallel.  The code remains very simple:

```rust
    
    // The regular iter
    vector.iter().map(|&x| /* do work */).sum()

    // Parallel iter
    vector.par_iter().map(|&x| /* do work */).sum()

```

<br/>

#### 1 million doubles (result += x*x)

Method |     Median |  
---------------------- |----------- |
      iter | 1.05 ms | 
    par_iter |  .375 ms | 

<br/>

#### 1 million doubles (result += Math.sin(x))

Method |     Median |  
---------------------- |----------- |
      iter | 9.65 ms | 
    par_iter |  2.44 ms | 

<br/>

These are excellent results, tied with C++, and requiring only a single line of code to express.

## Summary

The loop abstractions examined here are just one type of parallel or concurrent programming abstraction available. There is
a whole universe out there, Actor Models, Async/Await, Tasks, Thread Pools, and so on.  Be sure to understand what you are using,
and measure whether it will really be useful, or whether you should focus on fast single threaded algorithms or look for third party
tools with better performance.


## Aggregated Testing Results

#### 1 million doubles ( result += x*x)  No SIMD ( Except SIMDArray)

Method |     Time |  Lines Of Code
---------------------- |----------- | | ---|
  .NET / F# SIMDArray | 0.26ms | 1 |
  Java Parallel Streams |  0.375 ms | 1 |
  Rust Rayon | 0.375 ms | 1 |  
  C++ OpenMP | 0.375 ms | ~5 |     
 .NET / F# Nessos Streams | 1.05ms | ~2 |
 .NET Parallel.For | 1.9ms | ~6 | 
 .NET / F# ParallelSeq | 3.1ms |  1 |
 .NET Parallel Linq (Sum) | 3.3ms | 1 |
 .NET Parallel Linq (Aggregate) | 8ms | 1 |
 

<br/>

#### 1 million doubles ( result += sin(x))  No SIMD

Method |     Time |  Lines Of Code
---------------------- |----------- | | ---|
  Rust Rayon | 2.44 ms | 1 | 
  C++ OpenMP | 2.44 ms | ~4 |  
 .NET / F# Nessos Streams | 6.7ms | ~2 |
  Java Parallel Streams |  7.8 | 1 |
 .NET Parallel.For |  8.5615 ms |  ~6 |
 .NET Parallel Linq (Sum) |  9.8497 ms | 1 |     
 .NET / F# ParallelSeq | 9.9ms |  1 |
 .NET Parallel Linq (Aggregate) | 45.6ms | 1 |
 
 
<br/>

## Benchmark Details 

All benchmarks run with what I believe to be the latest and greatest compilers available for Windows for each language (Debateable for C++). 
JIT warmup time is accounted for when
applicable. If you identify cases where code or compiler/environment choices are sub optimal, email me please. 

#### Environment
```ini
Host Process Environment Information:
BenchmarkDotNet=v0.9.8.0
OS=Microsoft Windows NT 6.2.9200.0
Processor=Intel(R) Core(TM) i7-4712HQ CPU 2.30GHz, ProcessorCount=8
Frequency=2240907 ticks, Resolution=446.2479 ns, Timer=TSC
```

#### F# / C# Runtime Details
```ini
CLR=MS.NET 4.0.30319.42000, Arch=64-bit RELEASE [RyuJIT]
GC=Concurrent Workstation
JitModules=clrjit-v4.6.1590.0
Type=SIMDBenchmark  Mode=Throughput  Platform=X64  
Jit=RyuJit  GarbageCollection=Concurrent Workstation  

```

### C++ Details
Visual Studio 2015 Update 3, Optimizations set for maximum speed, SIMD off

### Java Details
Oracle Java 64bit version 8 update 102

### Rust Details
rustc 1.13.0-nightly
build with `cargo rustc --release -- -C lto -C target-cpu=native`