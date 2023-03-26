---
title: "Diagnosing More Java Disk I/O using JFR"
categories:
  - JVM
tags:
  - Java
  - JFR
  - Profiling
  - JMC Agent
last_modified_at: 2023-03-26T12:15:00-05:00
---

The JDK provides JFR events for file reads and writes. These report the duration of the action and the number of bytes read/written. However, there are many more disk IO operations that are not instrumented, but can nevertheless have a significant impact on the performance characteristics of production applications. Examples include checking whether a file exists, looking up a modified timestamp, and creating a directory. This post explores instrumenting these actions.

{% include toc %}

# The Basics
The general approach to instrumenting IO operations is to wrap the real implementation with the instrumentation. The JDK does this through bytecode manipulation, rewritting the bytecode for framework classes shortly after they are loaded. Bytecode manipulation is also used by the [JMC Agent] `javaagent`. An alternative, but easier to understand approach is to write a plain Java wrapper implementation that delegates to the real implementation for actual IO. Fortunately, this isn't as hard as it may seem.

## FileSystemProvider
The NIO [`FileSystemProvider`] SPI was introducedIn Java 7 as a more flexible approach to writing File IO code. It is possible to write a custom implementation that delegates actual IO operations to the JDK's `FileSystemProvider`, but wraps them with instrumentation logic. By setting the `java.nio.file.spi.DefaultFileSystemProvider` property prior to JVM startup, we can ensure that all `FileSystem`s throughout the JDK use our custom implementation.

However, there are a few downsides to this. Primarily, this approach doesn't help with instrumenting `java.io.File`. There are still a lot of applications, frameworks, and libraries that make use of this API, and ignoring it in our instrumentation is problematic.

