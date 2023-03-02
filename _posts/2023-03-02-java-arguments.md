---
layout: post
title: Tactfully Handling Common Java Complaints
tags: [java, tutorial]
comments: true
---

## Tactfully Handling Common Java Complaints 

### Java Sucks! ... Or Not?
So, your friends tell you Java blows chunks. They've either heard/read it elsewhere, had a bad history with slow-loading applets
in 1996, or have personally worked with the language and loathe it. I'd like to dispel some of the arguments against Java in the
modern age since I think it is a decent language. Let's start with some common things we hear or see about Java...

### Java is Slow!
I remember loading applets on websites in Windows 95 and absolutely detesting the experience. Of course, this was on a 486 SX operating
at 25 MHz with 4M of RAM and a 14.4Kbps modem. Java was also still in its infancy. Many people had similar experiences and the rancor
spread. Unfortunately, such sentiment still exists today -- 20 years later. I am even guilty. A coworker in 2006 had an O'Reilly Java
book he offered to lend me. I declined the offer and poked fun at him for even suggesting such a thing. Fast forward two years and I
had become a Java fan. It was not until I was forced to learn the language in college that I developed an appreciation for it. Let's
review some facts:

1. Hardware has improved since the 1990s. Processors are faster and have more cache and registers. Bus speeds, disks, memory, and
network access are also faster which improve program load times.
2. The JVM has improved since the 1990s. HotSpot/JIT (just-in-time compilation), JNI (Java Native Interface), and other features
have been added. There is also an array of garbage collection algorithms to choose from depending on your application.

With the convergence of #1 and #2 above, Java performance has come a long way. However, in many (not all) cases, C and C++ programs
still handily beat Java in the performance department. Have a look at the matrix addition performance comparison below. The code
is functionally equivalent in both the C and Java programs. Two one-dimensional arrays are filled with random numbers ranging from
0 to 65,535. The sum of the elements at the current index in both arrays are stored at the current position in the first array.

```c
#include <stdlib.h> /* rand() and srand() */
#include <time.h>   /* time_t */

void fill_rand(int *arr, int length)
{
  int i;
  for(i = 0; i < length; i++)
    arr[i] = rand() % 65535 + 1;
  return;
}

void add_arr(int *arr1, int *arr2, int length)
{
  int i;
  for(i = 0; i < length; i++)
    arr1[i] += arr2[i];
  return;
}

int main()
{
  int arr1[5000];
  int arr2[5000];
  time_t t;
  srand((unsigned)time(&t));
  fill_rand(arr1, 5000);
  fill_rand(arr2, 5000);
  add_arr(arr1, arr2, 5000);
  return 0;
}
```

```java
import java.util.Random;

class Arr
{
  static void fillRand(int[] arr)
  {
    Random rand = new Random();
    for(int i = 0; i < arr.length; i++)
      arr[i] = rand.nextInt(65535) + 1;
  }

  static void addArr(int[] arr1, int[] arr2)
  {
    for(int i = 0; i < arr1.length; i++)
      arr1[i] += arr2[i];
  }

  public static void main(String[] args)
  {
    int[] arr1 = new int[5000];
    int[] arr2 = new int[5000];
    fillRand(arr1);
    fillRand(arr2);
    addArr(arr1, arr2);
  }
}
```

The C code executes in approximately 4ms. In comparison, the Java equivalent takes about 207ms. That is over fifty times longer than
the C program. Why? Well, the JVM "warm-up" time needs to be considered. If we make the following changes to the `main()` method to
disregard warm-up time, we get a more reasonable execution time of about 6ms:

```java
    long startTime = System.nanoTime();
    int[] arr1 = new int[5000];
    int[] arr2 = new int[5000];
    fillRand(arr1);
    fillRand(arr2);
    addArr(arr1, arr2);
    long endTime = System.nanoTime();
    System.out.println((endTime - startTime) / 1000000); // get ms
```

That isn't so bad. In fact, that is where JIT shines for long-running and commonly-executed code. If we arbitrarily loop over the
C program 100 times and sleep 1 second between iterations, the execution time will be similar for each iteration whereas the Java
program should become faster (until a peak is achieved.) Modifying `main()` demonstrates this:

