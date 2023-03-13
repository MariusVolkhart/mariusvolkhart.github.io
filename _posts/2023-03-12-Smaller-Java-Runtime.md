---
title: "Smaller Java runtime"
categories:
  - JVM
tags:
  - Java
last_modified_at: 2023-03-12T21:40:00-06:00
---

I asked my colleagues at PKWARE what they dislike most about the JVM. One commented, "relying on a (rather large) runtime." I read that and though, "challenge accepted!"

{% include toc %}

Now, a "large runtime" could mean 2 things: installation size of the runtime, or RAM requirements of the runtime. I'll see what we can do about both. To test the runtime memory requirements, I'm going to run a standard "Hello World" program using `/usr/bin/time -l java HelloWorld` and report on the maximum resident set size (RSS).

# Baseline
We need a baseline to compare against. I'm going to be building OpenJDK in various configurations, so as a baseline, I'll compile using the default configuration, `./configure`, which results in setup `server-release`. Compiled for MacOS, this produces a JDK folder that's 515.2MB in size. Running the programm, I get a maximum RSS of 33.29MB.

# Low-hanging fruit
The obvious first-step is to apply `jlink`. `jlink` is a tool to build a Java Runtime Environment (JRE) from an existing JDK. The JDK contains many things not needed at runtime: `javac`, `jarsigner`, `jshell`, etc. Those tools can be removed. Furthermore, the full JDK contains many modules, that, in other langauges, are required to be pulled in as dependencies. For example, `java.xml`, `java.logging`, `jdk.httpserver`. To keep the comparison reasonable, I'm only going to include `java.base`. This already gives the application access to a large standard library, and lets simple apps work.

The command I'll use is `jlink --compress=zip-6 --no-header-files --no-man-pages --strip-debug --output <path> --module-path <module path> --add-modules java.base`. This results in a JRE folder that's 30MB in size. That's 6% the size of the full JDK! The amazing thing about this "repackaging" is that's it's available to nearly everyone. Whether you're deploying a Docker image, sending a binary to a customer, or just want to make better use of VM storage, `jlink` is an amazing tool for shrinking the storage required for your Java app.

Doing the maximum resident set size check leads to 32.82MB. Not much of an improvement.

# Going further
 `jlink` is cool, but we started with a pretty heavy JDK. The default `server` target includes Class Data Sharing, the C1 and C2 JIT compilers, `dtrace` support, 6 different GCs (Epsilon, G1, Parallel, Serial, Shenandoah, ZGC), and support for various tooling like JNI, management, debugging, etc. August Nagro has a [fantastic blog](https://august.nagro.us/small-java.html) post linking to explanations of many of these. My little "Hello World" app doesn't use most of these things. So... what if I got rid of them?

## Going minimal
I need to reconfigure the build. `./configure --with-jvm-variants=minimal`. The result will be a JDK image that has only C1 and the Serial GC. After applying `jlink`, this JRE has a size of 18MB. This time the program has a maximum RSS of 22.45MB. That's an RSS 67% the size of the plain JDK. Nifty!

Still, for comparison, a C++ Hello World program has a maximum RSS of only 1.13MB. So my colleague has a point when talking about a large runtime.

This minimal configuration is interesting for academic reasons, but it's unlikely one would use it for a real application. Of course, there are situations where this configuration _is_ the right answer. For example, serverless functions that never heat up enough to benefit from C2, or small-compute environments where minimal GC overhead is important. Still, what is the smallest image I can make that I can reasonably ship?

## A happy medium?
Let's review our BOM for this image:
- G1GC is a reasonable option for a wide variety of applications and use cases. I'll use G1 as my garbage collector.
- I also want C2. The types of deployments we have at PKWARE benefit from the performance gains C2 can deliver.
- I'm also going to include JFR. This adds such valuable diagnostics. At PKWARE, we deploy at customer sites, so can't make use of interactive profilers or management tools (which means I can leave those components out). JFR is hugely valuable here.

`./configure --with-jvm-variants=custom -with-jvm-features=cds,compiler1,compiler2,g1gc,jfr,jvmti,serialgc,services`

The resulting JRE has a size of 27.1MB and a maximum RSS comparable to that of the JRE from the full `server` variant. I think it's safe to say that building a custom JDK isn't really worth it for typical production scenarios.

# Conclusion
Compared to something like a minimal C++ program, that Java runtime is big, both in storage and RAM. However, the large storage requirement of the JDK can be drastically reduced for application deployments by using `jlink`, though there's not much to be done about the RAM requirements. That said, it's important to note that comparing to a C++ program is like comparing apples to oranges. The JVM runtime comes with a JIT, continuous production profiler, excellent debugging and instrumentation support, and more. Those features don't matter for simple programs, but for application teams deploying large apps, they provide valuable developer productivity.

Much thanks to my colleague for pushing me to do this investigation!