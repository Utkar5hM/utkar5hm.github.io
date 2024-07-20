---
title: Building A Smart Network Interface card on FPGA - Major Project edition
date: 2024-07-20 00:00:00 +530
categories: [Guides, Tutorials]
tags: [guide, linux, fpga, verilog, veriloghdl, namespaces, veth, microblaze, c, AXI] # TAG names should always be lowercase
---

Hey there folks, I'm writing here after a long aff time. So, In this blog, I will be discussing about how we implemented our smart NIC card on a FPGA. This was my final year major project that I did with my GYAT bros. The idea for the project popped into our face like acne due to our exposure to xdp/ebpf without a sunscreen, trying to do some hardware offloading kind of a thing for the same.

The project aimed to do several things initially:

1. Get this shi done, get the degree and GTFO (I'm kidding but I'm not.)
2. NIC designed entirely with Verilog, fanum taxing a pre-existing ethernet core like [liteeth](https://github.com/enjoy-digital/liteeth) from the github where the data is serially processed and modifying it to our needs.
3. LAN on FPGA to be connected to the router with a common interface such as PCIe or UART for communication between Host and FPGA(NIC).
4. highly performant custom offloading Logic to be built onto the NIC with Verilog.
5. Write NIC drivers in C for the host linux OS to support the built hardware.

This is how it would look like overall:


![architecture](/assets/img/other/smart-nic-fpga/arch.png)


If you haven't figured it out, the `major project edition` in the title here means something, its:

1. **We were COOKED**, I mean obviously.
2. We didn't do shit for the initial few months and barely had a week left for the **major** part of our MAJOR project.
3. We needed to clutch and take a lotta shortcuts.

So and `hence`, this isn't a full proof guide to building an entirely working industry standard NIC allowing offloads thats gonna be fast aff or do performancemaxxing but about how one can get started to produce a mvp (without the viable part) while knowing or having a lil bit experience with Verilog HDL, Linux(you'll know why later) and C to produce a NIC or whatever someone wants on a FPGA with mostly software oriented knowledge. We wanted to write this blog because we couldn't find a single goto place for such kinda things as everything was just  scattered in the ocean of the internet.

Though we had good enough experience working with Verilog HDL in the past, We had never flashed and played with an FPGA before. Getting one itself from the university's pocket took considerable amount of time. We ended up with **Nexys 4 DDR** because that was the one that the university had and could be used with Vivado as anyone who has ever used quartus knows how much of a bad UI/UX it has (This matters to me :3). 

