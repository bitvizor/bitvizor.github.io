---
layout: post
title:  "Kafka Series: Getting started with Apache Kafka Development - Part 1"
date:   2021-07-13 12:37:00 +0500
categories: java apache kafka ESB
---

# Introduction

* Part 1 (Setting up the environment)
* Part 2 (Develope a StreamProcessor application)


Before going into detail, let's present our case study. We need to model the systems of bitVizor, a fictional company whose core business is real-estate exchange. bitVizor wants to base its IT infrastructure on an enterprise service bus (ESB) built with Apache Kafka. The bitVizor IT department wants to unify the service backbone across the organization. bitVizor also has worldwide, web-based, and mobile-app-based clients, so a real-time response is fundamental.

Online customers worldwide browse the bitVizor website to browse, search, buy and sell properties. There are a lot of use cases that customers can perform in bitVizor, but this example is focused on the part of the user tracking workflow specifically from the web application.

bitVizor's marketing department wants to feed realtime user tracking data to their analytics engine, but before that they need to enrich the data with some more valuable information using information from third party API's and data sources


## Setting up the project

We are going to build our project using Gradle, the first step is to download and install the latest gradle 


```
cd ~/bin
wget https://downloads.gradle-dn.com/distributions/gradle-7.1-bin.zip
unzip gradle-7.1-bin.zip
ln -s ~/bin/gradle-7.1/bin/gradle ~/bin/gradle
rm -f ~/bin/gradle-7.1-bin.zip
```
To check that `gradle` is installed successfully, issue the following command

```bash
gradle -v
```

The output should be something like this

```
------------------------------------------------------------
Gradle 7.1
------------------------------------------------------------
```
The first step is to create the project directory `eventprocessor`

```bash
mkdir ~/eventprocessor
```

And execute following from that directory

```bash
gradle init --type java-library
```

The output should be something like this 

```log
> Task :init
Get more help with your project: https://docs.gradle.org/7.1/samples/sample_building_java_libraries.html

BUILD SUCCESSFUL in 11s
2 actionable tasks: 2 executed

```

Gradle generates a skeleton project inside the directory. The directory should be similar to the following

```
.
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── lib
│   ├── build
│   ├── build.gradle
│   └── src
│       ├── main
│       │   ├── java
│       │   │   └── eventprocessor
│       │   │       └── Library.java
│       │   └── resources
│       └── test
│           ├── java
│           │   └── eventprocessor
│           │       └── LibraryTest.java
│           └── resources
└── settings.gradle

```


> You may delete Library.java and LibraryTest.java 

Now let's modify the `build.gradle` and add dependencies for our project

```
    implementation 'org.apache.kafka:kafka-clients:2.8.0'
    implementation 'org.apache.kafka:kafka-streams:2.8.0'
    implementation 'com.maxmind.geoip2:geoip2:2.15.0'
    implementation 'com.fasterxml.jackson.core:jackson-core:2.12.3'
    implementation 'com.github.javafaker:javafaker:1.0.2'
```

To download the dependencies and compile the sources, run following command

```bash
gradle compileJava
```

If everything goes as expected, the output should be like this

```
BUILD SUCCESSFUL in 1s
1 actionable task: 1 executed
```

Project can also be created using Maven or an IDE like IntelliJ, Eclipse etc;
we are using `gradle` for simplicity 

## Overview
Now that our project's skeleton is ready, let's recall the project requirements for the stream processing engine. The first step in our pipeline is to enrich the stream with more valueable information.

In this context, we understand enrichment as adding extra data that was not in the original message. In this section, we will see how to enrich a message with geographic location using the MaxMind database. The events that we are going to model for bitVizor, each one includes the IP address of the customer's computer.

Our system in bitVizor searches for the IP address of our customer in the MaxMind database to determine where the customer is located when the request to our system was made. The use of data from external sources to add them to our events is what we call message enrichment.

Please stay tuned for the next post in this series, we will implement our StreamProcessor in next post 