# JMC Agent
The JMC Agent is a tool provided by the JMC project. The agent [is configured](https://developers.redhat.com/blog/2020/10/29/collect-jdk-flight-recorder-events-at-runtime-with-jmc-agent#introduction_to_the_jmc_agent_plugin) using an XML file that contains classes and methods to instrument, and reports instrumented events via JFR. The targeted classes have their bytecode modified at runtime and are then reloaded.

## Instrumenting JDK classes
The JMC Agent is typically used to instrument either library classes or a running application that can't be redeployed. For this use case, we're instrumenting JDK code, which we'd normally consider library code. However, things are a bit more complicated.

### The bootclasspath
The JVM loads classes using `ClassLoader`s. `ClassLoaders` are layered - child `ClassLoaders` have access to all the classes in their parent `ClassLoader`. For a simplified discussion, assume that when the JVM starts, there are two `ClassLoaders`: the system `ClassLoader`, and it's parent, the boot `ClassLoader`. These load classes from the `classpath` and `bootclasspath` respectively.

The `bootclasspath` contains classes essential for booting the JVM. Think `String`,  file IO to read classes from disk, ZIP classes for JAR loading/extraction, etc. The `classpath` contains everything else: the application, libraries, instrumentation agents, etc. Applications typically interact only with the system `ClassLoader` and have access to the classes on the `bootclasspath` through the parent relationship. However, we're interested in instrumenting classes on the `bootclasspath`, so things are a bit more interesting.

When the JMC Agent instruments a class, it makes use of classes that are part of the JMC Agent JAR, which is part of the `classpath`. However, the classes we're interested in instrumenting are part of the `bootclasspath`, which doesn't have access to the `classpath` entries. Fortunately, it's possible to add JARs to the `bootclasspath`. We modify our `java` command to be `java -Xbootclasspath/a:/path/to/agent.jar`.

### Java modules
Java 9 introduced JPMS. The classes we're interested in instrumenting live in the `java.base` module. JFR events live in the `jdk.jfr` module. Notice that the [module graph](https://docs.oracle.com/en/java/javase/20/docs/api/java.base/module-summary.html) for `java.base` does not include `jdk.jfr`. The JMC Agent generates `jdk.jfr.Event` subclasses in the same module as classes being instrumented. This results in a runtime error: `java.lang.IllegalAccessError: superclass access check failed: class java.nio.file.fileioExists (in module java.base) cannot access class jdk.jfr.Event (in module jdk.jfr) because module java.base does not read module jdk.jfr`.

To work around this, we have to tell the JVM that the `java.base` module may in fact read the `jdk.jfr` module. We do this by modifying the `java` command to be `java --add-reads java.base=jdk.jfr`.

### Agent config
We have to create an agent config file: `jmcAgent.xml`. The config below instruments the check performed when `java.nio.Files.exists(Path)` is invoked on unix file systems.
```xml
<jfragent>
  <config>
    <!-- Prefix applied to generated code. We never interact with this; it's just for uniqueness. -->
    <classprefix>fileio</classprefix>
    <!-- Allow `toString() to be called for types not supported by JFR. -->
    <allowtostring>true</allowtostring>
    <!-- We're not including custom converters, but you could and would set this to true -->
    <allowconverter>false</allowconverter>
  </config>
  <events>
    <event id="FileIO.Exists">
      <label>Exists</label>
      <path>FileIO</path>
      <description>Checks if a file exists</description>
      <class>sun.nio.fs.UnixFileSystemProvider</class>
      <!-- Stack trace so we can figure out where the call is coming from -->
      <stacktrace>true</stacktrace>
      <rethrow>false</rethrow>
      <location>WRAP</location>
      <method>
        <name>exists</name>
        <descriptor>(Ljava/nio/file/Path;)Z</descriptor>
        <parameters>
          <parameter index="0">
            <name>Path</name>
            <description>Path to file</description>
            <contenttype>None</contenttype>
          </parameter>
        </parameters>
        <!-- Optional -->
        <returnvalue>
          <name>Exists</name>
          <contenttype>None</contenttype>
        </returnvalue>
      </method>
    </event>
  </events>
</jfragent>
```

Something to note here is that we have to look at JDK internal classes; namely, `sun.nio.fs.UnixFileSystemProvider`. This this unfortunate, as these classes aren't meant to be used directly by application developers and are liable to change from one release to the next.

### Putting it all together
We'll need a JFR configurtion file: `example.jfc`. I'm copying the default configuration from the JVM (`$JAVA_HOME/lib/jfr/default.jfc`), and adding the following event to the `Java Application` category. Note that we set the threshold to `0 ns`, which causes every event to be stored. Adjust this value as needed for your needs.
```xml
<event name="fileio.Exists">
  <setting name="enabled" contentType="jdk.jfr.Flag">true</setting>
  <setting name="threshold" label="Threshold" description="Record event with duration above or equal to threshold" contentType="jdk.jfr.Timespan">0 ns</setting>
</event>
```

And our sample application, in `App.java`
```java
import java.nio.file.Files;
import java.nio.file.Path;

public final class App {
  public static void main(String[] args) {
    Files.exits(Path.of("example.jfc"));
  }
}
```

This gives us a final command of
`java -Xbootclasspath/a:../agent.jar --add-reads java.base=jdk.jfr -XX:StartFlightRecording=disk=true,dumponexit=true,filename=example.jfr,settings=example.jfc -javaagent:../agent.jar=jmcagent.xml App`.

And in fact, we get our event!
![Custom file exists event](/images/diagnosing_disk_io_result.png){: .align-left}

# Conclusion
We see that it is possible, albeit cumbersome, to instrument JDK disk IO classes beyond what is provided out of the box as of JDK 20 through the use of JMC Agent. While the setup is time intensive, this could realistically be published as a pre-made configuration for easy re-use across projects.

## Future work
### More detailed events
One thing to note about the resulting event, is that that `Path` field of the event is relative. That may be OK, but if an absolute path is desired, one could create a custom Converter. Custom Converters are poorly documented, but take a look at [`FileConverter`](https://github.com/openjdk/jmc/blob/master/agent/src/main/java/org/openjdk/jmc/agent/converters/FileConverter.java) and it's usages to get an  idea. This could also work for cases where you want to report the number of bytes read by a call to load attributes, or similar statistics.

### First-class support in JDK
I've reached out to the JDK JFR mailing list [to propose](https://mail.openjdk.org/pipermail/hotspot-jfr-dev/2023-March/004816.html) expanding the number of events available in the JDK. Perhaps we'll see this functionality added, and application developers will get richer insights into what their applications are doing. If you're interested in this, or have a use case you'd like to share, subscribe to the [mailing list](https://mail.openjdk.org/mailman/listinfo/hotspot-jfr-dev) and send an email with your thoughts!