But sadly this limited us with a 100mbps link on the ethernet interface and only UART as a communication interface between host and the FPGA. (ngl, we could've gotten a better one if we had tried harder, like just if we had asked for one )

Due to our extreme procrastination skills (I'm writing this blog several months after the project so yk), We ended up changing our plans midway through the semester with barely a month left not having done anything to the following:

1. Instead of building transmission and reception logic entirely in Verilog, We will flash a Microblaze soft Core processor onto the FPGA and write the packet transmission and reception logic with C.
2. Use custom or already built IPs (Intellectual properties) with the microblaze for accelerating hardware offloading. like a T-CAM IP (Ternary - Content Addressable Memory) for a firewall allowing for faster IP address lookups with key masks.
3. Most other things were supposed to be the same.

Now coming to the actual Implementation or how-to guide. In the first section, I will discuss on how to get started working with an FPGA (Nexys 4 DDR specifically), as that's what we've ever used.

## Getting Familiar with Nexys 4 DDR and Microblaze

If you've ever googled about anything regarding this project, You'd probably end up with this one specific guide [Nexys 4 DDR - Getting Started with Microblaze Servers](https://digilent.com/reference/learn/programmable-logic/tutorials/nexys-4-ddr-getting-started-with-microblaze-servers/start?redirect=1), Which discuses on how to flash a pre-existing C-based
TCP echo server application from the Xilinx SDK. 

The echo server application is designed to set up a TCP server that echoes back any received text. It leverages the LWIP library, which provides a lightweight and streamlined TCP/IP stack facilitating high-level Ethernet functionality. 

However, It is an extremely old guide and probably won't work straightly if you tried it on the newer vivado versions as they removed the MII to RMII core for the nexys 4 DDR from the package, basically something that's required for our ethernet interfaces to work as expected. If you try to find it online, You won't really do it easily, We just ended up installing Vivado 2019 for the same. At this point, I do have it uploaded on the github repository of this project. so you could probably find it there and add it to the newer version.

As most of the above guide is pretty self explanatory, lets not waste further time. If followed up right, we end up with the following block diagram:

![alt text](/assets/img/other/smart-nic-fpga/bd1.png)

Now we also we get a working TCP echo server after configuring further steps via the SDK.

![alt text](/assets/img/other/smart-nic-fpga/echosvr.png)

So, We now atleast had something running.

## Capturing Raw Ethernet Frames

As previously stated, The echo server internally used LWIP, which looked quite promising. So initially, our strategy was to configure raw sockets in promiscuous mode to monitor all incoming packets/frames. However, we encountered a limitation that we couldn't do so with it.

After further looking into LWIP's source code however, We found out that it internally used emaclite drivers. Luckily, It had some documentation and examples on it's usage. [link](https://xilinx.github.io/embeddedsw.github.io/emaclite/doc/html/api/files.html)

So there were 2 ways to go further looking at examples:
1. Polling Mode
2. Interrupt Mode

We then decided to go with the Polling mode examples as DPDK does it and is highly performant, who cares about energy anyway. (Thanks to a previous course we did to know about that **one** point.)

Since we just need an example right now, the [ping reply example](https://github.com/Xilinx/embeddedsw/blob/master/XilinxProcessorIPLib/drivers/emaclite/examples/xemaclite_ping_reply_example.c) seemed to be the most appropriate option. Though this works,  that probably is the most blue pilled source code i've ever read, filled with brain rotted way of adding offsets to the pointer from current position. 

None the less, It's pretty simplistic and can be modified to view some further more details about the received packets as such:

![ethframe](/assets/img/other/smart-nic-fpga/ethframe.png)

This and an opposite example gave us a fair idea of how to proceed with the project so that we can now go and first implement reception, transmission and offloading/packet processing logic separately so it could be joined together.

## Communication between the FPGA and Hosts

### Uartlite

As said above, We decided to use Uartlite for sending and receiving data to/from the hosts while polling with the emaclite driver for the packets from the external world. It was important to realise a major skill issue of the MicroBlaze architecture, particularly its inability to support multithreading and the existing use of polling mode in other drivers such as EmacLite.

So our plan was to setup interrupts with uartlite drivers for transmission (from hosts) while reception of packet happens in polling.

However from the previously shown block diagram, we can see that interrupt's are not currently being handled and needs to be done so.
looking at the following yt video [link](https://www.youtube.com/watch?v=AlFKwqYTxtQ), we similarly added a `2:1` concat block to the design and regenerated everything like as in the getting started guide.

This is how the end result looks like with orange boxes indicating the new connections (I do not have a image of this stage stored, so ignore the censored part, you may scroll down for the uncensored version):

![block diagram](/assets/img/other/smart-nic-fpga/bdmid.jpeg)

Note: We've had an amazing discovery of a skill issue on our end to not read the driver manuals and code examples for both clearly indicating that the polling funcs here are non blocking!!! so this was all unnecessary at that situation, we could have just polled together but it was pretty late by then.

### Drivers??

We need drivers as well right, writing them in C in the very last week of the deadline while the endsems are going on. Ngl, brain wasn't braining, especially when we had to write drivers for the UART interface to act as a ethernet interface. Additionally Watching tuts like this [Youtube](https://www.youtube.com/watch?v=dSvhys-bppI),  we were probably more cooked than the crowdstrike employee who pushed that  code online without testing, He atleast probably had/will have his degree with him. 


Luckily, the most important objective of the project was the **1st** mentioned one and that required us to just get something working to show. So instead of writing drivers, We came up with the following:

### Linux and it's featuers

Well, All we needed is a ethernet interface on the OS through which an application can send and receive packets from and a way to put packets into it and take packets from it. 

So, we came up with a plan:

#### Linux Namespaces:

You want to test your NIC without rotting up your main network setup. Enter Linux namespaces, create mini, isolated containers within your main system. Think of it like setting up different profiles on your favorite social media app. Each namespace has its own network stack, so you can play around with transmission and reception separately without messing up your entire setup. It’s like having multiple sandboxes to test your toys in without breaking anything. Cool, right? *Thank chatgpt noises for the explanation*

#### Virtual Ethernet Interfaces

So this allows us to create a virtual ethernet interface, just like its name. I remember once renting a game CD as a 10y old with title `blank`, The cd was infact actually blank, Same situation. You don't actually need to have a physical ethernet interface to have this, It can be used as a real ethernet interface, This is how vpns work ig. *gotta add that ig everywhere*

#### scapy

This shi is dope. this can be used to sniff out packets during transmission and send packets during reception from/to an interface . 

#### how does this all go together?

![alt text](/assets/img/other/smart-nic-fpga/duomeme.png)

Well, using namespaces, we could just use a single laptop for testing things out. connect laptop and fpga to both UART(usb :3) and ethernet of the FPGA and work in different namespace to test things out easily. Hence, we no more needed our third guy who had another laptop acting as the external host. so we kicked him out. just kidding.

We create a virtual ethernet interface, say `veth0`. we configured it to use the same mac address as that of the FPGA's ethernet interface. 
We would sniff(listen) for packets on this `veth0` check if the source mac address of the packet is the interface's mac address. If it is, that means it was produced by the host and can be sent via the UART to FPGA. 

so, `FPGA's Mac address = veth0's Mac address`

While at reception, we would check for packets with destination mac address as that of the FPGA in the FPGA while polling, If it is, then we could process and send the packet to the host via UART and then insert it into the `veth0` or via another `veth1` which forwards the packet to `veth0`.

## Offloading / Packet processing / Firewall

This is something which we wanted to mainly do inside our NIC so that it would reduce the load on the hosts. As an example POC, we wanted to build a simple match action based firewall. we would see the IP address of the tcp packet and check if it is allowed. reject/drop if not. Our guide suggested us to use TCAM for making it more performant.

### TCAM - Ternary Content Addressable Memory
TCAM , designed to facilitate rapid data retrieval, operates on the principle of searching data by content rather than by address. This capability makes it ideally suited for networking tasks such as packet forwarding, where decision-making speed is critical.

The core consists of a memory array where each entry contains a data pattern (the key) and an associated mask. The mask specifies bits to be ignored in the search query, enabling the TCAM to execute flexible and complex matching operations. Operations are performed in parallel across all entries, drastically reducing the search time compared to sequential memory.

well, It was again the time for fanum taxing github and so we did. We found this particular repository 

## Packet Reception

![Packet Reception Diagram](/assets/img/other/smart-nic-fpga/det_reception.jpeg)

