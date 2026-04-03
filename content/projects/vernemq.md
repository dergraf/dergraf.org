+++
title = "VerneMQ"
description = "Distributed MQTT message broker for IoT at scale. Founded as a company."
weight = 2

[extra]
url = "https://vernemq.com"
repo = "https://github.com/vernemq/vernemq"
language = "Erlang"
+++

VerneMQ is a distributed MQTT message broker built on Erlang/OTP. I created it to handle large-scale IoT workloads — millions of concurrent connections with low latency and clustering built in from the start.

I founded a company around VerneMQ and ran it commercially. The project taught me the operational reality of distributed systems: state replication under partition, eventually consistent subscription routing, and what "high availability" actually means when customers depend on it.
