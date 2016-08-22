---
layout: post
title:  "When Big O Fools Ya"
date:   2016-08-20 14:17:27 -0500
categories: programming
---

Big O notation is a great tool.  It allows one to quickly make smart choices among various data structures and algorithms.  But sometimes a casual
Big O analysis can fool us if we don't think carefully about the impact of constant factors. One such examples comes up very often
when programming on modern CPUs, and that is when choosing between an Array, and a List, or Tree type structure.


### Memory, Slow Slow Memory

In the early 1980s, the time it took to get data from RAM, and the time it took to do computation on the data were roughly in parity.  You could 
use algorithms that hop randomly over the heap, grabbing data and working with it.  Since that time, CPUs have gotten faster at a much higher rate
than RAM has.  Today, a CPU can compute on the order of 100 to 1000 times faster than it can get data from RAM. This means when the cpu needs data from 
RAM it has to stall for hundreds of cycles, doing nothing.  Obviously this would be a useless situation, so modern CPUs have various levels of cache built in.
Any time you request one piece of data from RAM, you also get chunks of contiguous memory pulled into the caches on the CPU. The result is that when you iterate
over contiguous memory, you can access it about as fast as the CPU can operate, because you will be streaming chunks of data into the L1 cache.  If you iterate
over memory in random locations, you will often miss the CPU caches, and performance can suffer greatly.  If you want to learn more about this, 
[Mike Acton's CppCon talk](https://www.youtube.com/watch?v=rX0ItVEVjHc) is a great starting point and great fun too.

The consequence of this is that arrays have become the go to data structure if performance is important, sometimes even when Big O analysis suggests it would be slower. Where you wanted a Tree
before you may want a sorted array and a binary search algorithm. Where you wanted a Queue before you may want a growable array, and so on.

### Linked List vs Array List

Once you are familiar with how important contiguous memory access is, it should be no surprise that if you want to iterate over a collection quickly, that an array will be faster than
a Linked List.  Environments with clever allocators and garage collectors may be able to keep Linked List nodes somewhat contiguous, some of the time, but they can't guarantee it.  Using a raw
array usually involves quite a bit more complex code, especially if you want to be able to insert or add items, as you will have to deal with growing the array, shuffling elements around, and so on.
Most language's have core libraries which include some sort of growable array data structure to help with this.  In C++ you have [vector](http://www.cplusplus.com/reference/vector/vector/), in C# you have
[List&lt;T&gt; (aliased as ResizeArray in F#)](https://msdn.microsoft.com/en-us/library/6sh2ey19(v=vs.110).aspx), and in Java there is [ArrayList](https://docs.oracle.com/javase/8/docs/api/java/util/ArrayList.html). Usually 
these data structures expose the same, or similar interface as the Linked List collection.  **I will refer to such data structures as Array Lists from here on, but keep in mind all the C# examples are using the List&lt;T&gt; class, not the older ArrayList class.**

So what if you need a data structure that you can insert items into, and iterate over quickly? Let us assume for this example, that we have a use case where we will insert into the front
of a collection about 5 times more often that we iterate over it. Let us also assume that the Linked List and Array List in our environment have interfaces which are equally pleasant to work with for this task.
All that remains then to make a choice is to determine which one performs better.  In the interest of optimizing our own valuable time, one might turn to Big O analysis.  Referring to the handy 
[Big-O Cheat Sheet](http://bigocheatsheet.com/), the relevant time complexities for these two data structures are:

|             | Iterate | Insert |
|-------------|---------|--------|
| Array List  | O(n)    | O(n)   |
| Linked List | O(n)    |  O(1)  |

<br/>
Array Lists are problematic for insertion, at a minimum it has to copy every single element beyond the insertion point in the array to move them over by 1 to make space for the inserted element, making it O(n). Sometimes it will
also have to reallocate a new, bigger array to make room for the insertion.  This doesn't change the Big O time complexity, but does take time, and waste memory.  So it seems for our use case, where
insert happens 5 times more often than iterating, that the best choice is clear.  As long as n is large enough, Linked List should perform better overall.

### Empiricism

But, to know things for sure, we always have to count. So let us do an experiment in C#, using [BenchMarkDotNet](https://github.com/PerfDotNet/BenchmarkDotNet). C# provides generic collections
LinkedList<T> which is a classic Linked List, and List<T> which is an Array List. Their interfaces are similar, and both allow us to implement our use case with ease.  We will assume a worst case scenario for
Array List, by always inserting at the front, necessitating that the entire array be copied on each insertion. The testing environment specs are:

```ini
Host Process Environment Information:
BenchmarkDotNet.Core=v0.9.9.0
OS=Microsoft Windows NT 6.2.9200.0
Processor=Intel(R) Core(TM) i7-4712HQ CPU 2.30GHz, ProcessorCount=8
Frequency=2240910 ticks, Resolution=446.2473 ns, Timer=TSC
CLR=MS.NET 4.0.30319.42000, Arch=64-bit RELEASE [RyuJIT]
GC=Concurrent Workstation
JitModules=clrjit-v4.6.1590.0

Type=Bench  Mode=Throughput  
```

### Test Cases:

``` c# 
    [Benchmark(Baseline=true)]
    public int ArrayTest()
    {
        var local = arrayList;
        int sum = 0;
        for (int i = 0; i < this.inserts; i++)
        {
            local.Insert(0, 1); //Insert the number 1 at the front
        }

        // For loops iterate over List<T> much faster than foreach
        for (int i = 0; i < local.Count; i++)
        {
            sum += local[i];  //do some work here so the JIT doesn't elide the loop entirely
        }
        return sum;
    }

    [Benchmark]
    public int ListTest()
    {
        var local = linkedList;
        int sum = 0;
        for (int i = 0; i < this.inserts; i++)
        {
            local.AddFirst(1); //Insert the number 1 at the front
        }

        // Again, iterating the fastest possible way over this collection
        var node = local.First;
        for (int i = 0; i < local.Count; i++)
        {
            sum += node.Value;
            node = node.Next;
        }

        return sum;
    }
```

### Results:


 Method |   length | inserts |         Median |        StdDev | Scaled | Scaled-SD |    Gen 0 |    Gen 1 | Gen 2 | Bytes Allocated/Op |
---------- |--------- |-------- |--------------- |-------------- |------- |---------- |--------- |--------- |------ |------------------- |
 ArrayTest |      100 |       5 |     38.9983 us |     0.9040 us |   1.00 |      0.00 |        - |        - |     - |              25.10 |
  ListTest |      100 |       5 |     51.7538 us |     1.3161 us |   1.30 |      0.04 |        - |        - |     - |              95.66 |
 
<br/>

The Array List wins by a nose. But this is a small list, Big O only tells us about performance as `n` grows large, so we should see this trend eventually reverse as `n` grows larger. 
Let's try it:

 Method |   length | inserts |         Median |        StdDev | Scaled | 
---------- |--------- |-------- |--------------- |-------------- |------- |
 ArrayTest |      100 |       5 |     38.9983 us |     0.9040 us |   1.00 |  
  ListTest |      100 |       5 |     51.7538 us |     1.3161 us |   1.30 |  
 ArrayTest |     1000 |       5 |     42.1585 us |     1.0770 us |   1.00 |  
  ListTest |     1000 |       5 |     49.5561 us |     1.6787 us |   1.17 |  
 ArrayTest |   100000 |       5 |    208.9662 us |     3.3698 us |   1.00 |  
  ListTest |   100000 |       5 |    312.2153 us |    10.3753 us |   1.48 |  
 ArrayTest |  1000000 |       5 |  2,179.2469 us |    36.8483 us |   1.00 |  
  ListTest |  1000000 |       5 |  4,913.3430 us |   133.5379 us |   2.27 |  
 ArrayTest | 10000000 |       5 | 36,103.8456 us | 1,251.5668 us |   1.00 |  
  ListTest | 10000000 |       5 | 49,395.0839 us | 1,355.5119 us |   1.37 |  

<br/>
Here we get the result that will be counterintuitive to many. No matter how large `n` gets, the Array List still performs better overall. In order for performance to get worse, the *ratio*
of inserts to iterations has to change, not just the length of the collection. Note that isn't an actual failure of Big O analysis, it is merely a common human failure in our application of it. 

Where the break even point occurs will depend on many factors, though a good rule of thumb suggested by 
[Chandler Carruth](https://www.youtube.com/watch?v=fHNmRkzxHWs) at Google is that Array Lists will outperform Linked Lists until you are inserting about an order of magnitude more often than you are iterating. 
This rule of thumb works well in this particular case, as 10:1 is where we see Array List start to lose:

  Method |   length | inserts |             Median |            StdDev | Scaled | 
--------- |--------- |-------- |------------------- |------------------ |------- |
 ArrayTest |   100000 |      10 |    328,147.7954 ns |     5,721.7161 ns |   1.00 |  
 ListTest|   100000 |      10 |    324,349.0560 ns |     5,030.1883 ns |   0.86 |  

<br/>

### Devils in the Details

The reason Array List wins here is because the integers being iterated over are lined up contiguously in memory. Each time an integer is requested from memory an entire cache line of integers is pulled into the L1
cache, so the next 64 bytes
of data are ready to go. With the Linked List, each call to `node.Next` makes a pointer hop to the next node, and there is no guarantee that nodes will be contiguous in memory.  Therefore we will 
miss the cache sometimes.  But we aren't always iterating over value types like this,  especially in OOP oriented managed languages we often iterate over reference types.  In that case, even with an Array List, while the 
pointers themselves are contiguous in memory, the objects they point to are not. The situation is still better than with a Linked List, where you will be making two pointer hops per iteration instead of one, 
but how does this affect the relative performance?

It narrows it quite a bit, depending on the size of the objects, and the details of your hardware and software environment. Refactoring the example above to use Lists of small objects (12 bytes), the break even
point drops to about 4 inserts per iteration:

  Method | length | inserts |        Median |     StdDev | Scaled | 
---------------- |------- |-------- |-------------- |----------- |------- |
 ArrayTestObject | 100000 |       0 |   674.1864 us | 16.5796 us |   1.00 |  
  ListTestObject | 100000 |       0 | 1,140.9044 us | 12.5662 us |   1.67 |     
 ArrayTestObject | 100000 |       2 |   959.0482 us | 14.5091 us |   1.00 |  
  ListTestObject | 100000 |       2 | 1,121.5423 us | 23.0159 us |   1.17 |   
 ArrayTestObject | 100000 |       4 | 1,230.6550 us | 20.1380 us |   1.00 |  
  ListTestObject | 100000 |       4 | 1,142.6658 us | 23.2076 us |   0.92 |  
 
 <br/>

Managed C# code suffers a bit in this case because iterating over this Array List incurs some unnecessary array bounds checking. C++ vector would likely fare better.  If you were really aggressive about this you
could probably write a faster Array List class using unsafe C# code to avoid the array bounds checks.  Also, the relative differences here will depend greatly on how your allocator and garbage collector manage the heap, 
how big your objects are, and other factors.  Larger objects tended to cause the relative performance of the Array List to improve in my environment. In the context of a complete application the relative performance
of Array List might improve as well as the heap gets more fragmented, but you will have to test to know for sure.

As an aside, if your objects are sufficiently small (16 to 32 bytes or less, depending on various factors) you should consider  making them value types (`struct` in .NET) instead of objects. 
Not only will you benefit greatly from contiguous memory access, but you will potentially reduce garbage collection overhead as well, depending on your usage of them:

  Method | length | inserts |        Median |     StdDev |
----------------- |------- |-------- |-------------- |----------- |
  ArrayTestObject | 100000 |      10 | 2,094.8273 us | 15.6230 us |
 ListTestObject | 100000 |      10 | 1,154.3014 us | 16.2864 us |
  ArrayTestStruct | 100000 |      10 |   792.0004 us |  3.5715 us |
 ListTestStruct | 100000 |      10 | 1,206.0713 us | 18.1695 us |

<br/>

Java may handle this better since it does some automatic cleverness with small objects, or you may have to just use separate arrays of primitive types.  Though onerous to type, 
[this can sometimes be faster](https://software.intel.com/en-us/articles/memory-layout-transformations) than an array of structs, depending on your data access patterns.  Consider it when performance matters.

### Make Sure the Abstraction is Worth It
   
It is common for people to object to these sorts of considerations on the basis of code clarity, correctness, and maintainability.  Of course each problem domain has it's own
priorities, but I feel strongly that when the clarity benefit of the abstraction is small, and the performance impact is large, that we should choose better performance as a rule.
By taking time to understand your environment, you will be aware of cases where a faster but equally clear option exists, as is often the case with Array Lists vs Lists.


As some food for thought, here are 7 different ways to add up a list of numbers in C#, with their run times and memory costs. Notice how *much* better performing the fastest option is.
Notice how bad the most popular method (Linq) is. Notice that the `foreach` abstraction works out well with raw Arrays, but not with Array List
or Linked List.  Whatever your language and environment of choice is, understand these details so you can make smart default choices. 

  Method | length |        Median |      StdDev | Scaled | Scaled-SD | Gen 0 | Gen 1 | Gen 2 | Bytes Allocated/Op |
------------------ |------- |-------------- |------------ |------- |---------- |------ |------ |------ |------------------- |
    LinkedListLinq | 100000 | 1,020.2228 us | 175.0166 us |   1.00 |      0.00 | 58.00 |  1.00 |     - |          21,008.89 |
      RawArrayLinq | 100000 |   648.8836 us |  14.4778 us |   0.63 |      0.06 | 29.21 |  0.57 |     - |          10,628.37 |
 LinkedListForEach | 100000 |   570.7069 us |  30.0890 us |   0.56 |      0.06 | 22.60 |  0.37 |     - |           8,168.27 |
     LinkedListFor | 100000 |   372.5064 us |  16.9530 us |   0.36 |      0.04 | 14.78 |  0.32 |     - |           5,395.63 |
     ArrayListForEach | 100000 |   270.6204 us |  27.6803 us |   0.27 |      0.04 |  8.92 |  0.05 |     - |           3,162.80 |
      ArrayListFor | 100000 |    99.9233 us |  11.7005 us |   0.10 |      0.02 |  7.54 |  0.17 |     - |           2,754.28 |      
   RawArrayForEach | 100000 |    45.2560 us |   0.5464 us |   0.04 |      0.00 |  1.97 |  0.04 |     - |             719.81 |
       RawArrayFor | 100000 |    45.4073 us |   4.4140 us |   0.05 |      0.01 |  1.97 |  0.04 |     - |             716.41 |


<br/>

```c#
    [Benchmark(Baseline = true)]
    public int LinkedListLinq()
    {
        var local = linkedList;
        return local.Sum();
    }

    [Benchmark]
    public int LinkedListForEach()
    {
        var local = linkedList;
        int sum = 0;
        foreach (var node in local)
        {
            sum += node;
        }
        return sum;
    }

    [Benchmark]
    public int LinkedListFor()
    {
        var local = linkedList;
        int sum = 0;
        var node = local.First;
        for (int i = 0; i < local.Count; i++)
        {
            sum += node.Value;
            node = node.Next;
        }
        return sum;
    }

    [Benchmark]
    public int ArrayListFor()
    {
        var local = arrayList;
        int sum = 0;
        for (int i = 0; i < local.Count; i++)
        {
            sum += local[i];
        }
        return sum;
    }

    [Benchmark]
    public int ArrayListForEach()
    {
        var local = arrayList;
        int sum = 0;
        foreach (var x in local)
        {
            sum += x;
        }
        return sum;
    }

    [Benchmark]
    public int RawArrayLinq()
    {
        var local = rawArray;
        return local.Sum();
    }

    [Benchmark]
    public int RawArrayForEach()
    {
        var local = rawArray;
        int sum = 0;
        foreach (var x in local)
        {
            sum += x;
        }
        return sum;
    }

    [Benchmark]
    public int RawArrayFor()
    {
        var local = rawArray;
        int sum = 0;
        for (int i = 0; i < local.Length;i++)
        {
            sum += local[i];
        }
        return sum;
    }
    
```
