---
layout: post
title:  "Does Your Code Leave a Trail of Slowness?"
date:   2017-02-27 14:17:27 -0500
categories: 
---
# The Trail of Slowness

Often I implore people to make better performing coding decisions when the downsides
are small or nonexistent. A common response to this is that you should only worry
about performance when you have measured the code in question and found performance to 
be an issue.  There a couple of problems with that ethos, the first of which is that
sometimes early decisions will be hard to reverse late in a project if performance turns
out to be an issue. But there is a more insidious problem.  Areas of your code for which
performance is not important, may be causing *other* code, or even *other programs* to slow
down. This trail of slowness left behind by uninformed performance decisions will not
show up in any particular place in a profiler, they will just slow down everything a bit.

# Why Does This Happen

Modern CPU performance is severely limited by RAM latency. A request to get data from RAM
can take over 100 CPU cycles to complete, and while your CPU waits, it just sits there, doing
nothing useful. This problem is addressed by a series of caches in your CPU, smaller but much 
faster regions of memory.  Whenever you request data from RAM, you get back an entire cache line
of contiguous memory into the CPU caches.  Assuming that the next memory your request is in that
contiguous segment of memory in the cache line, you will get it very quickly.

This is why iterating over an array in order (contiguous memory) is much, much faster than iterating over
a LinkedList, where each node is in a random(ish) location in the heap.  Many of those requests
from RAM while iterating over a LinkedList miss the cache, and cause long CPU stalls.

But it is worse than just being slow. Every time you request memory that isn't in the cache
already, you have to pull down a whole cache line and replace data that is already in the cache.  Any code running
that might have hoped to use that cached data won't have it, and will now have to get it from RAM
again.  This is sometimes called "thrashing the cache".

# How Bad Can It Be?

This can vary wildly based on the workloads involved. In the contrived example below, I do a 
fixed amount of work with a LinkedList, while at the same time I have other threads running
doing as much work as possible for 5 seconds. They are able to perform the ArraySumSquare function
about 3.5 million times in that 5 seconds.

When I alter the RunLinkedList function to do the same fixed amount of work on arrays instead,
the original array loops are now able to perform ArraySumSquare about 4.4 million times.

This could be similar to a situation where a server has a periodic computation it does, where
taking a few seconds isn't considered a problem at all.  But that job is having a significant
impact on users using the live system, increasing latency by a ~20%. Or it could be similar to a code editor that
is doing some parsing behind the scenes where the performance seems fine when you profile it, but it is 
slowing down UI response due to the cache thrashing.

# What To Do

Use arrays or array backed lists by default unless you have good reasons not to. Most languages have resize-able array backed 
structures that are as convenient to use as LinkedLists: 

* Java - ArrayList
* C# - List<T>
* F# - ResizeArray (see FSharpX for higher order functions on these!)
* C++ - std::vector
* Rust - vec

Even if you are inserting or removing from the front or middle of your collection periodically it is usually still faster overall.  You may also
consider using a sorted array along with BinarySearch instead of a tree, when applicable.  Think about how you lay out your data structures,
avoid unnecessary pointer hops. Avoid virtual functions when they aren't really necessary. Understand how branch prediction works so you
can set your code up for success. [Take some time to learn how memory works](https://www.akkadia.org/drepper/cpumemory.pdf), and you can make better default decisions, and better designs
early in your projects.


# Sample Code
```c#
using System;
using System.Collections.Generic;
using System.Diagnostics;
using System.Threading;

namespace ConsoleApplication1 {
    class Program {
         
        static void Main(string[] args) {
            Thread [] arrayThreads = new Thread[8];
            Thread [] listThreads = new Thread[8];

            for (int i = 0; i < 8; i++) {
                arrayThreads[i] = new Thread(RunArray);
                listThreads[i] = new Thread(RunLinkedList);
                arrayThreads[i].Start();
                listThreads[i].Start();
            }

            for (int i = 0; i < 8; i++) {
                arrayThreads[i].Join();
                listThreads[i].Join();                
            }

            Console.WriteLine("Perf Critical Ops:" + totalOps);                        
            Console.ReadLine();

        }

        public static int totalOps = 0;
        public const int LEN = 10000;
        public const int TIME = 5000;

        public static void RunArray() {
            Stopwatch sw = new Stopwatch();
            sw.Start();
            int[] a = new int[LEN];
            for (int i = 0; i < a.Length; i++) {
                a[i] = 2;
            }
            long sum = 0;
            int count = 0;
            while (sw.ElapsedMilliseconds < TIME) {                
                sum += ArraySumSquare(a);
                count++;
            }
            Interlocked.Add(ref totalOps, count);                        
        }

        public static int ArraySumSquare(int[] a) {                        
            int sum = 0;
            for (int i = 0; i < a.Length; i++) {
                sum += a[i] * a[i];
            }
            return sum;
        }

        public static void RunLinkedList() {
            Stopwatch sw = new Stopwatch();
            sw.Start();
            LinkedList<int> l = new LinkedList<int>();
            for (int i = 0; i < LEN; i++) {
                l.AddLast(2);
            }
            long sum = 0;
            int count = 0;
            while (count < 75000) {
                sum += LinkedListSumSquare(l);
                count++;
            }            
        }
        
        public static int LinkedListSumSquare(LinkedList<int> l) {            
            int sum = 0;
            foreach (var num in l) {
                sum += num * num;
            }
            return sum;
        }
    }
}
```