---
layout:     post
title:      Quicker Clojure startup with AppCDS and AOT
date:       2017-10-04 21:55:19
summary:    Application CDS for quick Clojure startup
tags:
- jvm
- clojure
---

I found [previously](http://mjg123.github.io/2017/10/02/JVM-startup.html) that CDS and AOT can have a dramatic effect when used to speed up JVM startup for a simple "Hello World". Now, I want to see how effective that actually is in a more realistic setting. I chose to run [Clojure](http://clojure.org), because Clojure itself is quite large and complex in what it does, but very simple to invoke. It also gets a [lot](https://purelyfunctional.tv/article/the-legend-of-long-jvm-startup-times/) [of](https://dev.clojure.org/display/design/Improving+Clojure+Start+Time) [flak](https://purelyfunctional.tv/article/how-do-clojure-programmers-deal-with-long-startup-times/) about [startup](https://www.reddit.com/r/Clojure/comments/5zq45b/what_are_the_things_that_you_dont_like_in_clojure/df04iis/?st=j8d6uybf&sh=9186116c) time. Also, I ♡ Clojure.

Anyway the techniques here apply to any JVM workload. Let's go.

* TOC
{:toc}

### Starting out

Head over to the [Clojure Downloads page](https://clojure.org/community/downloads) and get the latest stable release. Now we can run:

```shell
⇒ java -cp clojure-1.8.0.jar clojure.main -e '(println :hello-clojure)'
:hello-clojure
```

OK I admit it's still "Hello world", but there's a lot of bytecode going on behind the scenes there.

Please see the [previous post](http://mjg123.github.io/2017/10/02/JVM-startup.html) for more details about using `perf` to record execution times. I'm only concerning myself with absolute time-taken and cpu clock time. Every command is run 50x and results aggregated.

### Raw invocation

```
Performance counter stats for 'java -cp clojure-1.8.0.jar clojure.main -e (println :hello-clojure)' (50 runs):
       1587.316794      cpu-clock (msec)          #    1.463 CPUs utilized            ( +-  0.78% )
       1.084688353 seconds time elapsed                                          ( +-  1.10% )
```

### Using CDS

Generate the [Class Data Sharing](https://docs.oracle.com/javase/9/vm/class-data-sharing.htm#JSJVM-GUID-7EAA3411-8CF0-4D19-BD05-DF5E1780AA91) cache:

```
$ java -Xshare:dump
```

then

```
Performance counter stats for 'java -Xshare:on -cp clojure-1.8.0.jar clojure.main -e (println :hello-clojure)' (50 runs):
       1504.706806      cpu-clock (msec)          #    1.488 CPUs utilized            ( +-  1.12% )
       1.011034802 seconds time elapsed                                          ( +-  1.53% )
```

This is a nearly 7% improvement, but if you read the post before you should know we're hoping for more than that! The benefit of regular CDS is going to be greater when your app loads fewer classes - ie when the core Java classes make a large proportion of your app. So for a larger app we need to look beyond regular CDS.

Time to learn about *Application CDS!*

### Application CDS

Application CDS, [described here](https://bugs.openjdk.java.net/browse/JDK-8185996), (I'll call it AppCDS from now on) is like regular CDS - it performs bytecode parsing and verification then caches the result to eliminate the fixed cost of doing that at every JVM startup. But this time it does it to any classes you specify, not just core Java classes.

**NB:** AppCDS is currently a "commercial" feature of Java, which means that you should not use it *in production* unless you have paid for a license. Experimenting, and using it in development is clearly not "production". Furthermore, at Java One in Oct 2017 [Mark Cavage](https://www.youtube.com/watch?v=UNg9lmk60sg) confirmed that Oracle has committed to opening up all the commercial features of the Oracle JVM, as [proposed by Mark Reinhold](http://mail.openjdk.java.net/pipermail/discuss/2017-September/004281.html) the previous month. So given that it *will* be opened up, I think it's something worth looking at right away.

EDIT: JVM team member Ioi Lam has [proposed a patch](http://mail.openjdk.java.net/pipermail/hotspot-runtime-dev/2017-October/024831.html) which moves AppCDS into OpenJDK.

With that said, lets do it. Create a list of loaded classes:

```
⇒ java -XX:+UnlockCommercialFeatures \
       -XX:+UseAppCDS \
       -Xshare:off \
       -XX:DumpLoadedClassList=appcds.classlist \
       -cp clojure-1.8.0.jar clojure.main -e '(println :hello-clojure)'
```

Quick inspection of where the classes are loaded from:

```
⇒ cat appcds.classlist | cut -d/ -f1 | sort | uniq -c
    1393 clojure
    742 java
    141 jdk
    142 sun

```

That seems about right. Then, lets create the AppCDS cache:

```
⇒ java -XX:+UnlockCommercialFeatures \
       -XX:+UseAppCDS \
       -Xshare:dump \
       -XX:SharedClassListFile=appcds.classlist \
       -XX:SharedArchiveFile=appcds.cache \
       -cp clojure-1.8.0.jar clojure.main -e '(println :hello-clojure)'
Allocated shared space: 50577408 bytes at 0x0000000800000000
Loading classes to share ...
Loading classes to share: done.
Rewriting and linking classes ...
Rewriting and linking classes: done
Number of classes 2434
    instance classes   =  2420
    obj array classes  =     6
    type array classes =     8
Updating ConstMethods ... done. 
Removing unshareable information ... done. 
ro space:   7024952 [ 27.6% of total] out of  10485760 bytes [ 67.0% used] at 0x0000000800000000
rw space:   9337400 [ 36.7% of total] out of  10485760 bytes [ 89.0% used] at 0x0000000800a00000
md space:    225968 [  0.9% of total] out of   4194304 bytes [  5.4% used] at 0x0000000801400000
mc space:     34053 [  0.1% of total] out of    122880 bytes [ 27.7% used] at 0x0000000801800000
st space:     12288 [  0.0% of total] out of     12288 bytes [100.0% used] at 0x00000007bff00000
od space:   8780248 [ 34.5% of total] out of  20971520 bytes [ 41.9% used] at 0x000000080181e000
total   :  25414909 [100.0% of total] out of  46272512 bytes [ 54.9% used]

⇒ ls -lh appcds.cache
-r--r--r-- 1 mjg mjg 25M Oct  4 14:49 appcds.cache
```

Use the AppCDS cache with a combination of:
  * `-XX:+UnlockCommercialFeatures`
  * `-XX:+UseAppCDS`
  * `-Xshare:on`
  * `-XX:SharedArchiveFile=appcds.cache`

And lets see how it performs.

```
Performance counter stats for 'java -XX:+UnlockCommercialFeatures -XX:+UseAppCDS -Xshare:on -XX:SharedArchiveFile=appcds.cache -cp clojure-1.8.0.jar clojure.main -e (println :hello-clojure)' (50 runs):
        829.748380      cpu-clock (msec)          #    1.545 CPUs utilized            ( +-  1.18% )
       0.537133681 seconds time elapsed                                          ( +-  1.42% )
```

Impressive! We've cut execution time in half! And it was pretty straightforward and quick to do.

We could stop here, but I previously found that we could use AOT with CDS at the same time and get even better results. AOT is rather more tricky to get right (at the moment).

### AOT

I described JDK9 Ahead-Of-Time compilation in my previous post. It is bytecode → native code compilation, **not** the same as AOT for Clojure (clojure source → bytecode).  We also saw how to generate an AOT cache [before](http://mjg123.github.io/2017/10/02/JVM-startup.html). There's a couple of gotchas here though:

  * You need to remove the classes that Clojure generates at runtime from the loaded classes list. They're called `fn__<some-number>`
  * You need to specify `--jar clojure-1.8.0.jar` *and* `-J-cp -Jclojure-1.8.0.jar` to `jaotc`, so that it considers it a source for classes *and* Graal has it on the classpath.
  
For creating the `touched.aotcfg` file using the process described in [my previous post](http://mjg123.github.io/2017/10/02/JVM-startup.html), I did this:

```shell
# Generate a list of the methods used
⇒ java -XX:+UnlockDiagnosticVMOptions \
       -XX:+LogTouchedMethods \
       -XX:+PrintTouchedMethodsAtExit \
       -cp clojure-1.8.0.jar clojure.main -e '(println :hello-clojure)' > touched_methods

# Filter & reformat the list
⇒ grep -v '^#' touched_methods | \
  grep -v 'hello-clojure' | \
  grep -v 'fn__' | \
  grep -v jdk/internal/module/SystemModules.hashes | \
  grep -v jdk/internal/module/SystemModules.descriptors | \
  sed -e 's/^/compileOnly /' | \
  java Convert > touched.aotcfg
```

[Convert.java](https://gist.githubusercontent.com/mjg123/d06fd39ce1aedb3032edd463e814cdd1/raw/c9497e6a1da5a656d181986ac094ba4d95a88dfa/Convert.java) is a small tool for reformatting class names. Honestly I find all this a bit of a chore, so I hope that some tooling is made to simplify it. Generally, `jaotc` will report an error for a line it doesn't understand, but the whole operation will still generate an AOT cache anyway. So you can afford to leave some errors in there but I am a perfectionist ;)

```shell
# Compile the AOT cache
⇒ jaotc --output touched_methods.so \
        --compile-commands touched.aotcfg \
        --module java.base \
        --jar clojure-1.8.0.jar \
        -J-cp -Jclojure-1.8.0.jar \ ## awkward
        --info
```

This generated an 86mb cache, which had the following effect on performance:

```
Performance counter stats for 'java -XX:AOTLibrary=./touched_methods.so -cp clojure-1.8.0.jar clojure.main -e (println :hello-clojure)' (50 runs):
        793.609082      cpu-clock (msec)          #    0.992 CPUs utilized            ( +-  0.48% )
       0.800247570 seconds time elapsed                                          ( +-  0.55% )
```

That's not too bad by itself either. I wondered if I could get it better by being more restrictive about what was compiled, so I tried a few different combinations:

**Only the jar** `--jar clojure-1.8.0.jar` : Cache size: 58mb, Runtime: ~1.05s

**Only the java.base module** `--module java.base` : Cache size: 28mb, Runtime: ~0.86s

**Both** `java.base` *and* `clojure-1.8.0.jar` : Cache size: 86mb, Runtime: ~0.8s (as above)

So, it seems that we might as well go with AOT for both. When you're running java with AOT you can specify multiple files, like this:

```
java -XX:AOTLibrary=./clojure-jar.so:./java-base.so ...
```

or one big one:

```
java -XX:AOTLibrary=./both.so ...
```

Doesn't seem to have any performance impact either way. Up to you.

### AppCDS and AOT

As before, using the AOT cache from `java.base` together with the AppCDS cache has a greater effect than using either individually:

```
Performance counter stats for 'java -XX:AOTLibrary=./both.so -XX:+UnlockCommercialFeatures -XX:+UseAppCDS -Xshare:on -XX:SharedArchiveFile=appcds.cache -cp clojure-1.8.0.jar clojure.main -e (println :hello-clojure)' (50 runs):
        409.418726      cpu-clock (msec)          #    1.000 CPUs utilized            ( +-  0.53% )
       0.409351100 seconds time elapsed                                          ( +-  0.74% )
```

Again, this is a really sizeable improvement over the plain `java` invocation:
  * **62%** reduction in wall clock time
  * **74%** reduction in total CPU time spent

There's bound to be some variance depending on hardware/machine-load/phase-of-moon, but the improvements are not small, really. Perhaps some of these techniques can be built into Clojure tooling in the future?

### Acknowlegements

Thanks again to the JVM team here at Oracle, especially Claes Redestad for proofreading and teaching me the technical details. I learned a lot about AppCDS from Kim Kinnear's [zprint](https://github.com/kkinnear/zprint).
