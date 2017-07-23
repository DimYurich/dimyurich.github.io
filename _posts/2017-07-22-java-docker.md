---
layout: post
title: Notes on running JVM-based apps in Docker containers
tags: [jvm, java, groovy, docker]
---

Docker has taken Ops world by storm, and JVM-based applications and services aren't an exception: they are also being containerized. However, as usual, devil is in the details, and there are a number of subtleties one needs to be aware of when running a JVM inside Docker. In this blog post I'll try to summarize some things that were already mentioned in different sources, and also will provide an example of a decent (a.k.a. good enough) Dockerfile that can be used as a reference.

<!--more-->

First of all, I strongly urge you to read the following (TL;DR -- summary below):
* A blog post from Fabio Kung: ["Memory inside Linux containers"](https://fabiokung.com/2014/03/13/memory-inside-linux-containers/)
* An article from Redhat: ["Java inside docker: What you must know to not FAIL"](https://developers.redhat.com/blog/2017/03/14/java-inside-docker/)
* A blog post from Matt Williams: ["Docker, Cgroups, Memory Constraints, and Java: A Cautionary Tale, or Here be Reapers (sometimes)"](http://matthewkwilliams.com/index.php/2016/03/17/docker-cgroups-memory-constraints-and-java-cautionary-tale/)
* An article from Takipi: ["Running Java on Docker? You're Breaking the Law"](http://blog.takipi.com/running-java-on-docker-youre-breaking-the-law/)
* An article from Oracle: ["Official Docker Image for Oracle Java and the OpenJDK Roadmap for Containers"](https://blogs.oracle.com/developers/official-docker-image-for-oracle-java-and-the-openjdk-roadmap-for-containers)
* One more article from Oracle: ["Java SE support for Docker CPU and memory limits"](https://blogs.oracle.com/java-platform-group/java-se-support-for-docker-cpu-and-memory-limits)

Here are some key takeaways that you can and should consider:
1. Do you even need a JDK on actual servers or in the cloud? Maybe you'd be just fine running your services using JRE? Most likely, the answer is yes. In this case, you can use [official HotSpot image from Oracle](https://store.docker.com/images/oracle-serverjre-8).
2. If you're sure you need an actual JDK, and if you don't have commercial support from any JVM vendor, use [official OpenJDK Docker images](https://hub.docker.com/_/openjdk/) as the base. It's built off Debian linux, and has an Alpine linux flavor, which is almost as small as it can get (more improvements potentially coming with JDK 9 modularity). Save yourself some time and effort, make use of community-maintained stuff. Moreover, since it's an OpenJDK build and distribution, you'll have one less reason to worry about from legal perspective. You can also use a JRE image from this family, if you prefer that.
3. Be aware of [JVM Ergonomics](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/ergonomics.html). You can (and probably should!) configure your memory limits manually, by using `-Xmx` in `JAVA_OPTIONS`, just make sure that you leave some space in container's RAM for JVM's own needs. There are several blog posts recommending other options for your production, with some explanations and, certainly, room for thought:
  * [Java VM Options You Should Always Use in Production](http://blog.sokolenko.me/2014/11/javavm-options-production.html)
  * [Useful JVM Flags – Part 1 (JVM Types and Compiler Modes)](https://blog.codecentric.de/en/2012/07/useful-jvm-flags-part-1-jvm-types-and-compiler-modes/)
4. Use JDK / JRE 8u131 (it is already adopted in official Docker images, BTW), and enable experimental support of cgroups memory limits, which just has been [introduced in that release](https://bugs.openjdk.java.net/browse/JDK-8170888). It still has [some bugs](https://bugs.openjdk.java.net/browse/JDK-8146115), but overall I haven't seen or heard it taking any major heat. Besides memory limits, [CPU limits are also supported since 8u131](https://bugs.openjdk.java.net/browse/JDK-6515172), which is a very nice thing if you don't want to over-commit your cloud VMs (as if they aren't already over-committed by the hypervisor, hehe).

Now let's have a look at that Dockerfile. It comes from this geo-location pet project I'm working on and which you'll be seeing a lot in future posts. Here it is:

```dockerfile
# (1) Don't need JDK
# (2) prefer Alpine linux
# (4) using docker-aware build
FROM openjdk:8u131-jre-alpine

ADD build/libs/geoip-0.0.1-SNAPSHOT.jar geoip.jar
# (3) setting VM options
# IMPORTANT! Run this with *at least* 2 GB of RAM allocated in Docker
# Yes, I know 1g is an overkill for this, but c'mon
ENV JAVA_OPTS="-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Xmx1g -XX:+PrintFlagsFinal \
               -XX:ErrorFile=fatal.log -XX:+UseConcMarkSweepGC -XX:+CMSParallelRemarkEnabled \
               -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=10 \
               -XX:+ScavengeBeforeFullGC -XX:+CMSScavengeBeforeRemark \
               -XX:+PrintGCDateStamps -verbose:gc -XX:+PrintGCDetails -Xloggc:gc-log.log \
               -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=100M \
               -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=`date`.hprof"
ENTRYPOINT [ "sh", "-c", "java $JAVA_OPTS -jar /geoip.jar" ]
```

Please let me know if you've found something that's not obvious or might benefit from better / deeper explanation or is contradictory in comments below, and hopefully see you soon! :)
