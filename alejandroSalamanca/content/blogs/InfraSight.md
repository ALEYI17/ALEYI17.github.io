---
title: "InfraSight: Linux & Kubernetes Observability from Syscall to Dashboard"
date: 2025-06-13T13:52:20-05:00
draft: false
author: "Alejandro Salamanca"
image: https://searchvectorlogo.com/wp-content/uploads/2023/06/ebpf-io-authors-logo-vector.png
github_link: https://github.com/ALEYI17/InfraSight
tags:
  - eBPF
  - Linux
  - Networking
  - Observability
  - Security
---

## Introduction Why This Project Exists

In modern systems built on Linux and Kubernetes, many observability tools focus on high level telemetry data like metrics, logs, and traces but often leave critical gaps in understanding what’s truly happening at the system level.

With **eBPF**, we can gain deep, real time visibility into low level kernel activity, all with minimal overhead.

To enable this, I created **InfraSight** an open source observability and auditing platform that uses eBPF to trace system activity such as **process execution**, **file access**, and **network connections** in real time. These events are **enriched with contextual metadata**, such as container image, name, and labels when running inside containers, and are then stored in a **ClickHouse** database optimized for high performance analytics.

InfraSight is designed to run on both **standalone Linux hosts** (as an executable or Docker container) and **Kubernetes clusters**, empowering engineers, SREs, and security teams to **deep dive into system activity across all nodes** with precision and insight.

## Architecture

