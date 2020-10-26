---
title: "Ops-Talks #02 - AVRO"
aliases:
  - "ops-talks-02"
date: 2020-10-20T12:52:23+08:00
draft: false
tags: ["ops-talks-02", "avro", "schema-registry", "cheatsheet"]
categories: ["Syntax Highlighting"]
toc: false
type: "cheatsheet"
slide: https://slides.com/floriandambrine/ops-talks-02-avro
---

> 📢 Special thanks to our speaker: [Karim Lamouri](https://www.linkedin.com/in/karimlamouri/)

### 1.1 Avro - What is it ?

* Avro is used for serialization and deserialization of payloads
* More compact than `JSON`
* Is a container file - It has **both** `the schema` and `the payload`
* Allows RPC calls
* Fast serialization and smaller than JSON (do not repeat all the JSON keys as the schema is at the top of the file)
* Allows schema documentation
* Format easy to leverage on Spark or ETL pipelines

> You can always deserialize an AVRO file as the schema is embedded in the file itself !

<!--more-->

### 1.2 Avro - Basics

* Avro Schema

```json {linenos=table,linenostart=1}
{
  "namespace": "com.gumgum.avro.verity",
  "type": "record",
  "name": "Callback",
  "fields": [
    {
      "name": "callback_uuid",
      "type": "string"
    },
    {
      "name": "target_url",
      "type": "string"
    }
  ]
}
```

### 1.3 Avro - Avro Protocol

```json {linenos=table,linenostart=1}
{
  "protocol": "Callback",
  "namespace": "com.gumgum.avro.verity",
  "types": [
    {
      "type": "record",
      "name": "Callback",
      "fields": [
        {
          "name": "callback_uuid",
          "type": "string"
        },
        {
          "name": "target_url",
          "type": "string"
        }
      ]
    }
  ],
  "messages": {}
}
```

> Avro Protocol allows the definition of RPC functions. But this schema is still defined in JSON which is not optimal for reading and documenting


### 1.3 Avro - Avro IDL - Higher level representation

* The AVRO IDL was introduced by the Apache AVRO project
* Much shorter writing
* Similar to `C structures`
* Easy definition of `default values`
* Easy documentation
* Allows `import`
* Automatic generation of Java [POJO](https://en.wikipedia.org/wiki/Plain_old_Java_object)
* `POJOs` or generated classes objects from `AVRO` enforce strong consistency on what fields are `mandatory` / `optional` or `null`. This is especially important in a microservice world where events are at the heart of the data flow.

```groovy {linenos=table,linenostart=1}
@namespace("com.gumgum.avro.verity")

protocol Callback {
  record Callback {
    string callback_uuid;
    string target_url;
  }
}
```

* Example with `default value` and `comments`:

```groovy {linenos=table,hl_lines=["5-8"],linenostart=1}
@namespace("com.gumgum.avro.verity")

protocol Callback {
  record Callback {
    /**
      This is the documentation for the Callback record
    **/
    string callback_uuid = 'uuid';
    string target_url;
  }
}
```

* Example with imports

```groovy {linenos=table,hl_lines=["3","12"],linenostart=1}
@namespace("com.gumgum.avro.verity")

import address.idl

protocol Callback {
  record Callback {
    /**
      This is the documentation for the Callback record
    **/
    string callback_uuid = 'uuid';
    string target_url;
    Address callback_address;
  }
}
```

### 2.1 Schema Registry

* [Schema registry](https://docs.confluent.io/current/schema-registry/index.html) is an open-source serving layer of AVRO schemas provided by [Confluent Platform](https://docs.confluent.io/current/)
* Schema registry is in charge of:
    * Registering schemas
    * Returning the ID of the schema

* A `schema` is a living object (it changes over time, new fields get added / removed / updated)
* In a Kafka world, AVRO schemas are not sent along with the payload but a `"pointer"` to the schema is passed down.
* Kafka libraries (`SerDe` or `Librdkafka`) prepends 5 bytes (the `"pointer"`) to the `AVRO binary` to inject schema information.

| Byte 0     | Byte 1-4                                         | Byte 5-...                                      |
|------------|--------------------------------------------------|-------------------------------------------------|
| Magic Byte | 4-bytes schema ID as returned by Schema Registry | Serialized data for the specified schema format |

### 2.2 Schema Evolution & Compatibility

| Compatibility Type  | Changes allowed                              | Check against which schemas     | Upgrade first |
|---------------------|----------------------------------------------|---------------------------------|---------------|
| BACKWARD            | ​Delete fields / Add optional fields         | Last version                    | ​Consumers    |
| BACKWARD_TRANSITIVE | Delete fields Add optional fields            | All previous versions           | ​Consumers    |
| FORWARD             | Add fields / Delete optional fields          | Last version                    | ​Producers    |
| FORWARD_TRANSITIVE  | Add fields / Delete optional fields          | All previous versions           | ​Producers    |
| FULL                | Add optional fields / Delete optional fields | Last version                    | Any order     |
| FULL_TRANSITIVE     | Add optional fields / Delete optional fields | All previous versions           | Any order     |
| NONE                | All changes are accepted                     | Compatibility checking disabled | Depends       |

* Tips for easy schema evolution in a non `TRANSITIVE` world:
    * be `nullable`
    * have `default values` (default value can be `null` as well)
    * fields that are `nullable` and have `default values` can be removed

### 2.3 Schema Evolution & Kafka-connect performances

* [Kafka-connect](https://docs.confluent.io/current/connect/index.html) is an open-source software from [Confluent Platform](https://docs.confluent.io/current/)

* Schema compatibility can directly impact Kafka-connect performance

> Story behind an outage resulting in 8h worth of data lost...
> Back in the day a change was made to a Kafka event where the schema compatibility was set to NONE (where it should have been set to FULL or at least BACKWARD). As a result of this event modification, Kafka-connect performance started dropping resulting in a lot a small files being uploaded to S3 impacting Spark jobs from running efficiently.

* [Checkout the slides to understand the impact of schema compatibility with Kafka-connect](https://slides.com/floriandambrine/ops-talks-02-avro#/4/4)

<iframe src="https://slides.com/floriandambrine/ops-talks-02-avro#/4/4/embed?style=dark" width="100%" height="420" scrolling="no" frameborder="0"
        webkitallowfullscreen mozallowfullscreen allowfullscreen>
      </iframe>

### Resources

* [AVRO Documentation](https://avro.apache.org/docs/current/spec.html)
