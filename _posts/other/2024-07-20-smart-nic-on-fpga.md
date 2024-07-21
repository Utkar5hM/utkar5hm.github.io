---
title: Building A Smart Network Interface card on FPGA - Major Project edition
date: 2024-07-20 00:00:00 +530
categories: [Guides, Tutorials]
tags: [guide, linux, fpga, verilog, veriloghdl, namespaces, veth, microblaze, c, AXI] # TAG names should always be lowercase
---

[Project Github Link](https://github.com/Utkar5hM/fpga-based-packet-processing-unit)


Hey there, folks; I'm writing here after a long aff time. I will discuss how we implemented our smart NIC on an FPGA in this lore. This was my final year major project that I did with my fellow bros ([Chinamay](https://www.linkedin.com/in/chinmay-sharma-9a68b8200/) & [Muthukumar](https://www.linkedin.com/in/muthuku37/)). The idea for the project popped into our faces like acne due to our exposure to XDP/eBPF without sunscreen. We wanted to do some hardware offloading, but due to unforeseen situations, things took a turn.

The project aimed to do several things initially:

1. Get this shi done, get the degree and GTFO (I'm kidding, but I'm not.)
2. NIC designed entirely with Verilog, fanum taxing a pre-existing ethernet core like [liteeth](https://github.com/enjoy-digital/liteeth) from the GitHub where the data is serially processed and modified to our needs.
3. LAN on FPGA to be connected to the router with a standard interface such as PCIe or UART for communication between Host and FPGA(NIC).
4. Highly performant custom offloading logic to be built onto the NIC with Verilog.
5. Write NIC drivers in C for the host linux OS to support the built hardware.

This is how it would look like overall:

![architecture](/assets/img/other/smart-nic-fpga/arch.png)


If you haven't figured it out, the `major project edition` in the title here means something:

1. **We were COOKED**, I mean obviously.
2. We didn't do shit for the initial few months and barely had a week left for the **major** part of our MAJOR project.
3. We needed to clutch and take a lot of shortcuts.

So and `hence`, this isn't a proof guide to building an entirely working industry-standard NIC allowing offloads thats gonna be fast aff or do performancemaxxing but about how one can get started to produce an MVP (minimum viable product, without the viable part) while knowing or having a lil bit experience with Verilog HDL, Linux(you'll know why later) and C to produce a NIC or whatever someone wants on a FPGA with primarily software oriented knowledge. We wanted to write this blog because we couldn't find a single go-to place for such things as everything was scattered in the ocean of the internet.

Though we had good enough experience working with Verilog HDL in the past, We had never flashed and played with an FPGA before. Getting one from the university's pocket took a considerable amount of time. We ended up with **Nexys 4 DDR** because that was the one the university had and could be used with Vivado, as anyone who has ever used Quartus knows how much of a bad UI/UX it has (This matters to me :3).

But sadly, this limited us with a 100mbps link on the ethernet interface and only UART as a communication interface between a host and the FPGA. (ngl, we could've gotten a better one if we had tried harder, like just if we had asked for one )

Due to our extreme procrastination skills (I'm writing this blog several months after the project, so yk), We ended up changing our plans midway through the semester with barely a month left, not having done anything to the following:

1. Instead of building transmission and reception logic entirely in Verilog, We will flash a Microblaze soft Core processor onto the FPGA and write the packet transmission and reception logic with C.
2. Use custom or already built IPs (Intellectual properties) with the MicroBlaze for accelerating hardware offloading. Like a T-CAM IP (Ternary - Content Addressable Memory) for a firewall, allowing for faster IP address lookups with key-masks.
3. Most other things were supposed to be the same.

Now, we come to the actual implementation or how-to guide. In the first section, I will discuss how to start working with an FPGA (Nexys 4 DDR specifically), as that's what we've ever used.

## Getting Familiar with Nexys 4 DDR and Microblaze

If you've ever googled about anything regarding this project, You'd probably end up with this one specific guide [Nexys 4 DDR - Getting Started with Microblaze Servers](https://digilent.com/reference/learn/programmable-logic/tutorials/nexys-4-ddr-getting-started-with-microblaze-servers/start?redirect=1), Which discusses on how to flash a pre-existing C-based
TCP echo server application from the Xilinx SDK.

The echo server application is designed to set up a TCP server that echoes back any received text. It leverages the LWIP library, which provides a lightweight and streamlined TCP/IP stack facilitating high-level Ethernet functionality.

However, It is an ancient guide and probably won't work straightly if you tried it on the newer Vivado versions as they removed the MII to RMII core for the Nexys 4 DDR from the package, basically something that's required for our ethernet interfaces to work as expected. If you try to find it online, You won't be able to do it quickly; We just ended up installing Vivado 2019 for the same. At this point, I have uploaded it to the GitHub repository of this project. So you can find it there and add it to the newer version.

As most of the above guide is pretty self explanatory, lets not waste further time. If followed up right, we end up with the following block diagram:

![alt text](/assets/img/other/smart-nic-fpga/bd1.png)

After configuring further steps via the SDK, we also get a working TCP echo server.

![alt text](/assets/img/other/smart-nic-fpga/echosvr.png)

So, we now have at least something running.

## Capturing Raw Ethernet Frames

As previously stated, The echo server internally used LWIP, which looked promising. So, initially, our strategy was to configure raw sockets in promiscuous mode to monitor all incoming packets/frames. However, we encountered a limitation that we couldn't do with it.

After further looking into LWIP's source code, however, We found that it internally used emaclite drivers. Luckily, it had some documentation and examples of its usage. [link](https://xilinx.github.io/embeddedsw.github.io/emaclite/doc/html/api/files.html)

So there were two ways to go further looking at examples:
1. Polling Mode
2. Interrupt Mode

We then decided to go with the Polling mode examples as DPDK does it and is highly performant; who cares about energy anyway. (Thanks to a previous course, we knew about that **one** point.)

Since we need an example right now, the [ping reply example](https://github.com/Xilinx/embeddedsw/blob/master/XilinxProcessorIPLib/drivers/emaclite/examples/xemaclite_ping_reply_example.c) seemed to be the most appropriate option.

Though this works, it is the most blue-pilled source code I've ever read, filled with brain-rotted ways of adding offsets to the pointer from the current position.

Nonetheless, It's pretty simplistic and can be modified to view some further details about the received packets as such:

![ethframe](/assets/img/other/smart-nic-fpga/ethframe.png)

Above and an opposite example of sending frames gave us a fair idea of how to proceed with the project so that we can now go and first implement reception, transmission and offloading/packet processing logic separately so they could be done together.

## Communication between the FPGA and Hosts

### Uartlite

As said above, we decided to use Uartlite to send and receive data to/from the hosts while polling with the Emaclite driver for the packets from the external world. It was essential to realize a significant skill issue of the MicroBlaze architecture, particularly its inability to support multithreading and the existing use of polling mode in other drivers such as EmacLite.

So, we set up interrupts with Uartlite drivers for transmission (from hosts) while packet reception happens in polling.

However, from the previously shown block diagram, we can see that interrupts are not currently being handled and need to be done so.
Looking at the following yt video [link](https://www.youtube.com/watch?v=AlFKwqYTxtQ), we similarly added a `2:1` concat block to the design and regenerated everything like as in the getting started guide.

This is how the result looks like with orange boxes indicating the new connections (I do not have an image of this stage stored, so ignore the censored part; you may scroll down for the uncensored version):

![block diagram](/assets/img/other/smart-nic-fpga/bdmid.jpeg)

Note: We've had a fantastic discovery of a skill issue on our end to not read the driver manuals and code examples for both, indicating that the polling functions here are non-blocking!!! So this was all unnecessary in that situation. We could have just pulled together, but it was late.

### Drivers??

We need drivers as well, right? We are writing them in C during the very last week of the deadline while the end-semester exams are going on. Ngl, the brain wasn't braining, especially when we had to write drivers for the UART interface to act as an ethernet interface. Additionally, Watching tuts like this [Youtube](https://www.youtube.com/watch?v=dSvhys-bppI), we were probably more cooked than the Crowdstrike employee who pushed that code online without testing; he at least probably had/will have his degree with him.

Luckily, the most crucial objective of the project was the **1st** mentioned one, which required us to get something working to show. So, instead of writing drivers, We came up with the following:

### Linux and its features

We only needed an ethernet interface on the OS through which an application could send and receive packets. So a way to put packets into and read from.

So, we came up with a plan:

#### Linux Namespaces:

You want to test your NIC without rotting up your main network setup. Enter Linux namespaces and create mini, isolated containers within your host system. Think of it like setting up different profiles on your favourite social media app. Each namespace has its network stack, so you can play around with transmission and reception separately without messing up your entire setup. It's like having multiple sandboxes to test your toys in without breaking anything. Cool, right? *Thank ChatGPT noises for the explanation*

#### Virtual Ethernet Interfaces

So this allows us to create a virtual ethernet interface, just like its name. I remember once renting a game CD as a 10y old with the title `blank`; the CD was, in fact, actually blank; Same situation. You don't need a physical ethernet interface to create one, but It can be used to act like an actual ethernet interface; this is how VPNs work, ig. *gotta add that ig everywhere*

#### Scapy - A Python library

This shi is dope. This can be used to sniff out packets during transmission and send packets during reception from/to an interface.

#### How does this all go together?

![alt text](/assets/img/other/smart-nic-fpga/duomeme.png)

Well, using namespaces, we could use a single laptop. Connect laptop and FPGA via UART(USB:3) and ethernet of the FPGA and work in multiple network namespaces to test things out quickly. Hence, we no longer needed our third guy with another laptop as the external host. So we kicked him out. Just kidding.

We created a virtual ethernet interface baked `vethmp0`. We configured it to use the same Mac address as the FPGA's ethernet interface's Mac address.

We were then able to sniff(listen) for packets on this `vethmp0` and check if the source MAC address of the packet is the interface's MAC address. If it is, the host produced it and can be sent via the UART to FPGA.

So, `FPGA's Mac and IP address = vethmp0's Mac and IP address`

While at reception, we were able to check for packets with the destination MAC address as that of the FPGA in the FPGA while polling; if it is, then we could process and send the packet to the host via UART and then insert it into the `vethmp0` or via another `vethmp1` which forwards the packet to `vethmp0`.

#### configuring the development environment:

We created a bash script for simplifying the process; just run it once, and we're done;
```bash
ip netns add mp_nwk2
ip link add vethmp0 type veth peer name vethmp1
ip link set vethmp1 netns mp_nwk2
ip netns exec mp_nwk2 ip link set lo up
ip link set dev vethmp0 address 00:18:3E:01:EB:3A
ip link set vethmp0 up
ip address add 192.168.1.10/24 dev vethmp0
ip netns exec mp_nwk2 ip link set vethmp1 up
ip netns exec mp_nwk2 ip address add 192.168.1.12/24 dev vethmp1

ip netns add mp_nwk1
ip netns exec mp_nwk1 ip link set lo up
ip link set vethmp0 netns mp_nwk1
ip netns exec mp_nwk1 ip link set vethmp0 up
ip netns exec mp_nwk1 ip address add 192.168.1.10/24 dev vethmp0
ip netns exec mp_nwk1 python veth2uart.py
#term2
# ip netns exec mp_nwk2 bash
# ip netns exec mp_nwk2 ip link set vethmp1 up
# ip netns exec mp_nwk2 ip address add 192.168.1.12/24 dev vethmp1
```

So our config would be like:

1. FPGA connected host's IP (connected to the host via USB): `192.168.1.10`
2. Externally connected host's IP (via ethernet to FPGA): `192.168.1.11`
3. a third interface on the host for testing, with IP: `192.168.1.12`

Each above is present in a different namespace on the same host.

## Offloading / Packet processing / Firewall

Our main objective was to do this inside our NIC to reduce the load on the hosts. As an example POC, we wanted to build a simple match action-based firewall. We would see the IP address of the TCP packet and check if it is allowed. Reject/drop if not. Our guide suggested we use TCAM to make it more performant.

### TCAM - Ternary Content Addressable Memory
TCAM, designed to facilitate rapid data retrieval, searches data by content rather than address. This capability suits it for networking tasks such as packet forwarding, where decision-making speed is critical.

The core consists of a memory array where each entry contains a data pattern (the key) and an associated mask. The mask specifies bits to be ignored in the search query, enabling the TCAM to execute flexible and complex matching operations. Operations are performed in parallel across all entries, drastically reducing the search time compared to sequential memory.

Well, It was again the time for fanum taxing, this time a TCAM IP, but to no avail; all of them required a propriety license.

Now we needed to develop our custom IP, this requires us to do the following:

1. Implement your TCAM or find one online built with Verilog HDL or VHDL.
2. Turn it into an IP to be compatible to be added to our existing architecture ( via the block diagram page like other cores). This can be done by making it AXI compatible by implementing an AXI wrapper around it.
3. Write MicroBlaze C drivers for the same.

#### Verilog Implementation

So, hard-working, we went ahead and searched for a verilog implementation on GitHub, and we did find a few like OpenTCAM, but this particular repository [mcjtag/tcam](https://github.com/mcjtag/tcam) was exactly what we needed, for various reasons like:

1. Easier to read code.
2. has a valid /reset signal, so we didn't need to modify its codebase to have those things, unlike OpenTCAM, as this would help us know when our output is ready.

#### Making its vibes match with MicroBlaze, Building an AXI - Wrapper

The AXI (Advanced eXtensible Interface) is a type of communication protocol designed by ARM, which provides a high-bandwidth, low-latency interface for interconnecting various components in an FPGA or a system-on-chip. In the context of the TCAM IP, the AXI interconnect acts as the bridge that allows the MicroBlaze processor to access and manipulate the TCAM entries. Through this interconnect, the processor can perform read and write operations on the TCAM's memory locations, set up search keys, and initiate search operations.

Before diving into making one, we need to understand few more things.

##### AXI Master / Slave

1. The master is the controller, the one that initiates transactions. Think of it as the driver on the highway, deciding where to go and when.
2. The slave is the responder, the one that reacts to the master's commands. It's like the toll booth or service station, providing what the master requests.

In our context, we needed to implement a slave interface because our MicroBlaze processor acts as the master. The MicroBlaze initiated the transactions, and our TCAM (acting as the slave) responded to those commands.

##### AXI Lite vs. AXI Full vs. AXI Stream

1. AXI Lite: This is the basic version, perfect for simpler tasks. It has a straightforward address and data path, making it super easy to use but a bit limited in bandwidth and performance.

2. AXI Full: This is the deluxe version, like Real Madrid with Mbappe. It's designed for high-performance tasks and can simultaneously handle complex data transfers with multiple channels for reading and writing. It's like having a high-speed, multi-lane highway compared to AXI Lite's single-lane road.

3. AXI Stream: This version is all about continuous data transfer, without the need for addressing. It's ideal for applications where data flows in a steady stream, like video or audio data.

We decided to proceed with AXI Lite as it looked straightforward to work with and that's what a tutorial we followed had used.

### So, How do we implement it?

There are quite a few resources for this. For one, Vivado Lets you generate an AXI wrapper which you can modify to add your logic inside, or you could write our own AXI wrapper entirely as suggested in [Buidilng an AXI-Lite slave the easy way
](https://zipcpu.com/blog/2020/03/08/easyaxil.html). This blog also claims that the pre-existing AXI wrapper template in Vivado has issues. I don't remember the points, but you can always click the link to read more. But we decided to go for the first. A video resource [Generating custom AXI4-Stream IP core using Xilinx Vivado](https://www.youtube.com/watch?v=chs5mdwMchQ) explains implementing the AXI interface from scratch.

To generate an AXI wrapper with Vivado, One can follow many blogs and YouTube videos available online, like [Integrating a custom AXI IP Core in Vivado for Xilinx Zynq ](https://www.youtube.com/watch?v=IDqzmcMR7p0), [Vivado Tutorial: Turn Verilog IP into AXI Module
](https://www.youtube.com/watch?v=mBRUK196qIA), [Microblaze RTL Simulation and AXI Slave wrapper tutorial](https://www.youtube.com/watch?v=EjnH-CgOp2g) and many more.

The following video [Custom Slave AXI LITE Interface for Microblaze with Xilinx Vitis P1](https://www.youtube.com/watch?v=BONXPKXyJTk) was the one that we followed and gave a basic understanding of instantiating your own custom Verilog module into the AXI Wrapper.

While following the tutorial, we created an IP package with six slave registers as we weren't sure how many were needed. Also, we configured 32-bit as the data width for the wrapper as an IP(Internet Protocol) address, which is 32 bits in length, would be our input key.

Our instantiation of the TCAM verilog module looked something like this inside the `IPNAME_v1_0_S00_AXI.v` file:

```verilog
	module ecemptcamip_v1_0_S00_AXI #
	(
        // blablabla
		parameter integer C_S_AXI_DATA_WIDTH	= 32,
		// Width of S_AXI address bus
		parameter integer C_S_AXI_ADDR_WIDTH	= 5
	)
	(
        // blablabla;
    );
        // blablabla;
	tcam tcam_inst(.clk(S_AXI_ACLK),
        .rst(~S_AXI_ARESETN),
        .set_addr(slv_reg0[4:0]),
        .set_data(slv_reg0[9:5]),
        .set_key(slv_reg1),
        .set_xmask(slv_reg2),
        .set_clr(slv_reg0[10]),
        .set_valid(slv_reg0[11]),
        .req_key(slv_reg3),
        .req_valid(slv_reg0[12]),
        .req_ready(out_req_ready),
        .res_addr(out_res_addr),
        .res_data(out_res_data),
        .res_valid(out_res_valid),
        .res_null(out_res_null)
        );
    endmodule
```

Note: I just realized while writing the blog that I didn't set the required parameter values, and hence, `req_key` became 4 bits wider instead of 32. It deeply cooked us then, and we couldn't figure out why in such a short time. This issue is discussed further down the blog.

We must also set the value to the `set_*` and `req_*` inputs. For that, this particular part of the AXI does it for us by storing data into slave registers one by one in clock cycles:

![alt text](/assets/img/other/smart-nic-fpga/axiread.png)

We just extract these values and map them into the TCAM as done above in instantiation.

We then configure new wires to extract out the output values from the TCAM and set it in the following block of the wrapper:

```verilog
	// Implement memory mapped register select and read logic generation
	// Slave register read enable is asserted when valid address is available
	// and the slave is ready to accept the read address.
	assign slv_reg_rden = axi_arready & S_AXI_ARVALID & ~axi_rvalid;
	always @(*)
	begin
	      // Address decoding for reading registers
	      case ( axi_araddr[ADDR_LSB+OPT_MEM_ADDR_BITS:ADDR_LSB] )
	        3'h0   : reg_data_out <= slv_reg0;
	        3'h1   : reg_data_out <= slv_reg1;
	        3'h2   : reg_data_out <= slv_reg2;
	        3'h3   : reg_data_out <= slv_reg3;
	        3'h4   : reg_data_out <= slv_reg4;
	        3'h5   : begin
	           reg_data_out[4:0] <= out_res_addr;
	           reg_data_out[9:5] <= out_res_data;
	           reg_data_out[10] <= out_req_ready;
	           reg_data_out[11] <= out_res_valid;
	           reg_data_out[12] <= out_res_null;
	        end
	        default : reg_data_out <= 0;
	      endcase
	end
```
The video linked above will give you a clearer explanation of how things work.

So we end up with a TCAM IP that looks like this:


![Packet Reception Diagram](/assets/img/other/smart-nic-fpga/axitcam.jpeg)

Packaging it up and connecting it to the MicroBlaze, regenerating stuff and running connection automation. We finally get this new block diagram; It's the last, I promise: **UNCENSORED, yey**

![block diagram final](/assets/img/other/smart-nic-fpga/bdf.jpeg)

### Writing drivers for the built TCAM-IP

We now need to be able to use it with C and run it with Microblaze; we do that by writing drivers for the same. Having an AXI interface makes it a lot easier for us. It provides library to be able to read and set register values.

We needed to do the following:

Set register values such that it properly sets our required input variables to the TCAM and reads values from registers in a similar way.

Vivado-generated AXI wrapper also creates a template driver for our use case, which is located under `your_app_bsc/microblaze_or_your_proc/libsrc/IP_NAME_v1_0/src`.

One of the previously listed videos [Microblaze RTL Simulation and AXI Slave wrapper tutorial](https://www.youtube.com/watch?v=EjnH-CgOp2g) nicely explains how to work with the created IP in Vivado SDK.

We defined several functions of our own:

1. `ECEMPTCAMIP_delay` to generate delay is a pretty dumb way, to be honest. MB_SLEEP could also probably be used instead of this:

```c
void ECEMPTCAMIP_delay(u64 d1){
	for(u64 i=0; i<d1; i++){
		for(u64 j=0;j<9999999999999999;)
			j=j+1;
	    }
}
```

2. `ECEMPTCAMIP_SetKey` to give inputs:

```c
void ECEMPTCAMIP_SetKey(u32 BaseAddress, u8 Address, u8 Data, u32 Key, u32 KeyMask){

	// Set Key
	ECEMPTCAMIP_mWriteReg(BaseAddress, 4*1, Key);

	// Set Key Mask
	ECEMPTCAMIP_mWriteReg(BaseAddress, 4*2, KeyMask);

	//	Set Address[4:0], data[9:5], set_clr[10]=0,set_valid[11]=1, req_valid[12]=0
	u32 RegData =0;
	// Mask Address and Data to use only lower 5 bits
	Address &= 0x1F; // 0x1F = 0001 1111 in binary, ensuring only lower 5 bits are used
	Data &= 0x1F;

	// Shift Address and Data to their correct positions and set in RegData
	RegData |= (Address << 0);  // Address is at bits 4:0
	RegData |= (Data << 5);     // Data is at bits 9:5

	// Set set_valid to 1 at bit 11
	RegData |= (1 << 11);       // Set bit 11
	ECEMPTCAMIP_mWriteReg(BaseAddress, 0, RegData);
	ECEMPTCAMIP_delay(999999);
	ECEMPTCAMIP_delay(999999);
	// Set set_valid back to 0
	RegData &= ~(1 << 11);
	ECEMPTCAMIP_mWriteReg(BaseAddress, 0, RegData);
	ECEMPTCAMIP_delay(999999);
	ECEMPTCAMIP_delay(999999);
	return;
}
```

3. `ECEMPTCAMIP_GetKey` to read input

```c
u32 ECEMPTCAMIP_GetKey(u32 BaseAddress, u32 Key){
	// set Req_key
	ECEMPTCAMIP_mWriteReg(BaseAddress, 4*3, Key);

	// Set_valid to false and other stuff as well.
	u32 RegData =0;
	// Set req_valid to 1 at bit 12
	RegData |= (1 << 12);       // Set bit 11
	ECEMPTCAMIP_mWriteReg(BaseAddress, 0, RegData);
	u32 Response = 0;
	u8 ResponseReady=0;
	u32 ResponseCheckCount=0;
	while(ResponseReady==0){
		ECEMPTCAMIP_delay(999999);
		Response = ECEMPTCAMIP_mReadReg(BaseAddress, 4*5);
	    // Extracting out_req_ready from Response[10]
	    ResponseReady = (u8)((Response >> 10) & 0x01) || (ECEMPTCAMIP_GetRespValid(Response) &( !ECEMPTCAMIP_GetRespNull(Response)));  // Only need 1 bit

	    ResponseCheckCount++;
	    if(ResponseCheckCount>1000) {

	    	Response = 1U << 12;
	    	break;
	    }
	}
	RegData &= ~(1 << 12);
	ECEMPTCAMIP_mWriteReg(BaseAddress, 0, RegData);
	ECEMPTCAMIP_delay(999999);
	return Response;
}
```

4. various functions to extract required data from obtained register values:

```c
u8 ECEMPTCAMIP_GetRespAddr(u32 Response) {
	return ((u8)(Response & 0x1F));
}

u8 ECEMPTCAMIP_GetRespData(u32 Response) {
	return ((u8)((Response >> 5) & 0x1F));
}

u8 ECEMPTCAMIP_GetRespValid(u32 Response) {
	return  (u8)((Response >> 11) & 0x01);
}

u8 ECEMPTCAMIP_GetRespNull(u32 Response) {
	return (u8)((Response >> 12) & 0x01);
}
```

Obviously a test bench:

```c
#include <stdio.h>
#include "platform.h"
#include "xil_printf.h"
#include "ecemptcamip.h"


int main()
{
    init_platform();
    xil_printf("Major Project TCAM-IP\n\r");
    xil_printf("--- Input 1 ---\n\r");
    u8 Address =0;
    u8 Data=2;
    u32 Key=73;
    u32 KeyMask=0;
    KeyMask |= 0xFF;
    xil_printf("Address %u\n\r", Address);
    xil_printf("Data %u\n\r", Data);
    xil_printf("Key %u\n\r", Key);
    xil_printf("KeyMask %u\n\r", KeyMask);
    ECEMPTCAMIP_SetKey(XPAR_ECEMPTCAMIP_0_S00_AXI_BASEADDR, Address, Data, Key, KeyMask);
    ECEMPTCAMIP_delay(999999);
    xil_printf("--- Input 2 ---\n\r");
    Address =1;
    Data=5;
    Key=4000;
    KeyMask=0;
    xil_printf("Address %u\n\r", Address);
    xil_printf("Data %u\n\r", Data);
    xil_printf("Key %u\n\r", Key);
	xil_printf("KeyMask %u\n\r", KeyMask);
	ECEMPTCAMIP_SetKey(XPAR_ECEMPTCAMIP_0_S00_AXI_BASEADDR, Address, Data, Key, KeyMask);
	ECEMPTCAMIP_delay(999999);
    xil_printf("--- Input 3 ---\n\r");
    Address =2;
    Data=4;
    Key=2502;
    KeyMask=0;
    xil_printf("Address %u\n\r", Address);
    xil_printf("Data %u\n\r", Data);
    xil_printf("Key %u\n\r", Key);
	xil_printf("KeyMask %u\n\r", KeyMask);
	ECEMPTCAMIP_SetKey(XPAR_ECEMPTCAMIP_0_S00_AXI_BASEADDR, Address, Data, Key, KeyMask);
	ECEMPTCAMIP_delay(999999);
    xil_printf("--- Reading Output 1 ---\n\r");
    Key=4000;
    xil_printf("Input Key : %lu\n\r", Key);
    u32 Response = ECEMPTCAMIP_GetKey(XPAR_ECEMPTCAMIP_0_S00_AXI_BASEADDR, Key);
    xil_printf("Data: %u\n\r", ECEMPTCAMIP_GetRespData(Response));
    xil_printf("Address: %u\n\r", ECEMPTCAMIP_GetRespAddr(Response));
    xil_printf("Response Valid?: %u\n\r", ECEMPTCAMIP_GetRespValid(Response));
    xil_printf("Response Null: %u\n\r", ECEMPTCAMIP_GetRespNull(Response));
	ECEMPTCAMIP_delay(999999);
    xil_printf("--- Reading Output 2 ---\n\r");
    Key=2503;
    xil_printf("Input Key : %lu\n\r", Key);
    Response = ECEMPTCAMIP_GetKey(XPAR_ECEMPTCAMIP_0_S00_AXI_BASEADDR, Key);
    xil_printf("Data: %u\n\r", ECEMPTCAMIP_GetRespData(Response));
    xil_printf("Address: %u\n\r", ECEMPTCAMIP_GetRespAddr(Response));
    xil_printf("Response Valid?: %u\n\r", ECEMPTCAMIP_GetRespValid(Response));
    xil_printf("Response Null: %u\n\r", ECEMPTCAMIP_GetRespNull(Response));
	ECEMPTCAMIP_delay(999999);
    xil_printf("--- Reading Output 3 ---\n\r");
    Key=62;
    xil_printf("Input Key : %lu\n\r", Key);
    Response = ECEMPTCAMIP_GetKey(XPAR_ECEMPTCAMIP_0_S00_AXI_BASEADDR, Key);
    xil_printf("Data: %u\n\r", ECEMPTCAMIP_GetRespData(Response));
    xil_printf("Address: %u\n\r", ECEMPTCAMIP_GetRespAddr(Response));
    xil_printf("Response Valid?: %u\n\r", ECEMPTCAMIP_GetRespValid(Response));
    xil_printf("Response Null: %u\n\r", ECEMPTCAMIP_GetRespNull(Response));
    cleanup_platform();
    return 0;
}
```

The above-linked video and [Drivers for custom IP](https://www.youtube.com/watch?v=U-75MjbZyJE&t=383s) will give you more insights on the above code.

Following is a diagram of how the code is supposed to work:

![tcam tb flow Diagram](/assets/img/other/smart-nic-fpga/flow.png)

Running the tests, we were able to get the following results:

![tcam tb results](/assets/img/other/smart-nic-fpga/TCAM_IP.jpg)

Well, we got this TCAM part before the endsems started. We Had an excellent cold coffee at the nearby Nescafe. We went ahead and slept for the week without worrying about it.

Later, tides turned on the last night, giving unexpected output for different IP addresses given as keys. As the diagram above shows, we mostly tried to play around with increasing and decreasing delay, switching between our custom delay function and `MB_Sleep()`. We planned to drop fixing it as it was already 6 a.m., and our presentation was at 10 a.m. It probably was due to not having set the parameter values described above but I'm not sure so. I hope the above part of the blog serves as a base for implementing an AXI wrapper and starting working with it.


As the rest part of this subsection is just working with C, We can skip it and see that the whole match action firewall could then be summarized to work like this:

![firewall diagram](/assets/img/other/smart-nic-fpga/firewall.jpeg)

## Ethernet Frame Reception

> Note: I will only be pasting some part of the code for this or the transmission section, as most of it is huge. You can visit the project's GitHub link and find it there if needed.

The diagram below is self-explanatory, but let's still explore the Ethernet Frame reception Implementation of the project in detail, as I wanted to increase the total length of this blog.

![Frame Reception Diagram](/assets/img/other/smart-nic-fpga/det_reception.jpeg)

Since you already know we're polling for the ethernet frames, We have to make certain decisions whenever the function catches one. Because packets of any type can be contained in them, we can't do match action against a non-IPV4/V6 type packet unless we also implement a MAC address-based filtering. So what we do is:

1. Poll for the ethernet frame.
2. Once received, check if the Ethertype of the ethernet frame is `IPv4` type shi since we only built a firewall for IPV4. Here, EtherType is a two-octet field in an Ethernet frame. It indicates which protocol is encapsulated in the frame's payload and is used by the data link layer at the receiving end to determine how the payload is processed. (Source: [wikipedia](https://en.wikipedia.org/wiki/EtherType))
3. so now, if the frame is indeed of type `IPv4`, we extract the IP address and send it to our Firewall Logic, which does its magic and sends us back a boolean value indicating if it should be dropped. If we have to drop. We just return and poll again.
4. If it's not an `IPv4`, we skip the part and proceed.
5. to send the frame to the host system, we first prepend the frame size, which is padded to 2 bytes. So, later, the system can know how much data is to be expected. and also appended it with newline(`\r\n`) so we could just use `ser.readline()`. Only either would have been enough, but caffeine can kick humans into doing many things that they shouldn't; I'm jk; you'll realize why later.
6. we send the data through UART with uartlite drivers. Ignore most of the initial part.

```c
	while(1){
			if(FrameCaptured > 0){
                // Ignore this part for now, It's for Transmission
				XEmacLite_Send(&EmacLiteInstance,
						(u8 *)&TxFrame,
						FrameLength);
				MB_Sleep(10);
				FrameCaptured=0;
				FrameLength=0;
			} else {
				while(TotalSentCount<RxFrameBufferLength){
				}
				TotalSentCount=0;
				RxFrameBufferLength=0;
				while (RecvFrameLength == 0) {
					RecvFrameLength = XEmacLite_Recv(EmacLiteInstPtr,
									 (u8 *)RxFrame); // part that matters, starts here.
				}
				u8 Response = ProcessRecvFrame((u16 *)RxFrame); // YO your firewall goes here
				if(Response==1){
					u8 RxFrameBuffer[1600];
					memset(RxFrameBuffer, 0, sizeof(RxFrameBuffer));
					memcpy(RxFrameBuffer, RxFrame, RecvFrameLength);
					u16 length = Xil_Htons(RecvFrameLength);  // Convert to network byte order
					memcpy(&RxFrameBuffer[1600 - 4], (u8*)&length, 2);
					RxFrameBuffer[1600-2]='\r';
					RxFrameBuffer[1600-1]='\n';
					RecvFrameLength=0;
					RxFrameBufferLength=1600;
					memset(RxFrame, 0, sizeof(RxFrame));
					XUartLite_Send(&UartLite, (u8*)RxFrameBuffer, RxFrameBufferLength);
				}
			}
	    }
```

7. The following Scapy based Python program is run and expects data from the UART COM Port.

```py
import serial
from scapy.all import Ether, sendp, UDP, IP , srp
from time import sleep

ser = serial.Serial('/dev/ttyUSB1')  # replace with your serial port
frame = b""
capturing = False
while True:
    try:
        data = ser.readline()[:-2]
        framelen = int.from_bytes(data[-2:], 'big')
        # if(framelen < 1520): idr whats this doing here
        frame = Ether(bytes(Ether(data[:framelen])))
        print(framelen)
        frame.show()
        sendp(frame, iface="vethmp0")
        # srp(frame, iface='vethmp0')
    except Exception as e:
        continue
```

To test it, we can try pinging from an external host(`192.168.1.10`) to the host/FPGA(`192.168.1.11`). This won't work as the external host won't be able to discover our host, but we can at least expect ARP request packets.

By testing it out, we can see that it works!!!!!! we can see the ARP packets :3 

![reception output](/assets/img/other/smart-nic-fpga/receptest.jpeg)

## Ethernet Frame/Packet Transmission

Here we go again.

![Frame Transmission Diagram](/assets/img/other/smart-nic-fpga/Transmission.jpg)

Well, it's pretty straightforward in transmission, as we don't need any processing.

The process for transmission is broken down into the following:

1. Sniff for packets with Scapy:

```py
from scapy.all import sniff, Ether
import serial

ser = serial.Serial('/dev/ttyUSB1') 
def handle_frame(frame):
    # This function will be called for each captured frame
    # Here you can analyze or forward the frame as needed
    #print source and destination mac address
    if((frame.src).lower() == "00:18:3e:01:eb:3a"):
        print(frame.summary())
        print("Source MAC: " + frame.src)
        try:
            tosend = bytes(frame)
            original_length = len(tosend)
            tosend = tosend.ljust(1518, b'\0')  # pad with null bytes to 1500 bytes
            tosend += original_length.to_bytes(2, 'big')  # append length at 1501st byte
            ser.write(tosend)
        except Exception as e:
            print("Error writing to serial port")
            print(e)

sniff(iface="vethmp0", prn=handle_frame, store=False, lfilter=lambda x: x.haslayer(Ether))
```

2. Check if the source Mac Address is the same as the FPGA/`vethmp0`.
3. pad the frame to 1500 bytes will null bytes.
4. append length again.
5. send the packet through the UART serial COM port.

> All the above is done under scapy itself.

6. The interrupt is invoked, and we check if the starting bytes of the received data start from the `\xFF` thing, as seen in the diagram, which is what an Ethernet frame begins with.
7. **This is particularly important**: Check if the reception of a packet is in progress so as not to interfere with it. Though there are different Rx and Tx buffers for UARTlite, the driver has only one function to reset it, so using them clears both of them, and we wouldn't want to interfere.
8. The data is processed, frame is extracted and then sent through the ethernet interface through the emaclite drivers.

```c
void RecvHandler(void *CallBackRef, unsigned int EventData)
{
	if(EventData==TX_BUFFER_SIZE&&
			TxBuffer[0]==0xFF&&
			TxBuffer[1]==0xFF&&
			TxBuffer[2]==0xFF&&
			TxBuffer[3]==0xFF&&
			TxBuffer[4]==0xFF&&
			TxBuffer[5]==0xFF
		){
		FrameLength = ((u16)TxBuffer[TX_BUFFER_SIZE -2 ] << 8) | TxBuffer[TX_BUFFER_SIZE - 1];
		if(FrameLength<XEL_MAX_FRAME_SIZE){
			memcpy(TxFrame, TxBuffer, FrameLength);
			FrameCaptured=1;
			XUartLite_Recv(&UartLite, TxBuffer, TX_BUFFER_SIZE);
            // Rest is handled in the code shown above in Reception section.xD
		}
	} else {
        // invalid data, reset fifo and begin again?
		XUartLite_ResetFifos(&UartLite);
		XUartLite_Recv(&UartLite, TxBuffer, TX_BUFFER_SIZE); 
	}
}
```

For testing, We try pinging the external hosts(`eth interface`, `192.168.1.11`) from FPGA/Host(`192.168.1.10`). Print through UART if we receive a frame in FPGA. We sent it with Emaclite driver as well, but I don't have its screenshot with tcpdump right now. same ARP related situation applies here.

![transmission test](/assets/img/other/smart-nic-fpga/veth2uart.jpeg)

## Combining everything

Well we practically started enjoying. These components were separately present and working well, (though I've shown code here is from the combined). We had a night left (yk, who needs sleep). Join it together and it should be done, right?

We felt like this:

![winning son meme](/assets/img/other/smart-nic-fpga/winning.png)

After some time spent at the night canteen, We started this work. As planned, add all the functions together and boom.

We tried pinging host(`192.168.1.11`) from our veth(`192.168.1.10`) to test this out.

The output:

![final output](/assets/img/other/smart-nic-fpga/final.jpeg)

It looks like it's working, But it wasn't. That output is what we got when we ran `ping`, instead of ICMP echos, where all we had was repeating ARP packets.

Upon inspecting with Wireshark, we could clearly see that the packets sent from FPGA to the host via UART were malformed.
![final output](/assets/img/other/smart-nic-fpga/wireshark_err.jpeg)

After a lot of thought and deep discussions, considering that we needed to make a report and a presentation and that we had to return the FPGA back to the university after all. We successfully **Rage Quitted**.


-------

We kinda followed or have seen a lot of resources, lost most of them in the process. But here are some if not already mentioned above, in no particular order:

1. NITK's CSE Dept Open Source Networking Technology Course Notes for working with veth's and namespaces.
2. https://digilent.com/reference/learn/programmable-logic/tutorials/nexys-4-ddr-getting-started-with-microblaze-servers/start?redirect=1
3. https://forum.digilent.com/topic/4753-multiple-uartlite-instantiation-w-microblaze/
4. https://github.com/NetFPGA/NetFPGA-SUME-public/wiki/NetFPGA-SUME-TCAM-IPs
5. https://xilinx.github.io/embeddedsw.github.io/emaclite/doc/html/api/example.html
6. https://xilinx.github.io/embeddedsw.github.io/emaclite/doc/html/api/index.html
7. https://github.com/rprinz08/hBPF
8. https://people.ece.cornell.edu/land/courses/ece5760/FinalProjects/f2011/mis47_ayg6/mis47_ayg6/index.html
9. https://gist.github.com/austinmarton/2862515
10. https://support.xilinx.com/s/question/0D54U00005WcW1LSAV/etherner-raw-data-sniffing?language=en_US
11. https://support.xilinx.com/s/question/0D52E00006hpiOxSAI/anyone-tried-using-lwip-to-send-raw-ethernet-frames-with-custom-ethertype?language=en_US
12. https://www.youtube.com/watch?v=pkDWzG8spvg&t=1145s
13. https://docs.amd.com/r/2.2-English/pg318-tcam/Introduction
14. https://github.com/dchammond/eth_debug