```java
  public static void main(String[] args) throws InterruptedException
  {
    for(int i = 0; i < 100; i++)
    {
      Thread.sleep(1000);
      long startTime = System.nanoTime();
      int[] arr1 = new int[5000];
      int[] arr2 = new int[5000];
      fillRand(arr1);
      fillRand(arr2);
      addArr(arr1, arr2);
      long endTime = System.nanoTime();
      System.out.println("Iteration " + i + ": " + (endTime - startTime) / 1000000 + "ms");
    }
  }
```

```bash
[lloyd@lindev ~]$ java Arr
Iteration 0: 6ms
Iteration 1: 3ms
Iteration 2: 3ms
Iteration 3: 3ms
Iteration 4: 3ms
Iteration 5: 3ms
Iteration 6: 3ms
Iteration 7: 2ms
Iteration 8: 2ms
Iteration 9: 2ms
Iteration 10: 2ms
Iteration 11: 2ms
Iteration 12: 1ms
Iteration 13: 1ms
Iteration 14: 1ms
Iteration 15: 1ms
Iteration 16: 1ms
Iteration 17: 1ms
Iteration 18: 1ms
Iteration 19: 0ms
Iteration 20: 0ms
Iteration 21: 0ms
Iteration 22: 0ms
Iteration 23: 0ms
Iteration 24: 0ms
Iteration 25: 0ms
Iteration 26: 0ms
Iteration 27: 0ms
Iteration 28: 0ms
Iteration 29: 0ms
Iteration 30: 0ms
Iteration 31: 0ms
...
```

The code within the loop body eventually executes faster than the equivalent C code due to JVM runtime optimizations. Even though the
program reports a time of 0ms, it obviously still takes micro/nanoseconds to compute which are truncated off. 

### Java is Insecure!
As with most programs written in C/C++ (Apache HTTPD, ISC BIND, OpenSSL, etc.) there are vulnerabilities detected periodically for Java.
These are primarily due to the potential dangers of inappropriate pointer use or from undersized buffers which allow overflows. The
Java language itself features policies (via security manager) you can manipulate to effectively sandbox an application. This isolation
limits what the program can do with resources such as disks and network access. Another thing to consider is that the JVM is an entire
platform and is fairly sizable. The JVM needs to define types, abstract networking and GUI elements, and more. There is obviously an
increased risk of bugs the more lines of code a program contains. For most Windows installations, Java even nags you when updates are
available or takes care of updating itself automagically. In summary for this section, Java has admittedly had a large number of
vulnerabilities over the years. Battling exploits is a part of life that IT folks must deal with. On the bright side, operating systems,
web browsers, and Adobe Flash seem to have more vulnerabilities and keeping Java up-to-date is relatively easy. 

### Java is Bloated!
Java can handily consume a substantial amount of memory. I had 4M of RAM in 1994 or so. It is fairly common for users to have 8G
or 16G these days. Of course, that is no reason to be wasteful. But, one must again consider that the JVM is an entire platform with
a large set of features. The memory overhead that accompanies this should be expected. There are many things you can do to tune
JVM memory usage. To put things into perspective, I am running a Tomcat instance, a Jetty instance, and one more JVM instance for
a JRuby daemon on a virtual private server with 2G of memory (which is also running a plethora of other services) without breaking a
sweat. Java also runs on many less-powerful mobile and embedded devices. To recap: Memory is plentiful and fairly cheap these days,
the JVM can be tuned to use less memory, and don't be such a tightwad!

### Why Java is Annoying
* Java language lawyers who believe the JLS (Java Language Specification) is the only thing that matters. To them, memory addresses
do not exist...
* Unlike Ruby, everything is not an object (primitives like `byte`, `short`, `int`, `long`, etc.)
* No explicit pointers
* The library is too big
* Calls to `System.gc()` are only suggestions that can be ignored
* Cannot explicitly call deconstructors
* Forced to put `main()` method in a class
* Syntax can be very verbose/repetitive
* No operator overloading
* No multiple inheritance
* It can be a hog unless you cap the heap
* No native way to become a daemon or service
* Others?

### Why Java is Awesome
* It picks up after you
* Not having to deal with explicit pointers
* Huge library
* No operator overloading
* No multiple inheritance
* Portable networking and GUI code
* Largely portable for most other things/compile once run anywhere
* No need for `sizeof`
* It's ubiquitous
* Multithreading
* Others?

### Author and Copyright Information
Copyright © 2015 Lloyd Dilley (originally written on 12/20/2015) under the terms of the [GNU Free Documentation License v1.3](https://www.gnu.org/licenses/fdl-1.3.txt)
