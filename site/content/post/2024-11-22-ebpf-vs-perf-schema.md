---
title: Performance Schema vs eBPF-based Query Profiling
slug: ebpf-vs-perf-schema
author: Utkarsh M
date: '2024-11-22'
categories:
  - Development
tags:
  - linux
  - c
  - ebpf
  - mariadb
  - mysql
  - grafana
  - performance_schema
  - query
  - golang
  - go

---

So, it all started at IRIS NITK, where I had the chance to work with some seriously cracked individuals, and we were testing out different distributed strategies for MariaDB. While I didn’t have a ton of time to dive super deep, I was picking up some gems here and there. The real kicker? A bunch of the guys were working on eBPF-based tools for their own projects/academics, and that’s where I first got exposed to eBPF. Safe to say, the tech rizz was strong.

Once my summer internship wrapped up and I had some breathing room, I decided to channel that extra time into learning eBPF by building my own monitoring tool. I finished building it but didn’t get around to testing or using it for quite some time.

So, I finally decided revisiting it to see if it’s actually any good.

[Project Github Link](https://github.com/Utkar5hM/mariadb-ebpf-exporter)

Before starting the comparison, Let's first see what the program does and how it works. The mariadb/mysql ebpf exporter collects query latencies as so the name suggests by hooking into the initial query execution funcition which is named `dispatch_command()`(_in both mysql/mariadb. but they act differently, very demure_) and then kind of normalizes the query in order to help us categorize similar query into a group, followed by exporting this data in form of prometheus metrics which can be collected by a prometheus server and then further beautifully visualized by Grafana.

I've used libbpfgo for building the program especially due to the CORE(compile once run everywhere) functionality of it and also because I wanted to use Golang for the userspace side of the program. Well, :3 there was a caveat that the compiled mariadb/mysql versions available have mangled function names and libbpfgo cannot go and find the mangled one and hook into it like bpftrace can(at least I didn't know any way), so I created a build script that would utilize `nm` tool to find the function name in the db's binary and store it and compile our bpf binary to hook to it. which is then embedded into the go program with go's `embed` lib. so after this, we can be sure that it would always run with the db version it was compiled for.

If you have ever worked with libbpf, you'd know how much of a pain it is to setup the requirements, so I wanted to make sure that this could be compiled and run with docker. Hence scripts were written and now it could be compiled within a docker container given the kernel specific required package are present in ubuntu's default apt repo list.  now being able to run in a docker container, we can also attach the program to a database running on the host or inside another container. This is achieved by starting the container in respective process namespaces.

