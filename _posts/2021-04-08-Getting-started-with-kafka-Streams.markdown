---
layout: post
title:  "Kafka Series: Getting started with Apache Kafka Development - Part 1"
date:   2021-07-13 12:37:00 +0500
categories: java apache kafka ESB
---

# Introduction

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


## Prepare the topics and the input data

## The Constants
The first step is to code our Constants class. This class is a static class with all of the Constants needed in our project.

Open the project with your favorite IDE and, under the `src/main/java/streamprocessor`
directory, create a file called `Constants.java` with the following contents

```java
package streamprocessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.SerializationFeature;
import com.fasterxml.jackson.databind.util.StdDateFormat;

public final class Constants {
    private static final ObjectMapper jsonMapper;
    static {
        ObjectMapper mapper = new ObjectMapper();
        mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
        mapper.setDateFormat(new StdDateFormat());
        jsonMapper = mapper;
    }
    public static ObjectMapper getJsonMapper() {
        return jsonMapper;
    }
}   
```

## The Message class
The second step is to code the Message class. This class is a Plain Old Java Object
(POJO). The model class is the template for the value object.

Open the project with your favorite IDE and, in the `src/main/java/streamprocessor` directory, create a file called `Message.java` with the following contents

```java
package streamprocessor;

import java.io.Serializable;

public class Message implements Serializable {

    private Long to = null;
    private String telco = null;
    private String service = null;
    private String message = null;

    public String getService() {
        return service;
    }

    public void setService(String service) {
        this.service = service;
    }

    public String getTelco() {
        return telco;
    }

    public void setTelco(String telco) {
        this.telco = telco;
    }

    public Long getTo() {
        return to;
    }

    public void setTo(Long to) {
        this.to = to;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    @Override
    public String toString() {
        return "Message{" +
                "to=" + to +
                ", message='" + message + '\'' +
                '}';
    }
}

```
## Create the Serializers

To build a custom serializer, we need to create a class that implements the `org.apache.kafka.common.serialization.Serializer` interface. 

In the `src/main/java/streamprocessor/serializers/` directory, create a file called `JsonSerializer.java` with the following contents 

```java
package streamprocessor.serializers;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Serializer;
import java.util.Map;

/**
 * Json Serializer
 */
public class JsonSerializer<T> implements Serializer<T> {
    private final ObjectMapper objectMapper = new ObjectMapper();
    public JsonSerializer() {}

    @Override
    public void configure(Map<String, ?> config, boolean isKey) {}
    /**
     * Serialize JsonNode
     *
     * @param topic Kafka topic name
     * @param data  data as JsonNode
     * @return byte[]
     */
    @Override
    public byte[] serialize(String topic, T data) {
        if (data == null) {
            return null;
        }
        try {
            return objectMapper.writeValueAsBytes(data);
        } catch (Exception e) {
            throw new SerializationException("Error serializing JSON message", e);
        }
    }
    @Override
    public void close() {}
}
```
In a similar way, to build a custom deserializer, we need to create a class that implements
the `org.apache.kafka.common.serialization.Deserializer` interface.

In the `src/main/java/streamprocessor/serializers/` directory, create a file called `JsonDeserializer.java` with the following contents 

```java
package streamprocessor.serializers;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Deserializer;
import java.util.Map;

/**
 * Json Deserializer
 *
 * @author Ghulam Mustafa
 * @author www.bitvizor.com
 */
public class JsonDeserializer<T> implements Deserializer<T> {
    private ObjectMapper objectMapper = new ObjectMapper();
    private Class<T> className;
    public static final String KEY_CLASS_NAME_CONFIG = "key.class.name";
    public static final String VALUE_CLASS_NAME_CONFIG = "value.class.name";
    public JsonDeserializer() {}
    /**
     * Set the specific Java Object Class Name
     *
     * @param props set specific.class.name to your specific Java Class Name
     * @param isKey set it to false
     */
    @SuppressWarnings("unchecked")
    @Override
    public void configure(Map<String, ?> props, boolean isKey) {
        if (isKey)
            className = (Class<T>) props.get(KEY_CLASS_NAME_CONFIG);
        else
            className = (Class<T>) props.get(VALUE_CLASS_NAME_CONFIG);
    }
    /**
     * Deserialize to a POJO
     *
     * @param topic topic name
     * @param data  message bytes
     * @return Specific Java Object
     */
    @Override
    public T deserialize(String topic, byte[] data) {
        if (data == null) {
            return null;
        }
        try {
            return objectMapper.readValue(data, className);
        } catch (Exception e) {
            throw new SerializationException(e);
        }
    }
    @Override
    public void close() {
        //nothing to close
    }
}
```

We will now construct a unified JSON serde from the JsonSerializer and JsonDeserializer via `Serdes.serdeFrom(<serializerInstance>, <deserializerInstance>)`. 

In the `src/main/java/streamprocessor/serdes/` directory, create a file called `AppSerdes.java` with the following contents 

