---
title: "Diagnosing Java File I/O using JFR"
categories:
  - JVM
tags:
  - Java
  - JFR
  - Profiling
last_modified_at: 2022-02-08T19:20:55-05:00
---

The other day, my team at [PKWARE] received a report that the latest version of our library had triggered their I/O performance benchmarks. The new version of our Java library showed a spike in IO operations for a given task.

I knew that the JDK Flight Recorder could be used to diagnose performance issues, but had little experience with it before. I found limited information about diagnosing file I/O, so I thought I'd share what worked for us.

# JFR fundamentals
JFR uses "profiles" that configure what the recorder should monitor. Mission Control ([JMC]) calls these "Templates", but Intellij IDEA and the JFR option, `-XX:StartFlightRecording` call them "Settings". There are 2 settings that are commonly available: a "default" setting that adds about 1% overhead, and a "profile" setting that adds about 2% overhead.

We tried out both settings, but weren't getting file I/O events in the recording.

# Build your own settings
Fortunately, it's possible to build your own settings file. I think the easiest way to do this is using [JMC], but you could use a text editor - [the file is just XML](https://github.com/openjdk/jdk/blob/d658d945cf57bab8e61302841dcb56b36e48eff3/src/jdk.jfr/share/conf/jfr/default.jfc).

In JMC, open the Flight Recording Template Manager. Start by selecting a running JVM and continue as if you were capturing a flight recording for this VM.
![Start a flight recording](/images/jmc_start_flight_recording.png){: .align-left}

From the "Start Flight Recording" screen, open the Template Manager.
![Open the template manager](/images/jmc_open_template_manager.png){: .align-left}

I found it easiest to duplicate an existing template. I disabled most of the major events, but crucially, I set the "File I/O Threshold" to `0 ns`. This effectivly tells the flight recorder to log an event for every file I/O operation.
![Export JFR template from JMC](/images/configure_jfr_template.png){: .align-left}

Save your template. At this point, you can proceed to profile using JMC, or, as I did, you can export the template and use it from Intellij.

# Using the template in Intellij
Export your template to a file.
![Export JFR template from JMC](/images/jmc_export_jfr_template.png){: .align-left}

Then follow JetBrains' excellent [documentation](https://www.jetbrains.com/help/idea/java-flight-recorder.html) on how to use and configure JFR in Intellij.

# Diagnosis

![JFR results in Intellij](/images/jfr_in_intellij.png){: .align-left}
The profile showed many small disk reads: 512 bytes each. A bit of stack walking showed that Apache Commons Compress `ZipArchiveInputStream` [uses](https://github.com/apache/commons-compress/blob/39abfb17b02acd7d07b0c3ff5bac666a7bd35ea7/src/main/java/org/apache/commons/compress/archivers/zip/ZipArchiveInputStream.java#L99) a [512 byte](https://github.com/apache/commons-compress/blob/39abfb17b02acd7d07b0c3ff5bac666a7bd35ea7/src/main/java/org/apache/commons/compress/archivers/zip/ZipArchiveOutputStream.java#L89) buffer. It turned out that a `BufferedInputStream` had been missed.

# What's next?
We're investigating [JFRUnit] as a way of catching such problems sooner in the future. 

## BTW
[We're hiring](https://www.linkedin.com/jobs/search/?currentJobId=2848670782&f_C=23724)! Join us in building the libraries that power PKWARE's data detection and security!

[PKWARE]: https://www.pkware.com/careers
[JMC]: https://adoptium.net/jmc.html
[JFRUnit]: https://github.com/moditect/jfrunit