---
title:  Building a Smart Network Interface Card on FPGA – Major Project Edition
slug: smart-nic-on-fpga
author: Utkarsh M
date: '2024-07-20'
categories:
  - Development
tags:
  - guide
  - linux
  - fpga
  - verilog
  - veriloghdl
  - namespaces
  - veth
  - microblaze
  - c
  - axi

---


[Project GitHub Link](https://github.com/Utkar5hM/fpga-based-packet-processing-unit)

In this post, I’ll walk through how we built a Smart NIC on an FPGA as part of our final-year major project. This project was developed with my teammates [Chinmay](https://www.linkedin.com/in/chinmay-sharma-9a68b8200/) and [Muthukumar](https://www.linkedin.com/in/muthuku37/). The concept stemmed from our prior exposure to XDP/eBPF, which led us to explore hardware offloading. While our initial goals were ambitious, the timeline and complexity forced us to adapt along the way.

---

## Project Objectives

Originally, the project aimed to:

1. Implement a NIC in Verilog, optionally integrating existing cores like [LiteEth](https://github.com/enjoy-digital/liteeth).
2. Interface the NIC with a host using PCIe or UART.
3. Incorporate offload logic for packet processing directly into the NIC.
4. Write C drivers for the NIC on a Linux host.

The architecture concept:

![architecture](/assets/img/other/smart-nic-fpga/arch.png)

---

## Constraints & Course Corrections

Due to procurement delays and tight timelines, we adapted our approach:

- Used **Nexys 4 DDR** (limited to 100 Mbps Ethernet and UART communication).
- Switched from pure Verilog implementation to **MicroBlaze soft processor** for control logic.
- Utilized C to implement transmission, reception, and firewall logic.
- Replaced hardware offload logic with IP blocks like TCAM (where feasible).

Despite the pivots, the experience turned out to be an excellent hands-on learning exercise in Verilog, C, and system integration with FPGAs.

---

## Working with Nexys 4 DDR and MicroBlaze

We followed the guide [Getting Started with MicroBlaze Servers](https://digilent.com/reference/learn/programmable-logic/tutorials/nexys-4-ddr-getting-started-with-microblaze-servers/start?redirect=1), which uses a prebuilt TCP echo server and the LWIP stack.

However, newer versions of Vivado don’t include the MII to RMII bridge IP required for the Nexys 4 DDR’s Ethernet. We used **Vivado 2019** to address this, and uploaded the necessary IP to our GitHub repository for future reuse.

By following the steps in the guide and SDK setup, we achieved a basic working TCP echo server:

![bd1](/assets/img/other/smart-nic-fpga/bd1.png)  
![echosvr](/assets/img/other/smart-nic-fpga/echosvr.png)

---

## Capturing Ethernet Frames

The LWIP stack internally uses **EmacLite** drivers. After reviewing its documentation, we implemented packet capture using **polling mode** (preferred for performance, similar to DPDK).

We based our logic on the [ping reply example](https://github.com/Xilinx/embeddedsw/blob/master/XilinxProcessorIPLib/drivers/emaclite/examples/xemaclite_ping_reply_example.c) and extended it for deeper packet inspection.

![ethframe](/assets/img/other/smart-nic-fpga/ethframe.png)

---

## Host-FPGA Communication (via UART)

We used **UARTLite** for serial data transfer. Due to MicroBlaze's lack of multithreading, we assigned EmacLite to polling mode and UARTLite to interrupt-driven mode.

To support UART interrupts, we updated the block diagram and added the necessary concat logic:

![bdmid](/assets/img/other/smart-nic-fpga/bdmid.jpeg)

Note: Later we realized EmacLite polling is non-blocking, making parts of this setup redundant—but the learning was valuable.

---

## Avoiding Kernel Driver Development

Instead of writing a full Ethernet driver in C, we used **Linux namespaces** and **virtual Ethernet (veth) interfaces** to simulate packet flow. This made testing easier and eliminated the need for a second physical host.

We configured the setup as follows:

- Used a veth pair (`vethmp0`, `vethmp1`) in separate namespaces.
- Matched MAC/IP of `vethmp0` with FPGA's Ethernet interface.
- Used **Scapy** for sending/receiving packets.

Example setup script:

```bash
ip netns add mp_nwk2
ip link add vethmp0 type veth peer name vethmp1
ip link set vethmp1 netns mp_nwk2
...
ip netns exec mp_nwk1 python veth2uart.py
```

---

## Offloading with TCAM (Firewall Logic)

We used **TCAM (Ternary Content-Addressable Memory)** for fast IP matching in firewall logic. Due to licensing issues with commercial IPs, we integrated an open-source Verilog TCAM: [mcjtag/tcam](https://github.com/mcjtag/tcam).

To make it work with MicroBlaze, we wrapped it in an **AXI Lite slave interface**, letting the processor communicate with it through memory-mapped I/O.

Useful learning references:
- [ZipCPU AXI Lite tutorial](https://zipcpu.com/blog/2020/03/08/easyaxil.html)
- [AXI wrapper creation YouTube guide](https://www.youtube.com/watch?v=BONXPKXyJTk)

Block diagram:

![axitcam](/assets/img/other/smart-nic-fpga/axitcam.jpeg)  
Final integrated diagram:

![bdf](/assets/img/other/smart-nic-fpga/bdf.jpeg)

---

## Writing C Drivers for AXI IP

Using the auto-generated AXI templates, we wrote drivers to:

- Insert entries into TCAM
- Query and extract matched values

Custom functions like `ECEMPTCAMIP_SetKey`, `ECEMPTCAMIP_GetKey`, and value extractors were added. A complete testbench verified functionality.

Example result:

![TCAM_IP.jpg](/assets/img/other/smart-nic-fpga/TCAM_IP.jpg)

---

## Ethernet Frame Reception Logic

Diagram:

![det_reception](/assets/img/other/smart-nic-fpga/det_reception.jpeg)

Flow:

1. Poll EmacLite for frames.
2. If Ethertype = IPv4, extract IP.
3. Run IP through TCAM.
4. If allowed, prepend size and send via UART.

Python Scapy script to read UART:

```python
import serial
from scapy.all import Ether, sendp
ser = serial.Serial('/dev/ttyUSB1')
...
sendp(frame, iface="vethmp0")
```

Test result:

![receptest](/assets/img/other/smart-nic-fpga/receptest.jpeg)

---

## Ethernet Frame Transmission

Diagram:

![Transmission.jpg](/assets/img/other/smart-nic-fpga/Transmission.jpg)

Flow:

1. Use Scapy to sniff packets on `vethmp0`.
2. If source MAC matches FPGA, serialize, pad, append size, send via UART.

```python
from scapy.all import sniff, Ether
import serial
...
ser.write(tosend)
```

On the FPGA, UART interrupt handler extracts the frame and sends via EmacLite.

Test result:

![veth2uart.jpeg](/assets/img/other/smart-nic-fpga/veth2uart.jpeg)

---

## Final Integration & Testing

We integrated all modules and ran tests. While pings resulted in ARP requests and partial success, malformed packets appeared in Wireshark.

![final.jpeg](/assets/img/other/smart-nic-fpga/final.jpeg)  
![wireshark_err.jpeg](/assets/img/other/smart-nic-fpga/wireshark_err.jpeg)

Despite that, the overall project provided valuable insights into FPGA-based NIC design and AXI integration.

---

## Acknowledgments & Resources

Key references used during development:
1. [Inbasekaran Perumal](https://www.linkedin.com/in/ACoAADMaz7wBukhl_Dsrd93MskvZwKDe93P5V6Q)
2. [Digilent MicroBlaze Server Guide](https://digilent.com/reference/learn/programmable-logic/tutorials/nexys-4-ddr-getting-started-with-microblaze-servers/start?redirect=1)
3. [Xilinx EmacLite Examples](https://xilinx.github.io/embeddedsw.github.io/emaclite/doc/html/api/example.html)
4. [NetFPGA TCAM IP](https://github.com/NetFPGA/NetFPGA-SUME-public/wiki/NetFPGA-SUME-TCAM-IPs)
5. [MicroBlaze AXI IP Tutorials](https://www.youtube.com/watch?v=EjnH-CgOp2g)
6. [AXI Lite Easy Tutorial](https://zipcpu.com/blog/2020/03/08/easyaxil.html)
7. Many others linked throughout the blog...

---

We hope this guide serves as a helpful starting point for anyone aiming to build basic NICs or network offload modules using FPGA hardware. While not production-grade, the learnings and techniques can be built upon for more sophisticated systems.
