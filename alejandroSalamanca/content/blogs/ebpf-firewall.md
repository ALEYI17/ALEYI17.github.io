---
title: "eBPF Firewall in Action: Blocking IPs with XDP"
date: 2025-03-24T17:03:23-05:00
draft: false
image: https://searchvectorlogo.com/wp-content/uploads/2023/06/ebpf-io-authors-logo-vector.png
author: "Alejandro Salamanca"
github_link: "https://github.com/ALEYI17/xdp-simple-firewall"
tags:
  - eBPF
  - Firewall
  - Networking
  - XDP
  - GO
---

### **Introduction**  

In this blog, we will build a **basic firewall using eBPF**. The goal of this firewall is to **block specific IP addresses** using a **REPL (Read-Eval-Print Loop) written in Go**. To achieve this, we will write the **kernel-space code in C** and the **user-space code in Go**, leveraging the [Cilium eBPF library](https://github.com/cilium/ebpf).  

This library allows **communication between user-space and kernel-space** eBPF programs using **maps**, which are essential data structures in eBPF. There are multiple types of maps, but for this project, we will use a **hash table** to store blocked IP addresses efficiently.  

This blog is designed as a **beginner-friendly introduction** to eBPF programming. We will keep things simple and focus on **core concepts** to ensure the tutorial is accessible and easy to follow. Hopefully, by the end of this guide, you will have learned something new and gained a **better understanding of eBPF**.

### **Setting Up the Development Environment**  

To start developing **eBPF programs**, you need to install a few essential tools. These tools will help you **compile, manage, and interact with eBPF programs** effectively.  

#### **1. Clang/LLVM**  
Clang is the **C compiler** that allows us to compile our eBPF program into **eBPF bytecode**, which can then be loaded into the Linux kernel. LLVM provides backend support for Clang and is required for compilation.  
🔗 **Installation Guide:** [Clang/LLVM Setup](https://clang.llvm.org/get_started.html)  

#### **2. Go (Golang)**  
Go is used for the **user-space program**, which will interact with the eBPF kernel program. We will use the **Cilium eBPF library** in Go to load and manage the eBPF program.  
🔗 **Installation Guide:** [Go Installation](https://go.dev/doc/install)  

#### **3. bpftool**  
`bpftool` is a powerful utility that helps us **inspect, load, and manage eBPF programs**. It also allows us to generate **C header files** from compiled eBPF programs, making it easier to interact with eBPF maps.  
🔗 **Installation Guide:** [bpftool Repository](https://github.com/libbpf/bpftool)  

Once you have installed these dependencies, you will be ready to start writing the **eBPF firewall**! 

### **Understanding eBPF and XDP**  

As mentioned in the last blog, **eBPF** allows us to dynamically load code into the **Linux kernel** to modify its behavior **without altering the kernel source code**. In this case, we will be using **XDP (eXpress Data Path)**, a high-performance networking technology built on top of eBPF.  

#### **What is XDP?**  
XDP enables developers to **hook into the earliest stage of the networking stack**, processing packets **directly at the network interface (NIC)**. This makes XDP **extremely fast** because it allows packet processing **before the kernel’s networking stack gets involved**, reducing the overhead of **context switching between kernel space and user space**.  

#### **Why Use XDP for a Firewall?**  
Since XDP runs at the **lowest level of the networking stack**, it provides several benefits for firewall applications:  
-  **High Performance:** XDP bypasses the traditional kernel networking stack, reducing latency and increasing throughput.  
-  **Early Packet Filtering:** Since XDP operates directly at the **NIC level**, it allows us to **block unwanted packets before they consume system resources**.  
-  **Low Overhead:** Traditional firewall solutions (like iptables) require processing in kernel space and user space, but XDP **minimizes this overhead** by keeping everything in kernel space.  

### **Writing the eBPF Kernel Program (C Code)**  

The first step in writing our **eBPF firewall program** is to generate the `vmlinux.h` file. This header file contains **all the relevant information about our Linux kernel**, including the necessary **structures and types** used in eBPF programs.  

#### **Generating vmlinux.h**  
To create `vmlinux.h`, we use **bpftool**, which extracts the kernel’s BTF (BPF Type Format) information and converts it into a C header file. Run the following command:  

```sh
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```  

This command dumps the **kernel’s type information** into `vmlinux.h`, allowing our eBPF program to access **networking structures** and other kernel definitions.  

#### **Creating the eBPF Program**  

Now that we have `vmlinux.h`, we can start writing the actual **eBPF kernel code**. In this example, we will create a file called **`xdp_firewall.bpf.c`** and begin by including the necessary headers.  

#### **Importing Required Header Files**  

```c
//go:build ignore
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>
```  

#### **Understanding the Headers**  
- **`vmlinux.h`** → This is the **generated header** containing kernel structures.  
- **`bpf/bpf_helpers.h`** → Provides **helper functions** that simplify eBPF programming (e.g., logging, map manipulation).  
- **`bpf/bpf_endian.h`** → Handles **endianness conversion**. In networking, most protocols use **big-endian**, while many CPUs (like x86) use **little-endian**, so this header helps with conversions.  

In eBPF programs, specifying a **license** is mandatory. If no license is provided, the **Linux kernel will reject the eBPF program**. To comply with this requirement, we define the license as follows:  

```c
char __license[] SEC("license") = "Dual MIT/GPL";
```  

This declaration tells the kernel that our **eBPF program is licensed under both MIT and GPL**, making it compatible with the Linux kernel's licensing requirements.

#### **Defining Data Structures**  

To manage **IP blocking**, we define a structure to store **IP addresses and their status** (allowed or denied).  

```c
enum ip_status {
    ALLOW = 0,
    DENY = 1
};

struct ip_entry {
    enum ip_status status;
    __u32 ip;
};
```  

- **`enum ip_status`** → Defines **two possible states** for an IP: `ALLOW` (0) or `DENY` (1).  
- **`struct ip_entry`** → Stores:
  - `status`: Whether an IP is allowed or blocked.
  - `ip`: The **IP address** to be stored in the map.  



#### **Defining an eBPF Map for IP Blocking**  

eBPF **maps** allow communication between **kernel space and user space**. We will create a **hash map** (`BPF_MAP_TYPE_HASH`) to store the **list of blocked IPs**.

```c
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, __u32);
    __type(value, struct ip_entry);
    __uint(max_entries, 256);
} block_list SEC(".maps");
```

- **Map Type** → `BPF_MAP_TYPE_HASH`: Similar to a **hash table**, used for fast lookups.  
- **Key** → `__u32`: The key is the **IP address** (stored as a 32-bit integer).  
- **Value** → `struct ip_entry`: Each IP address is associated with an **entry containing its status (allow/deny)**.  
- **Max Entries** → `256`: The map can store up to **256 blocked IP addresses**.  
- **Section** → `.maps`: This tells the eBPF verifier that this is a **map declaration**.  
- **Name** → block_list: This define the name of the map.

To process network packets efficiently, we need a **helper function** that extracts the **source IP address** from an **IPv4 packet**. Here’s the function:

```c
static __always_inline int parse_ip_src_addr(struct xdp_md *ctx, __u32 *ip_src_addr) {
    void *data_end = (void *)(long)ctx->data_end;
    void *data     = (void *)(long)ctx->data;

    // First, parse the Ethernet header.
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) {
        return 0;
    }

    // Check if the packet is an IPv4 packet.
    if (eth->h_proto != bpf_htons(ETH_P_IP)) {
        return 0; // Not an IPv4 packet, so we don't process it.
    }

    // Parse the IP header.
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) {
        return 0;
    }

    // Extract and store the source IP address.
    *ip_src_addr = (__u32)(ip->saddr);
    return 1; // Successfully extracted the IP address.
}
```
This function works by receiving a packet (`ctx`) and a pointer (`ip_src_addr`) to store the extracted IP address. The first step is to extract the **Ethernet header**. If the packet is too small to contain a complete Ethernet header, the function exits early to avoid errors. Next, it checks whether the packet is **IPv4** by examining the protocol field. If the packet is not IPv4, it is ignored, ensuring that only relevant packets are processed.  

After confirming that the packet is IPv4, the function proceeds to extract the **IP header**. If the packet is too small to contain a full IP header, it also exits early. Finally, it retrieves the **source IP address (`saddr`)** and stores it in the provided variable. This function ensures that only valid IPv4 packets are processed, improving efficiency and providing a reliable way to analyze network traffic within the eBPF program. 

#### **Writing the XDP Filtering Logic**  

Now that we have our helper function to extract the source IP address, let’s write the **main filtering function**. This is the core of our eBPF firewall.  

#### **The XDP Firewall Function**  

```c
SEC("xdp")
int filter_xdp(struct xdp_md *ctx){
  __u32 ip;
  
  if (!parse_ip_src_addr(ctx, &ip)){
    return XDP_PASS;
  }
  
  bpf_printk("Extracted IP: %u.%u.%u.%u",
    (bpf_ntohl(ip) >> 24) & 0xFF,
    (bpf_ntohl(ip) >> 16) & 0xFF,
    (bpf_ntohl(ip) >> 8) & 0xFF,
    bpf_ntohl(ip) & 0xFF);

  struct ip_entry *ie;
  ie = bpf_map_lookup_elem(&block_list, &ip);

  if (ie) {
    bpf_printk("IP found in block list: %u, status: %d", ip, ie->status);
    if (ie->status == DENY)
      return XDP_DROP;
  } else {
    bpf_printk("IP not found in block list: %u", ip);
  }

  return XDP_PASS;
}
```

#### **Breaking It Down**  

####  **Marking the Function as an XDP Program**  

```c
SEC("xdp")
int filter_xdp(struct xdp_md *ctx) {
```
The `SEC("xdp")` directive tells the kernel that this function should run as an **XDP program**. This means the function will be attached to a **network interface** and will process packets as soon as they arrive.  

The `ctx` parameter is an `xdp_md` structure that holds metadata about the incoming packet, such as the packet data and its length.  

#### **Extracting the Source IP Address**  

```c
  __u32 ip;
  
  if (!parse_ip_src_addr(ctx, &ip)){
    return XDP_PASS;
  }
```
Here, we declare a variable `ip` and use our `parse_ip_src_addr()` helper function to extract the **source IP address**.  

- If the function **fails** (meaning the packet is **not** an IPv4 packet), we return `XDP_PASS`, allowing the packet to continue through the network stack.  
- If it's an IPv4 packet, we move forward with processing.  

####  **Debugging: Printing the Extracted IP**  

```c
  bpf_printk("Extracted IP: %u.%u.%u.%u",
    (bpf_ntohl(ip) >> 24) & 0xFF,
    (bpf_ntohl(ip) >> 16) & 0xFF,
    (bpf_ntohl(ip) >> 8) & 0xFF,
    bpf_ntohl(ip) & 0xFF);
```
`bpf_printk()` is used for debugging in eBPF. This prints the extracted **source IP address** in human-readable format.  

####  **Checking If the IP Is Blocked**  

```c
  struct ip_entry *ie;
  ie = bpf_map_lookup_elem(&block_list, &ip);
```
This line **looks up the extracted IP address** in our `block_list` map using the helper function `bpf_map_lookup_elem()`.  

- If the IP is **not found**, it means the IP is **not blocked**, so we continue processing.  
- If the IP **is found**, we check its status.  

####  **Applying the Firewall Rule** 

```c
  if (ie) {
    bpf_printk("IP found in block list: %u, status: %d", ip, ie->status);
    if (ie->status == DENY)
      return XDP_DROP;
  } else {
    bpf_printk("IP not found in block list: %u", ip);
  }
```
- If the IP exists in the block list, we **print a debug message** and check its `status`.  
- If the status is `DENY`, we **drop the packet** by returning `XDP_DROP`.  
- Otherwise, the packet is allowed to continue.  


### **Creating the User-Space CLI with Go**  

Now that we’ve written our eBPF firewall program in C, we need a **user-space program** to load it into the kernel and manage blocked IP addresses. We’ll use **Go** and the [Cilium eBPF](https://github.com/cilium/ebpf) library for this.  

####  **Installing Dependencies**  

Before we start coding, install the necessary Go dependencies:  

```sh
go get github.com/cilium/ebpf
go mod init <name-of-your-project>
go mod tidy
go get github.com/cilium/ebpf/cmd/bpf2go
```

This will set up our Go project and fetch the required packages.  

####  **Generating eBPF Skeleton Code**  

We use `bpf2go` to generate Go bindings for our eBPF program. Add the following line at the top of `main.go`:  

```go
//go:generate go run github.com/cilium/ebpf/cmd/bpf2go -target amd64 -type ip_entry ebpf xdp_firewall.bpf.c
```

**What does this do?**  
- It runs `bpf2go`, which compiles `xdp_firewall.bpf.c` and generates **Go bindings**.  
- The `-type ip_entry` flag tells it to generate a Go struct for our `ip_entry` C struct.  
- The generated files will help us interact with **eBPF maps** and load the program.  

####  **Selecting a Network Interface**  

The eBPF program needs to be attached to a specific network interface. In our Go code, we take the interface name as a command-line argument. First, we retrieve the name of the interface from `os.Args[1]`, which allows the user to specify it when running the program. Then, we use `net.InterfaceByName(ifaceName)` to look up the network interface details. If the interface does not exist or there is an error retrieving it, the program logs the issue and exits. This step ensures that our firewall knows exactly which network interface to monitor for incoming packets.  

```go
ifaceName := os.Args[1]
iface, err := net.InterfaceByName(ifaceName)
if err != nil {
	log.Fatalf("lookup network iface %q: %s", ifaceName, err)
}
```  

####  **Loading the eBPF Program**  

Once we have the correct network interface, we need to load our eBPF program into memory. However, before doing that, we call `rlimit.RemoveMemlock()`, which ensures that the system has enough memory resources allocated for loading eBPF programs. Without this, the program might fail due to memory limitations.  

Next, we create an instance of `ebpfObjects{}`. This struct holds references to our eBPF program and maps, allowing us to interact with them from Go. The `loadEbpfObjects(&objs, nil)` function loads our compiled eBPF program from the generated object file into memory. If there is an error during this process, we log it and exit. Finally, we use `defer objs.Close()` to ensure that once the program exits, all resources allocated to the eBPF program are properly cleaned up.  

```go
if err := rlimit.RemoveMemlock(); err != nil {
	log.Fatal(err)
}

objs := ebpfObjects{}
if err := loadEbpfObjects(&objs, nil); err != nil {
	log.Fatalf("Error loading obj: %v", err)
}
defer objs.Close()
```  

####  **Attaching eBPF to the Interface**  

Now that our eBPF program is loaded, we need to attach it to the selected network interface. This is done using `link.AttachXDP()`, which allows us to hook the eBPF program into the network stack at the **XDP (eXpress Data Path)** level.  

The function takes an `XDPOptions` struct, where we specify the program to attach and the interface index. `objs.FilterXdp` refers to our eBPF program function that we compiled earlier, and `iface.Index` ensures that the program is attached to the correct network interface.  

```go
link.AttachXDP(link.XDPOptions{
	Program:   objs.FilterXdp,
	Interface: iface.Index,
})
```  

At this point, our firewall is now actively inspecting network traffic in real time. The eBPF program will intercept every incoming packet on the specified interface and determine whether to allow or block it. With this setup, we have successfully connected the kernel-space eBPF code with a user-space Go program, enabling a dynamic and efficient firewall solution. 

####  **Blocking an IP Address**  

The `block_ip` function is responsible for adding an IP address to the block list, effectively preventing packets from that IP from passing through the firewall.  

First, it converts the IP string into a 32-bit unsigned integer using the `ipToUint32` function. If the conversion fails, it returns an error. Then, it logs the IP address in both string and integer formats for debugging purposes.  

Next, it creates an `ebpfIpEntry` struct, where the `Ip` field holds the converted IP address, and `Status` is set to `1`, which represents **DENY**. The function then interacts with the eBPF map by calling `objs.ebpfMaps.BlockList.Update()`, which adds the IP to the hash map stored in the kernel. The `ebpf.UpdateNoExist` flag ensures that the entry is only added if it doesn’t already exist. If the update operation fails, the function returns an error; otherwise, it successfully blocks the IP.  

```go
func block_ip(ip string, objs *ebpfObjects ) error {
  u32_ip, err := ipToUint32(ip)
  if err != nil {
    return err
  }
  log.Printf("Blocking ip_str:%s , ip_u32:%d", ip, u32_ip)

  blocker := ebpfIpEntry{Ip: u32_ip, Status: 1}
  err = objs.ebpfMaps.BlockList.Update(u32_ip, blocker, ebpf.UpdateNoExist)
  
  if err != nil {
    return err
  }

  return nil
}
```  

####  **Unblocking an IP Address**  

The `unblock_ip` function is the counterpart of `block_ip`, removing an IP address from the block list so that packets from that IP can pass through again.  

Just like in the previous function, it first converts the IP string into a 32-bit integer. If the conversion fails, it returns an error. It then logs the IP address in both string and integer formats for debugging purposes.  

The function interacts with the eBPF map using `objs.ebpfMaps.BlockList.Delete()`, which removes the specified IP from the block list. If the deletion fails (e.g., if the IP was not found in the map), an error is returned. Otherwise, the function successfully unblocks the IP.  

```go
func unblock_ip(ip string, objs *ebpfObjects ) error {
  u32_ip, err := ipToUint32(ip)
  if err != nil {
    return err
  }
  log.Printf("Unblocking ip_str:%s , ip_u32:%d", ip, u32_ip)

  err = objs.ebpfMaps.BlockList.Delete(u32_ip)
  
  if err != nil {
    return err
  }

  return nil
}
```  

####  **Listing Blocked IPs**  

The `print_list` function retrieves and prints all IP addresses that are currently blocked.  

It begins by creating an iterator over the `BlockList` map using `objs.ebpfMaps.BlockList.Iterate()`. This iterator allows us to go through each key-value pair stored in the eBPF map.  

The function then defines two variables: `key` to store the IP address and `value` to hold the corresponding `ebpfIpEntry` struct. It prints a header `"Blocked IPs:"` before entering a loop that iterates over the blocked IPs.  

For each entry found in the map, it converts the integer IP back into a readable string using `uint32ToIP(value.Ip)` and logs it. This allows users to see which IPs are currently blocked by the firewall.  

```go
func print_list(objs *ebpfObjects) {
  itr := objs.ebpfMaps.BlockList.Iterate()

  var key uint32
  var value ebpfIpEntry
  fmt.Println("Blocked IPs:")
  
  for itr.Next(&key, &value) {
    log.Printf("IP: %s", uint32ToIP(value.Ip))
  }
}
```  

These functions together allow users to interact with the eBPF-based firewall dynamically, blocking and unblocking IPs in real-time while also providing visibility into the current block list. 

The rest of the code is not essential for the eBPF part, as it only handles the REPL (Read-Eval-Print Loop) logic, allowing users to interact with the firewall through terminal commands. If you want to check out the full code and see how the command-line interface works, you can find it in the GitHub repository:🔗 [xdp-simple-firewall](https://github.com/ALEYI17/xdp-simple-firewall).

To run the code, simply follow these commands:  

```bash
go build  
sudo ./xdp_firewall <interface>
```  

Replace `<interface>` with the name of your network interface (e.g., `eth0`). This will start the firewall and allow you to interact with it using the REPL.

### **Conclusion**

This project was a learning experience aimed at understanding how eBPF programs and XDP work under the hood. I am not an expert in C or kernel programming, but I did my best by researching examples and experimenting to make this work. I hope this guide helps you grasp the basics and sparks your interest in eBPF.  

eBPF is an emerging technology that is evolving rapidly, so some of the code here may become outdated in the coming years. If you want to dive deeper into eBPF and stay up to date with its advancements, I highly recommend visiting 🔗[ebpf.io](https://ebpf.io), where you can find extensive resources and documentation on developing eBPF programs.