![InfraSight Architecture](https://github.com/ALEYI17/InfraSight/blob/main/docs/images/infrasight.png?raw=true)

InfraSight is architected to deliver **deep system visibility and auditing** for both **standalone Linux environments** and **Kubernetes clusters** by leveraging **eBPF** to monitor kernel level activity in real time. Its architecture follows a modular pipeline focused on **efficient event collection**, **enrichment**, and **high performance storage and analysis**.

### **1. eBPF Tracing Agents**

InfraSight deploys lightweight **eBPF based agents** on each target node. These agents load custom eBPF programs that hook into key **Linux kernel tracepoints** and **kprobes** to monitor low level system events:

* **`execve`**: Captures process execution and command invocation.
* **`openat`**: Monitors file access activity.
* **`fchmodat`**: Observes file permission changes.
* **`inet_csk_accept`**, **`tcp_v4_connect`**, **`tcp_v6_connect`**: Track network socket connections and accept events.

These agents operate entirely in kernel space for high efficiency, capturing structured events with **minimal CPU and memory overhead**.

### **2. Real Time Local Enrichment**

Before transmitting data, each agent enriches raw kernel events with contextual metadata, such as:

* Translating **UIDs** to actual **usernames**.
* Extracting **container context** (ID, image, labels) using cgroups.
* Calculating **latency** between syscall entry/exit.
* Adding relevant process hierarchy data.

This allows early transformation of raw kernel data into structured telemetry with contextual awareness.

### **3. Central Event Processing Server**

Enriched telemetry is streamed over **gRPC** to a centralized **InfraSight Server**. This server is responsible for:

* Performing additional normalization (e.g., formatting timestamps to ISO8601).
* Final latency adjustments and derived field calculations.
* Coordinating ingestion into the analytics backend.

Its modular design makes it easy to expand with future capabilities like tagging, threat detection, or real time alerting.

### **4. High Speed Event Storage (ClickHouse)**

All structured telemetry is stored in **ClickHouse**, a powerful column oriented database built for analytics at scale. It supports:

* Fast querying over large event volumes.
* Efficient compression and low latency response times.
* Seamless integration with visualization tools like Grafana or SQL dashboards.

Data in ClickHouse includes syscall metadata, user and container context, and precise timing information.

### **5. Observability and Dashboards**

With all telemetry centrally stored, InfraSight enables rich **data exploration** and **real time visualization**:

* Build custom dashboards using **Grafana**.
* Perform deep forensic queries (e.g., “Which processes opened sensitive files?”).
* Detect anomalies in syscall behavior for security and performance use cases.

InfraSight makes syscall level data **accessible and actionable**.


### **6. Flexible, Modular Deployment**

InfraSight is designed for diverse environments:

* **Kubernetes Clusters**: Use the InfraSight Controller to deploy eBPF agents as DaemonSets, centrally manage probe configurations, and collect telemetry from all nodes.
* **Standalone Linux Hosts**: Deploy the loader as a binary or Docker container for monitoring individual systems without cluster overhead.

Whether you're running a homelab, managing production nodes, or investigating security incidents, InfraSight can adapt to your deployment model.

## How It Works

InfraSight is built around a modular data pipeline that uses **eBPF programs** to trace low level system activity, enriches the data with context, and sends it to a central server for analysis and storage. Here's a breakdown of how the system works from kernel space to the database.

### **eBPF Programs: Tracing System Activity**

At the core of InfraSight are several **custom eBPF programs** written in C and compiled using [`bpf2go`](https://pkg.go.dev/github.com/cilium/ebpf/cmd/bpf2go), a tool provided by the Cilium eBPF library. These programs are attached to specific **tracepoints** and **kprobes** in the Linux kernel to monitor key syscalls in real time with minimal performance impact.

#### Tracepoint-based Programs

The following eBPF programs use **tracepoints**, which are stable and predefined instrumentation points in the kernel:

| Syscall    | Tracepoints                               | Type       | Purpose                                                         |
| ---------- | ----------------------------------------- | ---------- | --------------------------------------------------------------- |
| `execve`   | `sys_enter_execve`, `sys_exit_execve`     | Tracepoint | Captures process execution, including arguments and exit status |
| `openat`   | `sys_enter_openat`, `sys_exit_openat`     | Tracepoint | Tracks file open operations and filenames                       |
| `fchmodat` | `sys_enter_fchmodat`, `sys_exit_fchmodat` | Tracepoint | Observes file permission changes                                |

These tracepoints provide access to syscall arguments and return values, enabling fine grained visibility into process and file activity.

#### Kprobe based Programs

Other eBPF programs use **kprobes and kretprobes**, which dynamically instrument kernel functions. This is useful for syscalls that don’t have stable tracepoints:

| Event     | Kernel Functions                     | Type             | Purpose                                                       |
| --------- | ------------------------------------ | ---------------- | ------------------------------------------------------------- |
| `accept`  | `inet_csk_accept` (entry and return) | kprobe/kretprobe | Detects incoming TCP connections (e.g., new accepted sockets) |
| `connect` | `tcp_v4_connect`, `tcp_v6_connect`   | kprobe/kretprobe | Captures outbound connection attempts (IPv4/IPv6)             |

These probes give visibility into network activity, allowing InfraSight to monitor both inbound and outbound connections on each node.

### **User Space Agent: Initial Enrichment and Streaming**

Each node runs a lightweight **user space agent** alongside the eBPF programs. This agent:

Reads raw event data from a **ring buffer** shared with the kernel.
Performs **first stage enrichment**, such as:
Resolving UID/GID to **usernames**.

Identifying whether the process is running in a container.

Converts timestamps, serializes data, and sends it to the **central InfraSight server** using **gRPC streaming**.

This separation between kernel and user space keeps the eBPF programs fast and simple, while the agent handles heavier processing tasks.

### **Server: Central Processing and Storage**

The **InfraSight Server** receives the enriched event stream from all connected agents and processes it in **batches**. Its responsibilities include:

**Final enrichment**:

Formatting timestamps to **ISO8601**.

Converting **latency** values from nanoseconds to milliseconds.

Applying any additional data transformations or filters.

**Batching and writing to storage**:
Events are grouped by batch size or flush interval.

Data is written to a **ClickHouse** database optimized for fast querying and analytics.

This central component enables efficient ingestion and supports high volume environments while preserving low latency insights.

## Use Cases / What You Can See


InfraSight gives you deep, syscall level visibility across your systems, unlocking use cases that go far beyond traditional metrics or logs.

Imagine you’re monitoring a Linux server or a Kubernetes node, and you suddenly want to understand **which processes are being spawned**, **what files they are touching**, and **who they are talking to over the network**. InfraSight lets you answer those questions in real time.

Let’s look at how.

When a new binary is executed using `execve`, InfraSight captures the full command line, the user who launched it, the container (if any) where it ran, and even the parent process tree. This is invaluable for detecting **suspicious process chains** like an unexpected shell being launched by a web server or a cryptominer started from an ephemeral container.

If a process accesses a file using `openat`, you’ll know exactly **which file was opened** and who accessed it. This is critical for monitoring **unauthorized access to sensitive files**, or detecting if a system binary is being tampered with.

Changes to file permissions via `fchmodat` are captured with both the **original and new modes**, helping security teams detect **unexpected privilege escalations** like a script changing its own permissions to become executable after being downloaded.

With network events, InfraSight goes a step further. When a process initiates a connection using `connect`, it logs the **remote IP**, **port**, and **local process identity**. This lets you trace **outgoing connections to unknown destinations**, which could indicate data exfiltration or command and control activity.

Similarly, when a process accepts a new incoming socket using `accept`, you can trace **who accepted the connection**, **from where**, and whether it happened inside a container. This level of introspection is key for tracking **exposed services**, misconfigurations, or unexpected behavior in microservices.

From this data, you can:

* Build real time **dashboards** that show which processes are making outbound connections.
* Create **audit trails** of every executed binary across your fleet.
* Analyze trends in file access and build **access patterns per service**.
* Feed the data into a **machine learning model** to detect anomalous behavior, like rare processes, unexpected file writes, or abnormal connection timing.
* Detect **container escapes** or **supply chain attacks** by tracing cross namespace activity and sudden spikes in system call frequency.

Whether you're an SRE, a security engineer, or just someone trying to understand your infrastructure better, InfraSight turns low level system noise into structured, actionable insights.


##  What’s Next / Future Work

InfraSight was built with extensibility in mind. While the current version already provides powerful visibility into low level system behavior, there are several features and improvements planned to make it even more useful:

 **Threat Detection & Rules Engine**
  Integrate a detection layer that uses rule based logic to flag common attack patterns such as unexpected shell spawns, privilege escalation attempts, or suspicious network connections. Think of it as lightweight runtime security, built directly on top of syscall visibility.

 **Anomaly Detection & Behavioral Profiling**
  Incorporate statistical analysis or machine learning to detect anomalies for example, a sudden spike in file access or a rare binary executing inside a container. This would help identify zero day activity or misconfigurations before they become incidents.

 **Web UI or CLI for Exploration**
  While InfraSight already stores structured, enriched data, the plan is to develop a more interactive experience either through a lightweight web dashboard or a CLI that lets you query, filter, and pivot across telemetry data.

 **Built in Dashboards (Grafana/Metabase)**
  Provide ready to use dashboards for different roles: system health for SREs, file/network access maps for security, and per container activity heatmaps for DevOps teams.

 **Additional eBPF Programs**
  Expand the catalog of syscalls being traced such as `unlink`, `mount`, or `clone` to broaden InfraSight’s coverage of filesystem and process behavior.

## Links / Get Started

You can dive into InfraSight right now:

  **Documentation**
  👉 [https://aleyi17.github.io/InfraSight/](https://aleyi17.github.io/InfraSight/)
  The official docs include architecture, setup guides, deployment methods, and more.

  **Main GitHub Repository**
  👉 [https://github.com/ALEYI17/InfraSight](https://github.com/ALEYI17/InfraSight)
  Source code, all live here.

Whether you’re monitoring a Kubernetes cluster, analyzing workload behavior, or experimenting with eBPF based observability InfraSight is a solid foundation for low level introspection.

## Final Thoughts

eBPF is transforming the way we observe and understand modern systems. With its ability to trace low level events directly in the Linux kernel, it unlocks a new level of visibility without the overhead or complexity of traditional tools.

InfraSight builds on this foundation to give engineers, SREs, and security teams real time insights into system activity. By capturing syscall level events and enriching them with context like container metadata, usernames, and human readable timestamps it makes it possible to answer questions that were previously out of reach.

From monitoring what processes are doing on each node, to tracing network activity and auditing sensitive file access, InfraSight is designed to be both powerful and practical. And by storing this data in ClickHouse, it’s ready for dashboards, anomaly detection, and long term analysis at scale.

Your feedback, ideas, and contributions are welcome whether you're an engineer, researcher, or just curious about how modern systems actually work. This is just the beginning, and there’s so much more we can build together.
