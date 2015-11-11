---
layout: post
title:  "Upgrade jdk7 in debian (jessie)"
date:   2015-11-11 18:09:30
categories: blog
tags: debian jdk7
---
Upgrading jre/jdk on debian is usually easy, but sometimes when certain repos are inaccessible, apt-get command fails to
automatically install jdk version for you. This post guides you to upgrade your jdk version in such a case.

I was able to upgrade my jdk7 version to 7u85-2.6.1-6+deb8u1_amd64 using the following commands. 

First you need to download three basic packages, which we will install in this order:

~~~
openjdk-7-jre-headless_7u85-2.6.1-6+deb8u1_amd64.deb
openjdk-7-jre_7u85-2.6.1-6+deb8u1_amd64.deb
openjdk-7-jdk_7u85-2.6.1-6+deb8u1_amd64.deb
~~~

You can download these from official debian packages website, then drill through by selecting appropriate debian
versions, and the architecture of the system etc, for eg: `https://packages.debian.org/jessie/amd64/openjdk-7-jre/download`

Install the above packages using:

~~~
sudo gdebi openjdk-7-jre-headless_7u85-2.6.1-6+deb8u1_amd64.deb
sudo gdebi openjdk-7-jre_7u85-2.6.1-6+deb8u1_amd64.deb
sudo gdebi openjdk-7-jdk_7u85-2.6.1-6+deb8u1_amd64.deb
~~~

gdebi will automatically resolve all the other dependencies needed for these packages.

Checking jdk versions
===

You can check the different java versions installed on debian using:

~~~
sudo update-alternatives --config java

There are 2 choices for the alternative java (providing /usr/bin/java).

  Selection    Path                                            Priority   Status
  ------------------------------------------------------------
*0            /usr/lib/jvm/java-8-oracle/jre/bin/java          1072      auto mode
1            /usr/lib/jvm/java-7-openjdk-amd64/jre/bin/java   1071      manual mode
2            /usr/lib/jvm/java-8-oracle/jre/bin/java          1072      manual mode

Press enter to keep the current choice[*], or type selection number:1 

~~~

As you can see now I have 2 different java versions, with java-8-oracle being the currently active one. You can change
the default java version by simply inputing the selection number here. Note that you can still override java version by
updating JAVA_HOME environment variable.

~~~
java -version
java version "1.7.0_85"
OpenJDK Runtime Environment (IcedTea 2.6.1) (7u85-2.6.1-6+deb8u1)
OpenJDK 64-Bit Server VM (build 24.85-b03, mixed mode)
~~~

References
==

- Update [default java][default] in debian
- gdebi [explained][gdebi]


[default]:       http://www.mkyong.com/linux/debian-change-default-java-version/
[gdebi]:         http://linuxcommando.blogspot.in/2015/01/using-gdebi-to-install-and-resolve.html
