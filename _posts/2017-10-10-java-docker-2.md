---
layout: post
title: Running JVM-based apps in Docker containers, a note on storage driver
tags: [jvm, java, groovy, docker]
---

After wrapping up my [previous post](http://dimyurich.github.io/2017/07/22/java-docker/) I started looking into how does Elastic the company run ElasticSearch the search server in Docker, and something interesting showed up.

**TL:DR**
When you're using Alpine Linux to run a JVM-based app, and said app is accessing docker image's file system for practically anything (from files of data to `.properties` files), make sure that you're using Docker's aufs storage driver, or, really, anything but the overlay or overlay2 or btrfs drivers. Keep that in mind when you'll be reading [Select a driver user guide  page](https://docs.docker.com/engine/userguide/storagedriver/selectadriver/).

## What's the problem?
<!--more-->
As you can see, [ElasticSearch Docker image ships being based on CentOS 7](https://www.elastic.co/blog/docker-base-centos7). They claim in that post that [this bug in JDK](https://bugs.openjdk.java.net/browse/JDK-8165852) is to blame, even though it's not Alpine-specific. On the other hand, [aufs has attracted some criticism](https://sthbrx.github.io/blog/2015/10/30/docker-just-stop-using-aufs/).

## What if I really want Alpine + overlay2?
1. You can try squashing your images. You can do that using `docker build --squash` [documented here](https://docs.docker.com/engine/reference/commandline/build/#squash-an-images-layers-squash-experimental-only), or by using something like [docker-squash](https://github.com/jwilder/docker-squash). However, squashing really defeats the purpose of Docker itself to share (and reuse) base images. So, if you build 10 different images `FROM openjdk:8u131-jre-alpine` and then squash them, you'll end up loading that base image 10 times (as part of your 10 different images) into your RAM instead of 1. This wastes RAM and makes images loading slower, so it's probably something that you might exactly NOT want to do.
2. Another option is to use devcemapper storage driver, since it seems to work fine according to [this comment](https://github.com/elastic/elasticsearch-docker/issues/44#issuecomment-290935503). I haven't checked it personally, though, since [the effort to get it up and running](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/) seems to be too complicated for my taste and what I'm trying to do.

## What are these Docker's storage drivers, anyway?
Here's a great presentation that's very well worth looking into for some initial dive into the topic: [link](https://jpetazzo.github.io/assets/2015-03-03-not-so-deep-dive-into-docker-storage-drivers.html)

## Are there any other resources to check out on this topic?
There are some, but not many:
* this video from [JEEConf](http://jeeconf.com/) gives a good overview of some aspects about running Java in Docker: [Andrey Adamovich on Dockerized Java (En)](https://www.youtube.com/watch?v=NQ5hTEp-GTM)
* please have a look at [OpenJDK bug tracker, look for label "docker"](https://bugs.openjdk.java.net/browse/JDK-8165852?jql=labels%20%3D%20docker)
* this [super-insightful rant](https://thehftguy.com/2016/11/01/docker-in-production-an-history-of-failure/) shares many more aspects on storage drivers as well as caveats of running Docker in production
* more info on storage drivers [here](https://integratedcode.us/2016/08/30/storage-drivers-in-docker-a-deep-dive/)
* a little dated, but still actual [post from RedHat](https://developers.redhat.com/blog/2014/09/30/overview-storage-scalability-docker/)