```java
package streamprocessor.serdes;
import org.apache.kafka.common.serialization.Serde;
import org.apache.kafka.common.serialization.Serdes;
import streamprocessor.VehicleLocation;
import streamprocessor.aggregator.BillingCount;
import streamprocessor.serializers.JsonDeserializer;
import streamprocessor.serializers.JsonSerializer;
import java.util.HashMap;
import java.util.Map;


public class AppSerdes {

    static final class VehicleLocationSerde extends Serdes.WrapperSerde<VehicleLocation> {
        VehicleLocationSerde() {
            super(new JsonSerializer<>(), new JsonDeserializer<>());
        }
    }

    public static Serde<VehicleLocation> Message() {
        VehicleLocationSerde serde = new MessageSerde();

        Map<String, Object> serdeConfigs = new HashMap<>();
        serdeConfigs.put(JsonDeserializer.VALUE_CLASS_NAME_CONFIG, Message.class);
        serde.configure(serdeConfigs, false);

        return serde;
    }
}

```

## Extracting the geographic location

In our case we will need to extract IP address from input payload and add a new field containing city and country name, A simple service to query Maxmind db 

Open the build.gradle and make sure we have maxmind.geoip dependency already in place.

```
    implementation 'com.maxmind.geoip2:geoip2:2.15.0'
```

Download a copy of the MaxMind GeoIP free database, extract it and move the `GeoLite2-City.mmdb` file in a route accessible to our program. Now, add a file called `GeoIPService.java` in the `src/main/java/streamprocessor/extractors/` directory  containing the following contents 


```java
package streamprocessor.extractors;
import com.maxmind.geoip2.DatabaseReader;
import com.maxmind.geoip2.exception.GeoIp2Exception;
import com.maxmind.geoip2.model.CityResponse;

import java.io.File;
import java.io.IOException;
import java.net.InetAddress;
import java.util.logging.Level;
import java.util.logging.Logger;


public final class GeoIPService {
  private static final String MAXMINDDB = "/home/mustafa/GeoLite2-City.mmdb";
  File database = new File(MAXMINDDB);
  DatabaseReader reader = new DatabaseReader.Builder(database).build();

  public GeoIPService() throws IOException {
  }

  public CityResponse getLocation(String ipAddress) {
    try {
      InetAddress ipAddress1 = InetAddress.getByName(ipAddress);
      return reader.city(ipAddress1);

    } catch (IOException | GeoIp2Exception ex) {
      Logger.getLogger(GeoIPService.class.getName()).log(Level.INFO, ex.getMessage());
    }
    return null;
  }
}
```

## Stream Processor

Now, in the `src/main/java/streamprocessor` directory, create a file called
PlainStreamsProcessor.java with the following content
```java
   ...

   public final void process() {

        Map<String, Object> serdeProps = new HashMap<>();

        // Stream DSL
        StreamsBuilder streamsBuilder = new StreamsBuilder();
        KStream<Integer, VehicleLocation> initialStream = streamsBuilder.stream(Constants.getMessageTopic(), Consumed.with(Serdes.Integer(), AppSerdes.Message()));
        Faker faker = new Faker();

        initialStream
                .mapValues(v -> {
                    String country = null;
                    try {
                        CityResponse cityResponse = new GeoIPService().getLocation(v.getIpAddress());
                        country = cityResponse.getCountry().getName();
                    } catch (IOException e) {
                        e.printStackTrace();
                    } catch (NullPointerException e) {
                        country = "N/a";
                    }
                    v.setCountry(country);
                    return v;
                })
                .selectKey((k, v) -> v.getVehicleId())
                .peek((k, v) -> System.out.println(k +" - " +v.getCountry()))
                .to("enriched_gpslocation", Produced.with(Serdes.Integer(), AppSerdes.Message()));

        // Build stream topology and start the stream processing engine!
        Topology topology = streamsBuilder.build();

        Properties props = new Properties();
        props.put("bootstrap.servers", this.brokers);
        props.put("application.id", "streamprocessor");
        KafkaStreams streams = new KafkaStreams(topology, props);
        streams.setUncaughtExceptionHandler((exception -> StreamsUncaughtExceptionHandler.StreamThreadExceptionResponse.REPLACE_THREAD));

        System.out.println(topology.describe());

        streams.start();
    }

    ...
```


All the magic happens inside the `process()` method. The first step in a Kafka Streams application is to get a `StreamsBuilder` instance, as shown in the following code:
```java
StreamsBuilder streamsBuilder = new StreamsBuilder();
```
The `StreamsBuilder` is an object that allows building a topology. A topology in Kafka Streams is a structural description of a data pipeline. The topology is a succession of steps that involve transformations between streams. A topology is a very important concept in streams; it is also used in other technologies such as Apache Storm.

The StreamsBuilder is used to consume data from a topic. There are other two important concepts in the context of Kafka Streams: a `KStream`, a representation of a stream of records, and a `KTable` , a log of the changes in a stream. To obtain a `KStream` from a topic, we use the `stream()` method of the StreamsBuilder , shown as follows:

```java
// Stream DSL
KStream<Integer, Message> initialStream = streamsBuilder.stream(Constants.getMessageTopic(), Consumed.with(Serdes.Integer(), AppSerdes.Message()));
```

There is an implementation of the `stream()` method that just receives the topic name as a parameter. But, it is good practice to use the implementation where we can also specify the serializers, as in this example we have to specify the Serializer for the key and the Serializer for the value for the Consumed class; in this case, both Key is of type Integer and Value is of type `Message`.


## Running the Stream processor
To build the project, run this command from the `streamprocessor` directory:

```bash
$ gradle build
```