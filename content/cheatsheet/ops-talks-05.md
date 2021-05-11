+++
aliases = ["ops-talks-05"]
categories = ["Syntax Highlighting"]
date = 2021-04-19T08:17:23Z
slide = "https://slides.com/floriandambrine/ops-talks-05-kafka"
tags = ["cheatsheet", "ops-talks-05", "kafka", "messaging", "confluent"]
title = "Ops-Talks #05 - Kafka"
toc = false
+++

### Kafka Listeners

While developing in a **Docker** environment it's important to get a good understanding of Kafka listeners / advertised listeners. This notion is really well explained in this article: [Kafka Listeners Explained](https://www.confluent.io/blog/kafka-listeners-explained/)

{{< figure src="https://github.com/Lowess/kafka-listeners-explained/blob/master/kafka-listeners.png?raw=true" class="center" >}}


> You need to set advertised.listeners (or `KAFKA_ADVERTISED_LISTENERS` if you’re using Docker images) to the external address (host/IP) so that clients can correctly connect to it. Otherwise, they’ll try to connect to the internal host address—and if that’s not reachable, then problems ensue.

You can also use this Github repository to play around with this notion:
* [https://github.com/Lowess/kafka-listeners-explained](https://github.com/Lowess/kafka-listeners-explained)

<!--more-->

### Kafka Partitioning

Here is a common rule of thumb for parition sizing in a Kafka cluster

> Every use case deserve its own reflexion... Do not apply these settings blindly

| Workload         | Partition Sizing |
| ---------------- | ---------------- |
| Common           | 8 - 16           |
| Big topics       | 120 - 200        |
| **You're wrong** | >200             |

### Kafka Tips & Tricks
https://github.com/birdayz/kaf is an amazingly easy Kafka client to use and install

If you feel lazy installing it, run it in docker:

```bash {linenos=table,linenostart=1}
alias kaf="docker run --entrypoint="" -v ~/.kaf:/root/.kaf -it lowess/kaf bash"
```

Here are a couple of **tips and tricks** you can do with this cli:

* Consume the content `<TOPIC>` and copy it to a file

```bash {linenos=table,linenostart=1}
kaf consume <TOPIC> --offset latest 2>/dev/null | tee /tmp/kafka-stream.log
```

* Consume the content `<TOPIC>` and generate reformat payload to keep url and uuid only

```bash {linenos=table,linenostart=1}
kaf consume <TOPIC> 2>/dev/null | jq '. | {"url": .url, "uuid": .uuid}'
```

* Send each line from `<FILE>` as individual records to `<TOPIC>`

```bash {linenos=table,linenostart=1}
cat <FILE> | while read line; do echo $line | kaf produce <TOPIC>; done
```

### Resources

* :thumbs_up: [Kafka Listeners Explained](https://www.confluent.io/blog/kafka-listeners-explained/)
* :thumbs_up: [Apache Kafka Rebalance Protocol, or the magic behind your streams applications](https://medium.com/streamthoughts/apache-kafka-rebalance-protocol-or-the-magic-behind-your-streams-applications-e94baf68e4f2)
