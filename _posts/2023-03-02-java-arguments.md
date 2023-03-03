---
layout: post
title: Tactfully Handling Common Java Complaints
tags: [java, tutorial]
comments: true
---

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

**C:**
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

**Java:**
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

The C code executes in approximately 75µs on my system. In comparison, the Java equivalent takes about 40,000µs on the same system.
That is nearly 533 times longer than the C program! Why? Well, the JVM "warm-up" time needs to be considered. If we make the
following changes to the `main()` method to disregard warm-up time, we get a more reasonable execution time of approximately 1,318µs:

```java
    long startTime = System.nanoTime();
    int[] arr1 = new int[5000];
    int[] arr2 = new int[5000];
    fillRand(arr1);
    fillRand(arr2);
    addArr(arr1, arr2);
    long endTime = System.nanoTime();
    System.out.println((endTime - startTime) / 1000 + "µs"); // microseconds
```

That isn't so bad. In fact, that is where JIT shines for long-running and commonly-executed code. If we arbitrarily loop over the
C program 100 times, the execution time will be similar for each iteration whereas the Java program should become faster with each
iteration (until a peak is achieved). Modifying `main()` in both programs to calculate and print the runtime, in microseconds, of
each iteration demonstrates this.

**C:**
```c
#include <stdlib.h>   /* rand() and srand() */
#include <stdio.h>    /* printf() */
#include <sys/time.h> /* gettimeofday() */
#include <time.h>     /* time_t */

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
  struct timeval stop, start;
  for(int i = 0; i < 100; i++)
  {
    int arr1[5000];
    int arr2[5000];
    time_t t;
    gettimeofday(&start, NULL);
    srand((unsigned)time(&t));
    fill_rand(arr1, 5000);
    fill_rand(arr2, 5000);
    add_arr(arr1, arr2, 5000);
    gettimeofday(&stop, NULL);
    printf("Iteration %d: %luµs\n", i, ((stop.tv_sec - start.tv_sec) * 1000000 + stop.tv_usec - start.tv_usec));
  }
  return 0;
}
```

**Java:**
```java
  public static void main(String[] args) throws InterruptedException
  {
    for(int i = 0; i < 100; i++)
    {
      long startTime = System.nanoTime();
      int[] arr1 = new int[5000];
      int[] arr2 = new int[5000];
      fillRand(arr1);
      fillRand(arr2);
      addArr(arr1, arr2);
      long endTime = System.nanoTime();
      System.out.println("Iteration " + i + ": " + (endTime - startTime) / 1000 + "µs");
    }
  }
```

**C Output:**
```bash
[lloyd@lindev ~]$ ./arr
Iteration 0: 76µs
Iteration 1: 70µs
Iteration 2: 70µs
Iteration 3: 70µs
Iteration 4: 70µs
Iteration 5: 80µs
Iteration 6: 70µs
Iteration 7: 70µs
Iteration 8: 69µs
Iteration 9: 70µs
Iteration 10: 69µs
Iteration 11: 70µs
Iteration 12: 69µs
Iteration 13: 69µs
Iteration 14: 70µs
Iteration 15: 70µs
Iteration 16: 83µs
Iteration 17: 70µs
Iteration 18: 70µs
Iteration 19: 69µs
Iteration 20: 70µs
Iteration 21: 69µs
Iteration 22: 70µs
Iteration 23: 70µs
Iteration 24: 70µs
Iteration 25: 70µs
Iteration 26: 70µs
Iteration 27: 80µs
Iteration 28: 69µs
Iteration 29: 69µs
Iteration 30: 69µs
Iteration 31: 69µs
Iteration 32: 70µs
Iteration 33: 70µs
Iteration 34: 70µs
Iteration 35: 70µs
Iteration 36: 70µs
Iteration 37: 70µs
Iteration 38: 100µs
Iteration 39: 70µs
Iteration 40: 70µs
Iteration 41: 70µs
Iteration 42: 70µs
Iteration 43: 76µs
Iteration 44: 70µs
Iteration 45: 70µs
Iteration 46: 71µs
Iteration 47: 70µs
Iteration 48: 71µs
Iteration 49: 80µs
Iteration 50: 71µs
Iteration 51: 70µs
Iteration 52: 71µs
Iteration 53: 70µs
Iteration 54: 73µs
Iteration 55: 70µs
Iteration 56: 70µs
Iteration 57: 69µs
Iteration 58: 70µs
Iteration 59: 69µs
Iteration 60: 99µs
Iteration 61: 70µs
Iteration 62: 70µs
Iteration 63: 70µs
Iteration 64: 69µs
Iteration 65: 74µs
Iteration 66: 70µs
Iteration 67: 70µs
Iteration 68: 70µs
Iteration 69: 71µs
Iteration 70: 70µs
Iteration 71: 79µs
Iteration 72: 70µs
Iteration 73: 70µs
Iteration 74: 70µs
Iteration 75: 70µs
Iteration 76: 73µs
Iteration 77: 70µs
Iteration 78: 69µs
Iteration 79: 70µs
Iteration 80: 69µs
Iteration 81: 69µs
Iteration 82: 69µs
Iteration 83: 69µs
Iteration 84: 71µs
Iteration 85: 70µs
Iteration 86: 70µs
Iteration 87: 70µs
Iteration 88: 78µs
Iteration 89: 70µs
Iteration 90: 70µs
Iteration 91: 71µs
Iteration 92: 70µs
Iteration 93: 70µs
Iteration 94: 80µs
Iteration 95: 71µs
Iteration 96: 70µs
Iteration 97: 70µs
Iteration 98: 70µs
Iteration 99: 70µs
```

