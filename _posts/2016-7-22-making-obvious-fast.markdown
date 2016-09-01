---
layout: post
title:  "Making the obvious code fast"
subtitle: "A tour of Go, Java, C#, F#, Rust, Javascript and C"
date:   2016-07-22 14:17:27 -0500
categories: programming
---
[Jonathan Blow](http://number-none.com/blow/) of "The Witness" fame likes to talk about just typing the obvious code first.  Usually it will turn
out to be fast enough.  If it doesn't, you can go back and optimize it later.  His thoughts come in the context of working on games in C/C++. 
I think these languages, with modern incarnations of their compilers, are compatible with this philosophy. Not only are the compilers very 
mature but they are low level enough that you are forced to do things by hand, and think about what the machine is doing most of the time, especially if
you stick to C or a 'mostly C' subset of C++. However in most higher level languages, there tend to be performance traps where the obvious, or idiomatic solution is 
particularly bad.  

What counts as obvious or idiomatic, is of course often a matter of opinion.  The language itself may encourage certain choices
by making them easier to type, or highlighting them in documentation and teaching materials.  The community that grows up around a language
may just come to prefer certain constructs and encourage others to use them. It is very common to see programmers encouraged to use
high level constructs over lower level ones, in the interest of readability and simplicity.  This is a worthy ideal, but often people
aren't aware of what the cost really is.  Some of these constructs have a much higher cost than people realize.

In this article I will explore a number of languages, with a toy map and reduce example. Within each language, I will explore 
a number of approaches, ranging from high level to hand coded imperative loops and SIMD operations.  Some of the performance pitfalls I will show may be specific
to this toy example. With a different toy example, the languages that excel and those that do poorly could be totally different.  This is meant
merely to explore, and get people thinking about the performance cost of abstractions. For each case I will show code examples so you
can consider the differences in complexity.

## The Task

We wish to take an array of 32 million 64bit floating point values, and compute the sum of their squares. This will let us explore some
fundamental abilities of various languages. Their ability to iterate over arrays efficiently, whether they can vectorize basic loops, and 
whether higher order functions like map and reduce compile to efficient code. When applicable, I will show runtimes of both map and reduce, 
so we get insight into whether the language can stream higher order functions together, and also the runtime with a single
reduce or fold operation. 

## The Results

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
ANSI C is a bare bones language, no higher order functions or loop abstractions exist to even think about, so this imperative loop is what most
programmers wil turn to to complete this task. If I thought that this would be a performance critical piece of code, I might use SIMD intrinsics, 
which requires this nasty mess:

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

However, notice that the runtime is the same for the obvious and SIMD versions!  It turns out that the obvious code was automatically turned into SIMD
enhanced machine instructions. A process called "Auto vectorization".  Visual C++ is not known for being the most clever of C++ compilers but it still 
gets this right:

``` asm
double sum = 0.0;    
	for (int i = 0; i < COUNT; i++) {
00007FF7085C1120  vmovupd     ymm0,ymmword ptr [rcx]  
00007FF7085C1124  lea         rcx,[rcx+40h]  
		double v = values[i] * values[i];  //square em
00007FF7085C1128  vmulpd      ymm2,ymm0,ymm0  
00007FF7085C112C  vmovupd     ymm0,ymmword ptr [rcx-20h]  
00007FF7085C1131  vaddpd      ymm4,ymm2,ymm4  
00007FF7085C1135  vmulpd      ymm2,ymm0,ymm0  
00007FF7085C1139  vaddpd      ymm3,ymm2,ymm5  
00007FF7085C113D  vmovupd     ymm5,ymm3  
00007FF7085C1141  sub         rdx,1  
00007FF7085C1145  jne         imperative+80h (07FF7085C1120h)  
		sum += v;
	}

```
Visual C++ will actually use older SSE SIMD instructions by default, which can do a few operations on two doubles or four floats at a time.
To get AVX2 instructions used here, which can operate on 4 doubles at a time, you have to specify to the compiler that you want 'fast floating point'
and specify that you want to target AVX2 instructions as well. Results will be different when vectorized, though they will actually be more 
accurate, not less. (in this case, maybe all?)

### C# Linq Select Sum - 260 milliseconds

``` c#
    var sum = values.Sum(x => x * x);
```

### C# Linq Aggregate - 280 milliseconds

``` c#
    var sum = values.Aggregate(0.0,(acc, x) => acc + x * x);
```

### C# for loop - 34 milliseconds

``` c#
    double sum = 0.0;
    foreach (var v in values)
    {
        checked {
            double square = v * v;
            sum += square;
        }
    }
```

Stepping up a level to C#, we have a couple of idiomatic solutions.  Many C# programmers today might use Linq which as you can see is much slower. It also creates a 
lot of garbage, putting more pressure on the garbage collector. Oddly, the Aggregate function, which is equivalent to fold or reduce in most other languages, is slower
despite being a single step instead of two.  The foreach loop in the second example is also commonly used.  While this pattern has big 
performance pitfalls when used on collections like List&lt;T&gt;, with arrays it compiles to efficient code. This is nice as it saves you some typing without 
runtime penalty. The  runtime here is still twice as slow as the C code, but that is entirely due to not being automatically vectorized with AVX2 instructions.  
With the .NET JIT, it is not considered a worthwhile tradeoff to do this particular optimization, though it does use SSE SIMD here.

With C# you also have to take some care with array access in loops, 
or [bounds checking overhead can be introduced](http://www.codeproject.com/Articles/844781/Digging-Into-NET-Loop-Performance-Bounds-checking). In this case the 
JIT gets it right, and there is no bounds checking overhead. Checked arithmetic is used in the loop body for parity with Linq, which also uses checked
arithmetic on sum, though this has almost immeasurable impact on runtime.

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

While the .NET JIT won't do AVX2 SIMD automatically, we can explicitly use some SIMD instructions, and achieve performance nearly identical to C.  An advantage here for C# is that the SIMD code is a
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
map and sum operations together, and avoid iterating over the array twice.  The [Nessos Streams](https://github.com/nessos/Streams) library provides this, 
with a nice performance improvement as a result.

### F# Fold - 75 milliseconds

``` ocaml
    let sum = 
        values
        |> Array.fold (fun acc x -> acc + x*x) 0.0
```

When we use a single fold operation, we no longer iterate over the collection twice and allocate extra memory, and runtime improves even more.  Since
there is no overhead for streaming together multiple higher order functions as there is in the Streams library, it does slightly better.

### F# Imperative - 38 milliseconds

``` ocaml
    let mutable sum = 0.0
    for i = 0 to values.Length-1 do
            let x = values.[i]
            sum <- sum + x*x            
```

One of the nice things about F#, is that while it is a functional leaning language, very few barriers are put in your way if you want
to go imperative for the sake of speed.  Write a normal for loop, and you get the same performance as SSE vectorized C.

### F# SIMD - 18ms

``` ocaml
    let sum =
        values
        |> Array.SIMD.fold (fun acc v -> acc +v*v) (+) 0.0    
```

Now to get serious. First we use fold, so that we can combine the summing and squaring into a single pass. Then we use the
[SIMDArray extensions](https://github.com/jackmott/SIMDArray) that I have been working on which let you take full advantage
of SIMD with more idiomatic F#.  Performance here is great, nearly as fast as C, but it took a lot of work to get here.
At the moment there is no way to combine the lazy stream optimization with the SIMD ones. If you want to filter->map->reduce you will
still be doing a lot of extra work.  This should be possible in principle though. Please submit a PR!


### Rust - 34ms

``` rust
    let sum = values.iter().
                map(|x| x*x).        
                sum()  
```

Rust achieves impressive numbers with the most obvious approach. This is super cool. I feel that this behavior should be the goal for any language 
offering these kinds of higher order functions as part of the language or core library. Using a traditional for loop or a 'for x in y' style loop
is also just as fast. It is also possible to use rust intrinsics to get the same speed as the AVX2 vectorized C code here, but to use those you have to 
write out the loop explicitly:

### Rust SIMD - 18ms

``` rust
    let mut sum = 0.0;
    unsafe {
        for v in values {
            let x : f64 = std::intrinsics::fmul_fast(*v,*v);
            sum = std::intrinsics::fadd_fast(sum,x); 
        }
    }
    sum
```

It would be nice if the rustc compiler had an option to just apply this globally, so you could use the higher order functions. Also,
these features are marked as unstable, and likely to remain unstable forever.  This might make it problematic to use this feature for
any important production project.  It would also be nice if the unsafe block was not required. 
Hopefully the Rust maintainers have a plan to make this better.

### Javascript map reduce (node.js) 10,000ms

``` javascript
var sum = values.map(x => x*x).
                 reduce( (total,num,index,array) => total+num,0.0);
```

### Javascript reduce (node.js) 800 and then 300 milliseconds

``` javascript
var sum = values.reduce( (total,num,index,array) => total+num*num,0.0)
```

It is common to see these higher order javascript functions suggested as the most elegant way to do this, but it is incredibly slow. Simplifying the 
combined map and reduce improves runtime by an order of magnitude to 800ms, though after 3 or 4 iterations the JIT does some magic and runtime
drops to 300ms thereafter.  This represents the first time I have seen any substantive JIT optimization happen during runtime in the wild!  

### Javascript foreach (node.js) 800 and then 300 milliseconds

``` javascript
    var sum = 0.0;
    array.forEach( (element,index,array) => sum += element*element  )
```
Slightly less elegant but also a popular idiom in javascript, this is faster than map and reduce, but is still amazingly slow. Again, after
3 or 4 iterations the JIT does some magic and it speeds up from around 800 to 300 milliseconds.

### Javascript imperative (node.js) 37 milliseconds

``` javascript

    var sum = 0.0;
    for (var i = 0; i < values.length;i++){
        var x = values[i];
        sum += x*x;
    }

```

Finally, when we get down to a basic imperative for loop, javascript performs comparably to SEE vectorized C.

### Java Streams Map Sum 138 milliseconds

``` java
    double sum = Arrays.stream(values).
                        map(x -> x*x).
                        sum();
```

### Java Streams Reduce 34 milliseconds

``` java
    double sum = Arrays.stream(values).
                        reduce(0,(acc,x) -> acc+x*x);
```

Java 8 includes a very nice library called [stream](https://docs.oracle.com/javase/8/docs/api/java/util/stream/package-summary.html) which provides higher
order functions over collections in a lazy evaluated manner, similar to the F# Nessos streams library and Rust. Given that this is a lazy evaluated system, it is
odd that there is such a performance difference between map then sum and a single reduction.  The reduce function is compiling down to the equivalent of SSE vectorized C,
but the map then sum is not even close. It turns out that the `sum()` method on `DoubleStream`:

>may be implemented using compensated summation or other technique to reduce the error bound in the numerical sum compared to a simple summation of double values.

A nice feature, but not clearly communicated by the method name!  If we tweak the java code to do normal summation the runtime remains as fast as SSE vectorized C, a nice 
accomplishment:

### Java Streams Map Reduce 34 milliseconds 

``` java
    double sum = Arrays.stream(values).
                        map(x -> x*x).
                        reduce(0,(acc,x) -> acc+x);
```

There does not appear to be a way to get AVX2 SIMD out of Java, either explicitly or via automatic vectorization by the Hotspot JVM. There are 3rd
party libraries available that do it by calling C++ code.

### Go for Range 37 milliseconds

``` go
    sum := 0.0
    for _,v := range values[:] {
        sum = sum +  v*v
    }
```

### Go for loop 37 milliseconds
``` go
    sum := 0.0
    for i := 0; i < len(values); i++ {
        x := values[i]
        sum = sum +  x*x
    }
```

Go has good performance with the both the usual imperative loop and their 'range' idiom which is like a 'foreach' in other languages.   
Auto vectorization using newer SIMD instructions appears to be completely not on the Go radar.  There are no map/reduce/fold higher order functions 
in the standard library, so we can't compare them.  Go does a good thing here by not providing a slow path at all.

### Conclusion

I have shown some performance pitfalls in various languages here.  One should not read too much into this as an argument for general performance of these languages.
Every language has some pitfalls where the preferred or easiest approaches to solving a problem can lead to performance pitfalls. 
In Java, for instance, everything is objects. Objects all allocate on the heap (unless the JIT does some work at runtime to determine it doesn't 
need to go on the heap, but that isn't a freebie).  Since Java is also a garbage collected language, this can lead to performance pitfalls when
you type the obvious code.  With experience, you can learn about these pitfalls and do work to avoid them, just like you can avoid pitfalls of Linq in C#, by not using it, 
or the pitfalls of F# by using Stream or SIMD libraries instead of the core ones.  But even then, you have to take extra care, and type extra code, or take on more 
dependencies to do that.  This is partially purpose defeating, since high level languages are supposed to let you type less, and get things working faster.

What I would like to see is more of an attitude change among high level language designers and their communities.  None of the issues above need to exist.
Java could (and will, soon) provide value types (as C# does) to make it less painful to avoid GC pressure if you use lots of small, short lived constructs. Go
could provide more SIMD support, either via a SIMD library or better auto vectorization. F# could provide efficient Streams as part of the core library like Java does. 
 .NET could auto vectorize 
code with newer SIMD instructions in the JIT and/or provide more complete coverage of SIMD instructions in the Vector library.  We, the community, can help by providing libraries 
and [submitting PRs](https://jackmott.github.io/programming/2016/08/13/adventures-in-fsharp.html) to make the obvious code faster.  Time and energy will be saved, 
batteries will last longer, users will be happier.


### Benchmark Details <a name="benchmark"></a>

All benchmarks run with what I believe to be the latest and greatest compilers available for Windows for each language. JIT warmup time is accounted for when
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

### C Details
Visual Studio 2015 Update 3, fast floating point, 64 bit, AVX2 instructions enabled, all speed optimizations on

### Rust Details
v1.13 Nightly, --release -opt-level=3

### Javascript/Node Details
v6.4.0 64bit
NODE_ENV=production

### Java Details
Oracle Java 64bit version 8 update 102

### Go Details
Go 1.7