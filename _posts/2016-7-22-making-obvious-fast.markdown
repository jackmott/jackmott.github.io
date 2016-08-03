---
layout: post
title:  "Making the obvious code fast"
date:   2016-07-22 14:17:27 -0500
categories: programming
---
[Jonathan Blow](http://number-none.com/blow/) of "The Witness" fame likes to talk about just typing the obvious code first.  Usually it will turn
out to be fast enough.  If it doesn't, you can go back and optimize it later.  His thoughts come in the context of working on games in C/C++. 
I think these languages, with modern incarnations of their compilers, are especially compatible with this philosophy.  I think that in most higher level
languages today, there tend to be performance traps where the obvious, idiomatic solution is particularly bad.
Let's look at an example, comparing F# to C# and to C.

Pretend we wish to take an array of 32 million numbers, and compute the sum of their squares. The most obvious code for this in C is as follows:

* [Benchmark Details](#benchmark)

### C - 17 milliseconds

``` c
    double sum = 0.0;    
    for (int i = 0; i < COUNT; i++) 
    {
        double v = values[i] * values[i]; 
        sum += v;
    }
```

However we can get much trickier, if we thought that this would be a performance critical piece of code, we might use SIMD intrinsics, which requires
this nasty mess:

### C - SIMD Explicit - 17 milliseconds

``` c
    __m256d vsum = _mm256_setzero_pd();
    for(int i = 0; i < COUNT/4; i=i+1) {
        __m256d v = values[i];
        vsum = _mm256_add_pd(vsum,_mm256_mul_pd(v,v));
    }
    double *tsum = &vsum;
    double sum = tsum[0]+tsum[1]+tsum[2]+tsum[3];
```

However, notice that the runtime is the same, for the obvious and SIMD versions!  It turns out that the obvious code was automatically turned into SIMD
enhanced machine instructions. A process called "Auto vectorization".  Visual C++ is not known for being the most clever of C++ compilers but it still 
gets this right.

### C# - 34 milliseconds

``` c#
    double sum = 0.0;
    foreach (var v in values)
    {
        double square = v * v;
        sum += square;
    }
```

Stepping up a level to C#, we have a fairly idiomatic solution.  Some C# programmers today might use Linq, or an Enumerator by default, which would be 
slower and put pressure on the garbage collector. This foreach loop isn't too unusual however, and saves us some typing over a for loop. The  runtime here 
is twice as slow as the C code, and that is entirely due to not being automatically vectorized.  With the .NET JIT, it is not considered a worthwhile 
tradeoff to do this particular optimization. Additionally, with C# you have to take some care with array access in loops, or [bounds checking overhead can be introduced](http://www.codeproject.com/Articles/844781/Digging-Into-NET-Loop-Performance-Bounds-checking). 
In this case the JIT gets it right, and there is no bounds checking overhead.

### C# SIMD Explicit - 17 milliseconds
``` c#
    Vector<double> vsum = new Vector<double>(0.0);
    for (int i = 0; i < COUNT; i += Vector<double>.Count)
    {
        var value = new Vector<double>(values, i);
        vsum = vsum + (value * value);
    }
    double sum = 0;
    for(int i = 0; i < Vector<double>.Count;i++)
    {
        sum += vsum[i];
    }
```

We can, however, explicitly use SIMD instructions, and achieve performance nearly identical to C.  An advantage here for C# is that the SIMD code is a
bit less nasty than using intrinsics, and that particular instructions whether they be AVX2, SSE2, NEON, or whatever the hardware supports, can be decided
upon at runtime.  Whereas the C code above would require separate compilation for each architecture. A disadvantage for C# is that not all SIMD instructions
are exposed by the Vector library, so something like [SIMD enhanced noise functions](https://github.com/Auburns/FastNoiseSIMD) can't be done with nearly the
same performance. As well, the machine code produced by the Vector library is not always as efficient when you step out of toy examples.

### F# - 127 milliseconds

``` ocaml
    let sum =
        values
        |> Array.map squares
        |> Array.sum
```

The obvious F# code is beautiful, I like typing this, and I like working with it.  But performance is terrible. Just as with C# you get no auto vectorization,
as they use the same JIT.  Additionally the array is iterated over twice, once to map them to squares, and once to sum them. Finally, since 
immutability is the default, each operation returns a new array, incurring allocation costs and GC pressure. So the total performance impact
on an application is likely to be worse than this micro benchmark would suggest.

### F# Streams - 98 milliseconds

``` ocaml
    let sum = 
        values
        |> Stream.map square
        |> Stream.sum
```

F# is a functional first language, rather than a pure functional language like Haskell. If you do happen to use pure functions, you can stream your
map and sum operations together, and avoid iterating over the array twice.  The [Nessos Streams](https://github.com/nessos/Streams) provides this, 
with a nice performance improvement as a result.

### F# Fast - 18ms

``` ocaml
    let sum =
        values
        |> Array.SIMD.fold (fun acc v -> acc +v*v) (+) 0.0    
```

Now to get serious. First we use fold, so that we can combine the summing and squaring into a single pass. Then we use the
[SIMDArray extensions](https://github.com/jackmott/SIMDArray) that I have been working on which let you take advantage
of SIMD instructions with more obvious F# style.  Performance here is great, nearly as fast as C, but it took a lot of work to get here.

I'm not picking on F#, every high level language seems to have similar issues where the obvious, encouraged, pleasant coding style leads to performance pitfalls. 
In Java, for instance, everything is objects, and objects all allocate on the heap (unless the JIT does some work at runtime to determine it doesn't 
need to go on the heap, but that isn't a freebie).  Since Java is also a garbage collected language, this can lead to performance pitfalls when
you type the obvious code.  With experience, you can learn about these pitfalls and do work to avoid them, just like you can avoid the F# 
pitfalls above.  But even then, you have to take extra care, and type extra code to do that.  This is partially purpose defeating, since high level 
languages are supposed to let you type less, and get things working faster.

What I would like to see is more of an attitude change among high level language designers and their communities.  None of the issues above need to exist.
Java could (and will, soon) provide value types (as C# does) to make it less painful to avoid GC pressure if you use lots of small, short lived constructs.
F# could provide Streams as part of the core library (as Java does).  .NET could auto vectorize code in the JIT (as Java does, to a degree) and/or provide more 
complete coverage of SIMD instructions in the Vector library.  We, the community, can help by providing libraries and submitting PRs to make the obvious 
code faster.  Time and energy will be saved, batteries will last longer, users will be happier.

### Benchmark Details For The Pedants<a name="benchmark"></a>

All benchmarks run with the latest Microsoft compiler for each language, multiple trials, 64 bit, release mode, with all settings set for maximum speed, with
JIT warmup time accounted for.

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