Now that we know the capabilities of the program, We need to know how it impacts performance compared to raw (without any monitoring) and pre-existing monitoring methods like performance_schema and slow query logs. But since there already exists a good comparison between the later both([PERFORMANCE_SCHEMA vs Slow Query Log- percona](https://www.percona.com/blog/performance_schema-vs-slow-query-log/)), we will only be comparing it with raw and performance_schema.

There are some thing to consider before we begin testing this thing out.
- query fingerprinting/normalization uses regex and it is intensive. So it might be one of the reasons why the performance might take a lot of hit. One interesting thing that could be done to solve this (if it is), would be to transfer data to another system first and then normalize/export.
- The program I've written could be just baddd even though I might've tried to make it not to be.

I thought it would also be pretty interesting to see how the program would behave when I only try to collect slower queries, like say greater than 200ms or 1000ms, kind of makes sense, right? So, I've done that here but the above linked comparison blog by percona explains why it is not such a good idea to do so and It would be better for you to have a read through it to understand why. It would be pretty interesting to try out their explained better approach and see how that performs as well but maybe another time.

## Benchmarking Configuration

### sysbench scripts
Initially, I wanted to emulate a kind of an scenario by finding a setup where scripts could be run against the database to keep CPU and memory usage around 50-60% under raw conditions. This would allow me to observe how performance dropped with the monitoring tool in action. However, trying to achieve and maintain this balanced state brought its own complications, making it tough to establish a consistent baseline.

Given this, I pivoted to running default sysbench scripts. While this approach will max out CPU/memory usage. It did introduce a new metric we could leverage: the number of transactions executed. This would allow us to gauge throughput and provide another perspective on the tool's performance impact.

```sh
sysbench oltp_common --db-driver=mysql --table-size=100000 --mysql-user=root --mysql-password=L0CK3D_1N --mysql-port=3306 --mysql-host=10.128.0.3 --mysql-db=sysbenchtest prepare
sysbench oltp_read_write --db-driver=mysql --table-size=100000 --mysql-user=root --mysql-password=L0CK3D_1N --mysql-port=3306 --mysql-host=10.128.0.3 --mysql-db=sysbenchtest --threads=12 --report-interval=1 run
```

Additionally, I turned off indexing so that the queries are slower and I could get this nice grafana graph while using the tool:

![grafana-dashboard-screenshot](/assets/img/other/ebpf-perf/grafana-db.jpeg)

query for the above graph:
```sh
rate(ebpf_exporter_query_latencies_histogram_sum[1m])/rate(ebpf_exporter_query_latencies_histogram_count[1m])
```

Before running every test, A cleanup was run with the following command, just to make sure that nothing is cached anywhere.

```sh
sysbench oltp_common --db-driver=mysql --table-size=100000 --mysql-user=root --mysql-password=L0CK3D_1N --mysql-port=3306 --mysql-host=10.128.0.3 --mysql-db=sysbenchtest cleanup
```

### performance schema

For configuring performance schema, I've followed the following guide: [Query Profiling Using Performance Schema](https://dev.mysql.com/doc/mysql-perfschema-excerpt/5.7/en/performance-schema-query-profiling.html)

An example of how the query latency measured would look like:

![performance-schema-query-latency-ss](/assets/img/other/ebpf-perf/perf_schema_ex.jpeg)


### Process Exporter

Okay bet, so I had the [Process Exporter](https://github.com/ncabatoff/process-exporter) running, logging CPU and memory usage like a pro, but I lowkey fumbled and missed the Prometheus retention window. No screenshots, no backup but just vibe.  You’re gonna have to trust me on this one fr.

### System Configuration

2 GCP Instances with following configuration: `e2-standard-2 (2 vCPU, 1 core, 8 GB memory) - OS: Debian`

## Benchmarking

So I kicked things off with my eBPF exporter grabbing 0ms latency queries, and bruh, the exporter was just hogging CPU like it was the main character.

![htop-ss](/assets/img/other/ebpf-perf/htop.jpeg)

Hmm, wellllllll
![muller-meme](/assets/img/other/ebpf-perf/muller.jpg)

This drastically changed when I filtered for queries with latency over 200ms—CPU and memory usage chilled out, and the benchmark results do reflect that.

### Results

I ran each test three times, averaged the numbers, so It would be easy to look at. You can dig into the full results [here](https://github.com/Utkar5hM/utkar5hm.github.io/tree/main/assets/other/ebpf-perf/sysbenchscripts).

Now, here are the benchmark results:

| test - averages    | read_queries | write_queries | total_queries | transactions | total_time  | total_events | min_latency | avg_latency | max_latency | 95th_latency |
| ------------------ | ------------ | ------------- | ------------- | ------------ | ----------- | ------------ | ----------- | ----------- | ----------- | ------------ |
| ebpf               | 2108526      | 602436        | 3012180       | 150609       | 300.0397    | 150609       | 8.343333333 | 47.81666667 | 375.13      | 89.27        |
| ebpf>200ms         | 2155041      | 615726        | 3078630       | 153931.5     | 300.0355    | 153931.5     | 7.425       | 46.77       | 378.97      | 94.1         |
| performance_schema | 1821535.333  | 520438.6667   | 2602193.333   | 130109.6667  | 300.05895   | 130109.6667  | 8.648333333 | 56.50833333 | 318.805     | 86.785       |
| raw                | 2120234.667  | 605781.3333   | 3028906.667   | 151445.3333  | 300.0370667 | 151445.3333  | 8.206666667 | 47.55166667 | 365.9416667 | 90.595       |

Some nice graphs that chad Google sheets generated:

![no-of-transactions](/assets/img/other/ebpf-perf/nt.png)

![no-of-queries](/assets/img/other/ebpf-perf/nq.png)

![average-latency](/assets/img/other/ebpf-perf/al.png)

![chill-guy](/assets/img/other/ebpf-perf/lazy.jpeg)
