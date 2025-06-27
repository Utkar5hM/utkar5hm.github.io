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

During my time at IRIS NITK, I had the opportunity to work with a talented group of peers, While I was  exploring distributed strategies for MariaDB with a friend of mine. While I couldn’t dive deeply into every aspect, I gained valuable exposure, particularly to eBPF, which several guys were using for their own research and internal tools.

After completing my summer internship at Oracle, I began building an eBPF based monitoring tool to understand the it better. Although the initial version was completed, I didn’t revisit it for over a year.

Recently, I decided to revisit the tool and evaluate its performance and usefulness.

[Project GitHub Link](https://github.com/Utkar5hM/mariadb-ebpf-exporter)

## How It Works

The `mariadb-ebpf-exporter` collects query latency metrics by attaching an eBPF probe to the `dispatch_command()` function—present in both MySQL and MariaDB. It then normalizes queries for grouping and exposes the results as Prometheus metrics, which can be visualized using Grafana.

I chose `libbpfgo` due to its CO-RE (Compile Once, Run Everywhere) capabilities and because I wanted to build the userspace in Go. One limitation I faced was that compiled MariaDB/MySQL binaries often have mangled function names. Since `libbpfgo` doesn't support resolving these easily (unlike `bpftrace`), I wrote a build script that uses the `nm` tool to extract the function name and compile the eBPF object accordingly. The BPF binary is then embedded in the Go code using Go’s `embed` package, ensuring compatibility with the target DB version.

To make setup easier, I containerized the entire compilation and runtime process with Docker. This allows running the tool against both host and containerized databases by joining the appropriate namespaces.

## Performance Considerations

There are a few aspects to consider before benchmarking:

- Query normalization uses regular expressions, which can be resource intensive. Offloading normalization/export to a different system could reduce overhead.
- As this is a custom tool, its performance will naturally depend on implementation quality.

I also experimented with filtering to only capture slower queries (e.g., >200ms or >1000ms). While this approach sounds logical, [Percona’s blog](https://www.percona.com/blog/performance_schema-vs-slow-query-log/) explains why it may not always be the best idea. Still, I tried both approaches and compared the results.

## Benchmarking Setup

### Sysbench Scripts

My initial idea was to create a workload where CPU and memory usage hovered around 50–60% under raw conditions, to observe how the tool impacts performance. However, maintaining that balance proved tricky. I eventually settled on using default sysbench scripts, which max out CPU/memory usage and allow us to measure throughput via the number of transactions.

```sh
sysbench oltp_common --db-driver=mysql --table-size=100000 --mysql-user=root --mysql-password=L0CK3D_1N --mysql-port=3306 --mysql-host=10.128.0.3 --mysql-db=sysbenchtest prepare
sysbench oltp_read_write --db-driver=mysql --table-size=100000 --mysql-user=root --mysql-password=L0CK3D_1N --mysql-port=3306 --mysql-host=10.128.0.3 --mysql-db=sysbenchtest --threads=12 --report-interval=1 run
```

I also disabled indexing to deliberately slow down the queries and better visualize the metrics.

![grafana-dashboard-screenshot](/assets/img/other/ebpf-perf/grafana-db.jpeg)

Prometheus query used in Grafana:

```sh
rate(ebpf_exporter_query_latencies_histogram_sum[1m])/rate(ebpf_exporter_query_latencies_histogram_count[1m])
```

Before every test, I ran the following cleanup command:

```sh
sysbench oltp_common --db-driver=mysql --table-size=100000 --mysql-user=root --mysql-password=L0CK3D_1N --mysql-port=3306 --mysql-host=10.128.0.3 --mysql-db=sysbenchtest cleanup
```

### Performance Schema

For configuring performance schema, I followed this guide from the MySQL documentation: [Query Profiling Using Performance Schema](https://dev.mysql.com/doc/mysql-perfschema-excerpt/5.7/en/performance-schema-query-profiling.html)

An example of the latency metrics collected is shown below:

![performance-schema-query-latency-ss](/assets/img/other/ebpf-perf/perf_schema_ex.jpeg)

### Process Exporter

I had the [Process Exporter](https://github.com/ncabatoff/process-exporter) running to log CPU and memory usage, but unfortunately missed the Prometheus retention window and couldn’t recover the data. While I don’t have screenshots to share, I still used the tool during testing.

### System Configuration

All tests were performed on 2 GCP instances with the following configuration:

- `e2-standard-2` (2 vCPU, 1 core, 8 GB memory)  
- OS: Debian

## Benchmarking Results

Initially, collecting all queries (0ms threshold) caused high CPU usage:

![htop-ss](/assets/img/other/ebpf-perf/htop.jpeg)

However, filtering queries with latency over 200ms significantly reduced resource usage, which was also reflected in the benchmark results.

### Results

Each test was run three times, and results were averaged:

| Test                | Read Queries | Write Queries | Total Queries | Transactions | Total Time | Total Events | Min Latency | Avg Latency | Max Latency | 95th Latency |
|---------------------|--------------|---------------|---------------|--------------|------------|--------------|-------------|-------------|-------------|----------------|
| ebpf                | 2108526      | 602436        | 3012180       | 150609       | 300.0397   | 150609       | 8.34        | 47.81       | 375.13      | 89.27           |
| ebpf > 200ms        | 2155041      | 615726        | 3078630       | 153931.5     | 300.0355   | 153931.5     | 7.42        | 46.77       | 378.97      | 94.10           |
| performance_schema  | 1821535.33   | 520438.67     | 2602193.33    | 130109.67    | 300.05895  | 130109.67    | 8.65        | 56.51       | 318.80      | 86.79           |
| raw                 | 2120234.67   | 605781.33     | 3028906.67    | 151445.33    | 300.03707  | 151445.33    | 8.21        | 47.55       | 365.94      | 90.60           |

### Visualizations

Charts generated from Google Sheets:

![no-of-transactions](/assets/img/other/ebpf-perf/nt.png)

![no-of-queries](/assets/img/other/ebpf-perf/nq.png)

![average-latency](/assets/img/other/ebpf-perf/al.png)

![chill-guy](/assets/img/other/ebpf-perf/lazy.jpeg)
