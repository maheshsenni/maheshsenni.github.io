---
title: "Java Performance Profiling Using Flame Graphs"
date: 2020-12-31T11:01:02-06:00
tags: ["java", "performance", "visualization", "flame-graph", "how-to"]
---

One of the great advantages of microservices is that, when there is an issue, you already have a pretty good idea of where it is happening and which microservice is responsible for it. And if it is a performance issue, you have a manageable amount of code or libraries to investigate, rather than dealing with the monolith as a whole.

There are a lot of performance measurement tools that come as part of JDK itself — [JConsole](http://openjdk.java.net/tools/svc/jconsole/), [VisualVM](https://visualvm.github.io/), [HPROF](https://docs.oracle.com/javase/7/docs/technotes/samples/hprof.html), etc. Most of them profile the application as a whole and it would take some effort to get to class or method level hot spots. While I was trying to evaluate the performance of one of our microservices, I came across a method using flame graphs which I found very effective in finding out CPU usage of the code. This post is more of a how-to and all credits go to [Brendan Gregg](http://www.brendangregg.com/blog/2014-06-12/java-flame-graphs.html).

## Requirements

- A Linux machine with [perf](https://perf.wiki.kernel.org/index.php/Main_Page)
- JDK — JDK8u60 and above
- [FlameGraph visualizer](https://github.com/brendangregg/FlameGraph)
- [jvm-profiling-tools/perf-map-agent](https://github.com/jvm-profiling-tools/perf-map-agent)
- An application to profile :)

I used an EC2 machine running RHEL 7 for this exercise — although I never tried, I expect Vagrant or VirtualBox should also work. If the application is an API, you need a load testing tool like JMeter or wrk to generate traffic for the API.

## Steps

At a very high level, this is what needs to be done.

- Install `perf_events`
- Build `perf-map-agent` from source
- Run the Java application in the machine with `-XX:+PreserveFramePointer` JVM option
- Generate load for the application using a load testing tool
- Run `perf-record` command to capture performance counter profile
- Run `perf-map-agent` to generate a map for JIT-compiled methods
- Generate stack trace output from the previously recorded data by running `perf-script`
- Generate flame graph

I am using a RHEL machine, the commands in this post are based on it, but it should be easy to find equivalent commands for your OS. Let’s look at each of these steps in detail now.

### Install perf_events

As the flame graphs are generated from the output of Linux perf_events, the first steps is to install it which provides the perf CLI command. Command to install perf_events:

```bash 
yum install perf
```

### Build perf-map-agent

When an application is running, JVM performs just-in-time (JIT) compilation of the byte code at runtime to optimize frequently used “hot” code. The byte code is converted to native code to improve performance and this native code is stored in memory. When perf runs, only this memory address is accessible and not the actual Java class or method. A tool like perf-map-agent connects to a running a JVM process and exports a map file which can be used by perf to generate the stack trace with the actual Java method names.

To build `perf-map-agent` follow the instructions in the source repo. It should be something like this:

```bash
git clone https://github.com/jvm-profiling-tools/perf-map-agent.git
export JAVA_HOME=<JDK_DIR>
cd perf-map-agent
yum install cmake
cmake .
make
```

### Run the application

The next step is to run the application with the JVM option `-XX:+PreserveFramePointer`. Frame pointers are commonly used to provide information to the debuggers about the call stack. With this option set, perf can construct more accurate stack traces by using information in the frame pointer about the currently executing method. Using this feature requires, JDK8u60 and above.

```bash
java -XX:+PreserveFramePointer -jar app.jar
```

Keep the application running to until performance profile (perf record) and symbol table (`perf-map-agent`) are captured.

### Generate load

Generate load for your application using any of the load testing tools or a different approach depending on the application.

### Capture performance profile

When the application is running, start capturing the CPU profile using perf_events with the following command:

```bash
perf record -F 99 -p `pgrep java` -g -- sleep 10
```

- -F 99 — Run profile at this frequency
- -p — Profile an existing process with this PID
- -g — Generate call graph
- sleep 10 — Profile for ten seconds

Once the profiling is completed after ten seconds, this command will generate a file called `perf.data`.

### Export symbols

Assuming you have already built perf-map-agent, run the following command while the application is running to generate a map file of JVM symbols:

```bash
bin/create-java-perf-map.sh `pgrep java`
```

This command will create the file `/tmp/<PID>.map`. The application can be stopped at this point and is not needed for the subsequent steps.

### Generate trace output

Now that we have the profile data and the symbols map, we can generate a details trace output of the profiled information. Run this command in the same directory as the perf.data file generated earlier:

```bash
perf script > out.perf
```

This command will look for the map file in `\tmp` and use it to generate the output. It will fail if the `.map` file is not present in `/tmp`.

### Flame graph 🔥

Get the scripts to generate the flame graph from the [source repo](https://github.com/brendangregg/FlameGraph). Run the scripts by passing the trace output generated earlier.

```bash
git clone --depth 1 https://github.com/brendangregg/FlameGraph.git
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > graph.svg
```

`graph.svg` is the flame graph and it can be opened in your favorite browser to explore.

## Conclusion

Check out the reference articles and video linked below to get more information about flame graphs. Hope you found this useful.

### References
[http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html](http://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html)  
[https://medium.com/netflix-techblog/java-in-flames-e763b3d32166](https://medium.com/netflix-techblog/java-in-flames-e763b3d32166)

{{< youtube nZfNehCzGdw >}}

> This post was originally [published in Medium](https://medium.com/@maheshsenni/java-performance-profiling-using-flame-graphs-e29238130375) on Apr 15, 2019.