**Java Output:**
```bash
[lloyd@lindev ~]$ java Arr
Iteration 0: 1357µs
Iteration 1: 422µs
Iteration 2: 431µs
Iteration 3: 419µs
Iteration 4: 440µs
Iteration 5: 413µs
Iteration 6: 437µs
Iteration 7: 150µs
Iteration 8: 137µs
Iteration 9: 141µs
Iteration 10: 203µs
Iteration 11: 152µs
Iteration 12: 161µs
Iteration 13: 104µs
Iteration 14: 80µs
Iteration 15: 109µs
Iteration 16: 105µs
Iteration 17: 80µs
Iteration 18: 95µs
Iteration 19: 331µs
Iteration 20: 104µs
Iteration 21: 100µs
Iteration 22: 125µs
Iteration 23: 178µs
Iteration 24: 104µs
Iteration 25: 408µs
Iteration 26: 99µs
Iteration 27: 99µs
Iteration 28: 79µs
Iteration 29: 79µs
Iteration 30: 78µs
Iteration 31: 80µs
Iteration 32: 78µs
Iteration 33: 83µs
Iteration 34: 79µs
Iteration 35: 78µs
Iteration 36: 79µs
Iteration 37: 78µs
Iteration 38: 79µs
Iteration 39: 78µs
Iteration 40: 79µs
Iteration 41: 79µs
Iteration 42: 79µs
Iteration 43: 79µs
Iteration 44: 79µs
Iteration 45: 79µs
Iteration 46: 79µs
Iteration 47: 79µs
Iteration 48: 55µs
Iteration 49: 54µs
Iteration 50: 60µs
Iteration 51: 55µs
Iteration 52: 54µs
Iteration 53: 55µs
Iteration 54: 54µs
Iteration 55: 55µs
Iteration 56: 56µs
Iteration 57: 54µs
Iteration 58: 55µs
Iteration 59: 55µs
Iteration 60: 64µs
Iteration 61: 55µs
Iteration 62: 54µs
Iteration 63: 55µs
Iteration 64: 55µs
Iteration 65: 54µs
Iteration 66: 55µs
Iteration 67: 54µs
Iteration 68: 55µs
Iteration 69: 55µs
Iteration 70: 64µs
Iteration 71: 56µs
Iteration 72: 56µs
Iteration 73: 55µs
Iteration 74: 55µs
Iteration 75: 54µs
Iteration 76: 56µs
Iteration 77: 61µs
Iteration 78: 75µs
Iteration 79: 134µs
Iteration 80: 64µs
Iteration 81: 79µs
Iteration 82: 60µs
Iteration 83: 80µs
Iteration 84: 61µs
Iteration 85: 1045µs
Iteration 86: 55µs
Iteration 87: 74µs
Iteration 88: 54µs
Iteration 89: 78µs
Iteration 90: 56µs
Iteration 91: 75µs
Iteration 92: 56µs
Iteration 93: 75µs
Iteration 94: 46µs
Iteration 95: 48µs
Iteration 96: 47µs
Iteration 97: 56µs
Iteration 98: 45µs
Iteration 99: 45µs
```

The Java code within the loop body eventually executes faster than the equivalent C code due to JVM runtime optimizations.

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
