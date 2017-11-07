---
layout:     post
title:      Introduction to jlink
date:       2017-10-23 12:31:19
summary:    /me dips toe into jlink
tags:
- java-TIL
- jlink
- jvm
- draft
---

Naturally, Question 1 is: What is `jlink`?

[Answer](https://docs.oracle.com/javase/9/tools/jlink.htm):

> You can use the jlink tool to assemble and optimize a set of modules and their dependencies into a custom runtime image.

Indeed, it is a requirement for `jlink` that you use Java modules. There is a lot written about modules elsewhere so I won't add to the pile, just assume you know roughly what they are. One critical point: **modules have to state which other modules they depend on**. The JDK itself is now modularized with the dependencies explicit ([visualised here](https://github.com/accso/java9-jigsaw-depvis#what-is-this-about)). I shudder to think about how much hard work went into that!

If you package your code as a module then your code and its dependencies (including transitive ones) can be isolated from unused modules and a custom JVM can be created containing *only* necessary modules. That's what `jlink` does.

The rest of this post will show how to create a minimal Java module and use `jlink` to create a minimal JVM image. I'll chuck in some measurements too.

## A minimal Java module

Create a source directory

```shell
$ mkdir -p src/mjg123.module/mjg123.module
$ cat > src/mjg123.module/module-info.java
module mjg123.module {}

$ cat > src/mjg123.module/mjg123/module/Main.java
package mjg123.module;

public class Main {
  public static void main(String... args){
    System.out.println("Hello from mjg123.module");
  }
}

$ tree src 
src
└── mjg123.module
    ├── mjg123
    │   └── module
    │       └── Main.java
    └── module-info.java
```

Our module has no dependencies, except the implicit dependency on `java.base`.

Compile it:

```shell
$ javac -d mods/mjg123.module \
        src/mjg123.module/module-info.java \
        src/mjg123.module/mjg123/module/Main.java
```

We have made a module! Yeah!

```shell
$ tree mods
mods
└── mjg123.module
    ├── mjg123
    │   └── module
    │       └── Main.class
    └── module-info.class
```

## Running our module

We can run it using the `--module-path` option:

```shell
$ java --module-path mods -m mjg123.module/mjg123.module.Main
Hello from mjg123.module
```

This is using the regular `java` binary from the JDK 9.0.1 distribution, which comes with all the modules:

```shell
$ java --list-modules | wc -l
99
```

We've got ourselves a module, and I do believe we're ready to use `jlink`!

## Using jlink

```shell
$ jlink --module-path $JAVA_HOME/jmods:mods --add-modules mjg123.module --output linked
```

It takes a couple of seconds to create the new binaries in `linked/bin`

```shell
$ linked/bin/java --list-modules
java.base@9.0.1
mjg123.module
```
We run the code just as before but using the new `java` binary:

```shell
$ linked/bin/java --module-path mods -m mjg123.module/mjg123.module.Main
Hello from mjg123.module
```

## Measurements

### Disk usage

My `$JAVA_HOME` is 557Mb. How about the new one?

```shell
$ du -sh linked
45M     linked
```

There is some more stuff we could trim away in there, but this is already a lot smaller. If you're distributing your code with the JVM bundled with it (eg in a container image) then the benefit is clear.

### Execution time

If you read my previous posts ([1](/2017/10/02/JVM-startup.html), [2](/2017/10/04/AppCDS-and-Clojure.html), [3](/2017/10/16/Clojure-1.9-startup.html)), you'll know I am mildly obsessed with startup time. Here's a couple of measurements:

Full JDK distribution:

```shell
$ perf stat -r50 java --module-path mods -m mjg123.module/mjg123.module.Main
...snip...
       0.204575188 seconds time elapsed                                          ( +-  1.39% )
```

As shrunk by `jlink`:

```shell
$ perf stat -r50 linked/bin/java --module-path mods -m mjg123.module/mjg123.module.Main
...snip...
       0.164922551 seconds time elapsed                                          ( +-  1.56% )
```

20% improvement with no extra effort. We can also use CDS trivially:

```shell
$ linked/bin/java -Xshare:dump
...ignore warnings about missing classes...

$ perf stat -r50 linked/bin/java -Xshare:on --module-path mods -m mjg123.module/mjg123.module.Main
...snip...
       0.128856211 seconds time elapsed                                          ( +-  1.46% )
```

BAM! Another 36ms gone!


## Conclusion

If you're able to use Java modules, `jlink` is an easy way to improve the size and speed of your Java apps. I would expect AOT and AppCDS to make it even faster.

The ever-excellent Trisha Gee has some advice about [migrating existing code into modules](https://www.infoq.com/articles/Java-Jigsaw-Migration-Guide).