---
layout: post
title: JVM Memory Tuning
tags: [java, tutorial]
comments: true
---

### Introduction

The Java Virtual Machine (JVM) can be tuned to meet your program's memory needs. I will be discussing the various memory options
along with how you can monitor the status of your Java programs.

### Two Mounds
Let's start by looking at the various types of memory that Java uses. As you may already know, Java manages memory for you automatically
(much like C# and Ruby do) via a process called garbage collection. There are two areas of memory in Java just like in C and C++. These
areas are referred to as the stack and the heap. The stack is where local primitive variables, local object references (just the memory
address/location of object data and not the actual object contents itself), and sometimes objects go (using a compile-time optimization
as of Java 6 called [escape analysis](https://en.wikipedia.org/wiki/Escape_analysis)). Also note that method parameters are considered local to the method where they are declared. When
a code block or method goes out of a scope, the local variables go away. Any local reference variables pointing to objects are also gone.
When there are no more references pointing to a particular object, the object is garbage collected and that location can be used again
by another object. Since Java emphasizes concurrent programming, it is also worth mentioning that each thread has its own stack.

The heap typically contains the bulk of data for a Java program. Objects and their static and instance members reside on the heap. Have
a look at the program below to help illustrate this information.

```java
class Example
{
  public static int total = 0; // heap
  protected int a;             // heap
  protected int b;             // heap

  public static String sumOf(int a, int b) // primitives 'a' and 'b' live on the stack
  {
    String c = "The sum is: " + (a + b); // reference 'c' lives on the stack
                                         // but, "The sum is: ..." lives on the heap
    total += a + b;
    return c;
  }

  public static void main(String[] args)
  {
    Example example = new Example();

    example.a = 5;  // example reference is on the stack and value '5' is on the heap
    example.b = 10; // example reference is on the stack and value '10' is on the heap

    System.out.println(sumOf(example.a, example.b));
  }
}
```

Remember though that objects _may_ be allocated on the stack at times depending on what the compiler decides.

### Analyzing Java Processes
You can get an idea of how much total memory (stack + heap) your program is consuming using Task Manager (`taskmgr.exe`), `ps`, `/proc`,
`pmap`, `top`, Activity Monitor, and the like. The Java Development Kit (JDK) actually contains some useful utilities for looking at
running Java programs. These utilities live under the `$JAVA_HOME/bin` directory. First off, there's `jps`. This command displays
running Java processes. It provides the PID (process ID) and name of the program. Below is an example of its output.

```bash
ldilley@develop:~$ jps
9515 Jps
9464 LinPot
ldilley@develop:~$ ps -ef | grep -v grep | grep 9464
ldilley   9464  9217  0 03:46 pts/0    00:00:00 java me.dilley.linpot.LinPot
```

Now that we have the PID of a running Java process, let's investigate it some more using the `jmap` utility. To use `jmap`, grab the
PID of a running Java process you want to attach to. You can easily find running Java processes with `jps` as shown above.

```bash
ldilley@develop:~$ jmap -heap 9464
Attaching to process ID 9464, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 24.65-b04

using thread-local object allocation.
Parallel GC with 2 thread(s)

Heap Configuration:
   MinHeapFreeRatio = 0
   MaxHeapFreeRatio = 100
   MaxHeapSize      = 698351616 (666.0MB)
   NewSize          = 1310720 (1.25MB)
   MaxNewSize       = 17592186044415 MB
   OldSize          = 5439488 (5.1875MB)
   NewRatio         = 2
   SurvivorRatio    = 8
   PermSize         = 21757952 (20.75MB)
   MaxPermSize      = 174063616 (166.0MB)
   G1HeapRegionSize = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 8912896 (8.5MB)
   used     = 1434848 (1.368377685546875MB)
   free     = 7478048 (7.131622314453125MB)
   16.098561006433822% used
From Space:
   capacity = 1048576 (1.0MB)
   used     = 0 (0.0MB)
   free     = 1048576 (1.0MB)
   0.0% used
To Space:
   capacity = 1048576 (1.0MB)
   used     = 0 (0.0MB)
   free     = 1048576 (1.0MB)
   0.0% used
PS Old Generation
   capacity = 21495808 (20.5MB)
   used     = 0 (0.0MB)
   free     = 21495808 (20.5MB)
   0.0% used
PS Perm Generation
   capacity = 22020096 (21.0MB)
   used     = 4385528 (4.182365417480469MB)
   free     = 17634568 (16.81763458251953MB)
   19.916025797526043% used

1131 interned Strings occupying 76888 bytes.
```

`jmap` is handy since it provides a detailed view of the heap. Don't worry if this output looks unintelligible. We shall decipher it
together. We'll skip the thread-local object allocation setting since it is outside the scope of this post. It is enabled by default
and can be toggled with the `-XX:UseTLAB` option. You can read more about it [here](https://blogs.oracle.com/jonthecollector/the-real-thing)
if you are interested.

### Taking Out the Trash
Next comes the garbage collection method. My Java 7 JVM is using parallel garbage collection by default. This method uses multiple
threads to perform collection. It can be explicitly enabled with the `-XX:+UseParallelGC` option. The JVM also supports "Mark Sweep
Compact Garbage Collection" which is a serial collection method recommended for systems with a single processor or for applications
with small amounts of data. `-XX:+UseSerialGC` is used to enable it. Lastly, `-XX:+UseConcMarkSweepGC` enables the concurrent collector.
This collection method aims to increase application response time by minimizing pausing. If concurrent collection incurs significant
pausing, you can enable incremental mode with `-Xincgc` which should shorten the pauses.

To summarize:

#### Serial Garbage Collection
* Enabled with `-XX:+UseSerialGC`
* Also called "Mark Sweep Compact Garbage Collection"
* Useful on systems with a single processor
* Useful when object data does not exceed 100MB

#### Parallel Garbage Collection
* Enabled with `-XX:+UseParallelGC`
* Is similar to the serial method, but uses multiple threads for collection
* Useful on systems with multiple processors
* Number of collector threads can be controlled with `-XX:ParallelGCThreads=<number>`
* The number of collector threads should be equal to the number of system cores

#### Concurrent Garbage Collection
* Enabled with `-XX:+UseConcMarkSweepGC`
* Useful on systems with multiple processors
* Useful for low-latency applications such as 3D games
* Can be told to collect incrementally with -Xincgc which lessens pausing that may occur during collection
* `-Xincgc` is the equivalent of specifying `-XX:+UseConcMarkSweepGC` and `-XX:+CMSIncrementalMode`
* Is a good default to use

You can monitor garbage collection statistics to see which algorithm works best for your program by enabling the following JVM options:

    -verbose:gc
    -XX:+PrintGCDetails
    -XX:+PrintGCTimeStamps

I'd also like to bring up compaction. Just like filesystems on a hard disk can become fragmented, so too can the heap. You may already
know that some data structures such as arrays are allocated in memory sequentially. Let's say for the sake of simplicity that there
is 1 simple 1-byte object on the heap followed by a sequential array of 1,000 1-kilobyte elements and then 1 other simple 1-byte object.
What happens when the array is garbage collected and a new larger array needs room on the heap? It is not a very efficient use of space
to have many gaps. Compaction mitigates this issue by squeezing data closer together.

### Digging Through the Heap
As if things weren't complicated enough, Java breaks up the heap into other segments where objects reside. These segments are called
generations. Different implementations of the JVM may use different terms to describe these areas. However, they are all generational.
This means that objects essentially age. When an object is created, it is spawned in the young/new/Eden space. One exception is that
if the object is rather large, it will go directly to the tenured/old generation which typically is larger in size. If all references
to the object go away, it is marked for collection. If the object survives, it grows older and is moved to the tenured area. And as
you guessed, if an object is geriatric, it will live in the retirement housing area known as the old generation space. "PS" in this
context means "parallel scavenge" and simply reflects the garbage collection algorithm employed. If you change from parallel collection
to serial or concurrent, the number of areas and names may change.

I have not yet touched on the permanent generation area, but I will do so in the summary of heap areas below.

#### Eden/New
* Part of the young generation
* Where new objects are freshly allocated (unless they are too large -- then they go directly to the tenured/old generation)
* Used for quick allocation

#### Survivor (or "From Space" and "To Space")
* Also part of the young generation
* Objects that survive Eden/New space collection become survivors
* The "From" and "To" space roles are swapped once one area has been collected and cleared (one is usually empty)

#### Tenured/Old
* Objects that make it past the survivor phase retire to the tenured/old space to live out their days (or minutes/hours)
* This area is not as frequently collected as the Eden/New space
* Objects that are deemed too large skip the young generation altogether and go here

#### Permgen
* The permanent generation
* Loaded class information lives here such as reflective information about classes (metadata) and interned strings (until Java
7 at least)

The bit about interned strings provides details of how many string objects have been added to the string intern pool. This is an
optimization that stores unique instances of string literals thus reducing memory usage.

### Bending the JVM to Your Will
You might be wondering why we did not cover the heap configuration section of the `jmap` output yet. You may also be pondering when
and why you would want to change the way the JVM handles memory and how. Wonder and ponder no more. Read on for the respective JVM
options and the situations in which you would modify them.

#### `-Xss<number>[M]`
* Sets the thread stack size. Even if you have not created threads explicitly, your program is a single thread. If you have a large call
stack or are making heavy use of recursion, you will want to increase this value. You'll know when you start receiving `StackOverflowError`
messages.
* Example which sets the stack size to 2 megabytes: `java -Xss2M someClass`

#### `-Xms<number>[M/G] and -Xmx<number>[M/G]`
* Sets the minimum and maximum heap size. If you are loading a significant amount of objects in memory and see
`java.lang.OutOfMemoryError: Java heap space`, you'll want to increase these values.
* Example which sets the initial heap to 512M and maximum heap size to 4G: `java -Xms512M -Xmx4G someClass`

#### `-Xmn<number>[M/G]`
* Sets the total size of the young generation. ⅓ the size of -Xmx is usually a decent value. If you have frequent allocation of a
large number of objects, you should consider increasing this value.
* Example: java -Xmx3G -Xmn1G someClass

#### `-XX:PermSize=<number>[M/G] and -XX:MaxPermSize=<number>[M/G]`
* Sets the initial and maximum permanent generation sizes. If you are loading a large number of classes, you may need to bump these
values up. If you see `java.lang.OutOfMemoryError: PermGen space`, it is an indication that you need to raise at least `MaxPermSize`.
* Example: `java -XX:PermSize=64M -XX:MaxPermSize=256M someClass`

#### `-XX:NewSize=<number>[M/G] and -XX:MaxNewSize=<number>[M/G]`
* Sets the initial and maximum sizes for the young generation. Again, `MaxNewSize` should be ⅓ the size of `-Xmx`. By default,
`MaxNewSize` is extremely large. This is where new objects go.
* Example: `java -XX:NewSize=512M -XX:MaxNewSize=1G someClass`

#### `-XX:OldSize=<number>[M/G]`
* Explicitly sets the old generation size.
* Example: `java -XX:OldSize=1G someClass`

#### `-XX:NewRatio=<number>`
* Sets the ratio of space allocated to the young and old generations. For example, setting this option to a value of 2 would mean that
the maximum old generation space would be twice as large as the maximum young generation space. So, if `MaxNewSize` was 1G, the old
generation space (`OldSize`) would be 2G.
* Example of the old generation set to three times the size of the young generation: `java -XX:NewRatio=3 someClass`

#### `-XX:MinFreeHeapRatio=<number> and -XX:MaxFreeHeapRatio=<number>`
* Number is a value from 0 to 100 which represents a percentage. When less than `MinFreeHeapRatio` of the heap is available, the heap
grows up to the `-Xmx` value. When `MaxFreeHeapRatio` is reached, compaction occurs. In other words, if `MaxFreeHeapRatio` is 90%,
compaction occurs at 10% occupancy. When `-Xms` == `-Xmx`, `MinFreeHeapRatio` has no effect. Many recommend setting `MinFreeHeapRatio`
to 40% and `MaxFreeHeapRatio` to 70%.
* Example: `java -XX:MinFreeHeapRatio=40 -XX:MaxFreeHeapRatio=70 someClass`

#### `-XX:SurvivorRatio=<number>`
* This option sets the ratio between the `From/To` space (which are always equal in size) and `Eden`. So, if `SurvivorRatio` is equal
to 5, `Eden` is 5 times the size of the `From/To` space. The total parts would be 5 + 1 (`From`) + 1 (`To`) = 7.
* Example: `java -XX:SurvivorRatio=10 someClass`

### Conclusion
I hope this journey has been beneficial to you. Feel free to leave any feedback or questions. Happy JVM tweaking!

### Author and Copyright Information
Copyright © 2014 Lloyd Dilley (originally written on 9/2/2014) under the terms of the [GNU Free Documentation License v1.3](https://www.gnu.org/licenses/fdl-1.3.txt)
