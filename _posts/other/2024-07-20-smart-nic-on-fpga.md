---
title: Building A Smart Network Interface card on FPGA - Major Project edition
date: 2024-07-20 00:00:00 +530
categories: [Guides, Tutorials]
tags: [guide, linux, fpga, verilog, veriloghdl, namespaces, veth, microblaze, c, AXI] # TAG names should always be lowercase
---

[Project Github Link](https://github.com/Utkar5hM/fpga-based-packet-processing-unit)


Hey there folks, I'm writing here after a long aff time. Well, In this lore, I will be discussing about how we implemented our smart NIC  on a FPGA. This was my final year major project that I did with my fellow bros ([Chinamay](https://www.linkedin.com/in/chinmay-sharma-9a68b8200/) & [Muthukumar](https://www.linkedin.com/in/muthuku37/)). The idea for the project popped into our face like acne due to our exposure to xdp/ebpf without a sunscreen, we wanted to do some hardware offloading kind of a thing for the same but due to unforeseen situations, things did take a turn.

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

So and `hence`, this isn't a full proof guide to building an entirely working industry standard NIC allowing offloads thats gonna be fast aff or do performancemaxxing but about how one can get started to produce a mvp (minimum viable product, without the viable part) while knowing or having a lil bit experience with Verilog HDL, Linux(you'll know why later) and C to produce a NIC or whatever someone wants on a FPGA with mostly software oriented knowledge. We wanted to write this blog because we couldn't find a single goto place for such kinda things as everything was just  scattered in the ocean of the internet.

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

#### Scapy - A python library

This shi is dope. this can be used to sniff out packets during transmission and send packets during reception from/to an interface . 

#### how does this all go together?

![alt text](/assets/img/other/smart-nic-fpga/duomeme.png)

Well, using namespaces, we could just use a single laptop. connect laptop and fpga together via both UART(usb :3) and ethernet of the FPGA and work in multiple network namespaces to test things out easily. Hence, we no more needed our third guy with another laptop acting as the external host. so we kicked him out. just kidding.

We created a virtual ethernet interface caked `vethmp0`. we configured it to use the same mac address as that of the FPGA's ethernet interface. 

We were then able to sniff(listen) for packets on this `vethmp0`, check if the source mac address of the packet is the interface's mac address. If it is, that means it was produced by the host and can be sent via the UART to FPGA. 

so, `FPGA's Mac and IP address = vethmp0's Mac and IP address`

While at reception, we were able to check for packets with destination mac address as that of the FPGA in the FPGA while polling, If it is, then we could process and send the packet to the host via UART and then insert it into the `vethmp0` or via another `vethmp1` which forwards the packet to `vethmp0`.

#### configuring the development environment:

We created a bash script for simplifying the process, just run it once and we're done;
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

so our config would be like:

1. FPGA connected host's IP (connected to host via USB): `192.168.1.10`
2. Externally connected host's IP (via ethernet to FPGA): `192.168.1.11`
3. a third interface on the host for testing, with IP: `192.168.1.12`

each of them in a different namespace.

## Offloading / Packet processing / Firewall

This is something which we wanted to mainly do inside our NIC so that it would reduce the load on the hosts. As an example POC, we wanted to build a simple match action based firewall. we would see the IP address of the tcp packet and check if it is allowed. reject/drop if not. Our guide suggested us to use TCAM for making it more performant.

### TCAM - Ternary Content Addressable Memory
TCAM , designed to facilitate rapid data retrieval, operates on the principle of searching data by content rather than by address. This capability makes it ideally suited for networking tasks such as packet forwarding, where decision-making speed is critical.

The core consists of a memory array where each entry contains a data pattern (the key) and an associated mask. The mask specifies bits to be ignored in the search query, enabling the TCAM to execute flexible and complex matching operations. Operations are performed in parallel across all entries, drastically reducing the search time compared to sequential memory.

well, It was again the time for fanum taxing, this time a tcam IP but to no avail, all of them we found required a propriety license. 

Now we needed to develop our custom IP, this requires us to do the following:

1. Implement your own TCAM or find one online that's built with Verilog HDL or VHDL.
2. Turn it into an IP so that it is compatible to be added into our existing architecture ( via the block diagram page like other cores). This can be done by making it AXI compatible, which is by implementing an AXI wrapper around it.
3. Write microblaze C drivers for the same.

#### Verilog Implementation

So, hard working us, went ahead and searched for a verilog implementation on github and so we did found few like OpenTCAM but this particular repository [mcjtag/tcam](https://github.com/mcjtag/tcam), was exactly what we needed. This is for various reasons:

1. Easier to read code.
2. has a valid /reset signal, so we didn't need to go modify it's codebase to have those things unlike OpenTCAM as this would help us know when our output is ready.

#### Making it's vibes match with microblaze, Building an AXI - Wrapper

The AXI (Advanced eXtensible Interface) is a type of communication protocol designed by ARM, which provides a high-bandwidth, low-latency interface for interconnecting various components in an FPGA or a system-on-chip. In the context of the TCAM IP, the AXI interconnect acts as the bridge that allows the MicroBlaze processor to access and manipulate the TCAM entries. Through this interconnect, the processor can perform read and write operations on the TCAM’s memory locations, set up search keys, and initiate search operations.

Before diving into making one, we need to understand few more things.

##### AXI Master / Slave

1. Master is the controller, the one that initiates transactions. Think of it as the driver on the highway, deciding where to go and when.
2. Slave is the responder, the one that reacts to the master's commands. It's like the toll booth or service station, providing what the master requests.

In our context, We needed to implemented a slave interface because our MicroBlaze processor is acting as the master. The MicroBlaze initiated the transactions, and our TCAM (acting as the slave) responded to those commands.

##### AXI Lite vs. AXI Full vs. AXI Stream

1. AXI Lite: This is the basic version, perfect for simpler tasks. It’s got a straightforward address and data path, making it super easy to use but a bit limited in terms of bandwidth and performance.

2. AXI Full: This is the deluxe version, like Real Madrid with Mbappe. It’s designed for high-performance tasks and can handle complex data transfers with multiple channels for reading and writing simultaneously. It’s like having a high-speed, multi-lane highway compared to AXI Lite’s single-lane road.

3. AXI Stream: This version is all about continuous data transfer, without the need for addressing. It’s ideal for applications where data is flowing in a steady stream, like video or audio data.

We decided to proceed with AXI Lite as it looked quite simpler to work with and that's what a tutorial we followed had used.

### So How do we actually implement it?

Well, There are quite a decent number of resources for this. For one, Vivado Lets you generate a AXI wrapper which you can modify to add your own logic inside or you could write your own AXI wrapper entirely as suggested in [Buidilng an AXI-Lite slave the easy way
](https://zipcpu.com/blog/2020/03/08/easyaxil.html). This particular blog also claims that the pre existing axi wrapper template in Vivado has issues. I don't exactly remember the points but you can always click on the link to read more. But we decided to go for the first. There's also an video resource [Generating custom AXI4-Stream IP core using Xilinx Vivado](https://www.youtube.com/watch?v=chs5mdwMchQ) that explains implementing AXI interface from scratch.

In order to generate an AXI wrapper with vivado, One can follow many blogs and youtube videos available online like  [Integrating a custom AXI IP Core in Vivado for Xilinx Zynq ](https://www.youtube.com/watch?v=IDqzmcMR7p0), [Vivado Tutorial: Turn Verilog IP into AXI Module
](https://www.youtube.com/watch?v=mBRUK196qIA), [Microblaze RTL Simulation and AXI Slave wrapper tutorial](https://www.youtube.com/watch?v=EjnH-CgOp2g) and many more.

The following video [Custom Slave AXI LITE Interface for Microblaze with Xilinx Vitis P1](https://www.youtube.com/watch?v=BONXPKXyJTk) was the one that we followed and gives a basic understanding on instantiating your own custom verilog module into the AXI Wrapper.

while following the tutorial, we ended up creating an IP package with 6 slave registers as we weren't sure on how many were needed and also 32 bits data width as an IP(Internet Protocol) address is of length 32 bits which would be our input key.

our instantiation of the tcam verilog module looked something like this inside the `IPNAME_v1_0_S00_AXI.v` file:
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

Note: I just realized while writing the blog that I didn't set the parmater values which were required and hence `req_key` actually became 4 bits wide instead of 32. This deeply cooked us at that time and we couldn't really figure out why in the short time span. This will be discussed further down the blog.

well we also need to set value to the `set_*` and `req_*` inputs. For that, this particular part of the AXI does it for us by storing data into slave registers one by one in clock cycles:

![alt text](/assets/img/other/smart-nic-fpga/axiread.png)

we just extract this values and map it into the tcam as done above in instantiation.

We then configure new wires to extract out the output values from the tcam and set it in the following block of the wrapper:

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
The video linked above will give you a much clearer explanation on how things work.

so we end up with a TCAM IP that looks like this:


![Packet Reception Diagram](/assets/img/other/smart-nic-fpga/axitcam.jpeg)

Packaging it up and connecting it to the microblaze, regenerating stuff and running connection automation. we finally get this new block diagram, It's the last, I promise: **UNCENSORED, yey**

![block diagram final](/assets/img/other/smart-nic-fpga/bdf.jpeg)

### Writing drivers for the built TCAM-IP

We now need to be able to use it with C and run it with microblaze, we do that by writing drivers for the same. Having an AXI interface makes it a lot easier for us. It provides library to be able to read and set register values. This is what we need to do.

set register values such that it properly sets our required input variables to the TCAM and read values from registers in similar way.

Vivado generated AXI wrapper also creates a template driver for our use case which is located under `your_app_bsc/microblaze_or_your_proc/libsrc/IP_NAME_v1_0/src`.

One of the previously listed video [Microblaze RTL Simulation and AXI Slave wrapper tutorial](https://www.youtube.com/watch?v=EjnH-CgOp2g),  nicely explains on how to work with the created IP in Vivado SDK.

we defined several functions of our own:

1. `ECEMPTCAMIP_delay` to generate delay, pretty dumb way tbh. MB_SLEEP could also probably be used instead of this:
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

The obviously a test bench:

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

The above linked video and [Drivers for custom IP](https://www.youtube.com/watch?v=U-75MjbZyJE&t=383s) will give you more insights on the above code.

Following is a diagram on how the code is supposed to work:

![tcam tb flow Diagram](/assets/img/other/smart-nic-fpga/flow.png)

Running the tests, we were able to get the following results:

![tcam tb results](/assets/img/other/smart-nic-fpga/TCAM_IP.jpg)

Well, We got this TCAM part before endsems had started, Had a nice cold coffee at the nearby nescafe. went ahead and slept for the week not worrying about it. 

Later turned tides on the last night where it gave unexpected output for different IP addresses given as keys. We mostly tried to play around with increasing and decreasing delay, switching between our custom delay func and MB_Sleep and so does the diagram above has it. We just planned to drop fixing it as it was already 6 a.m. in the morning and our presentation was at 10 a.m. It probably was due to not having set the parameter values described above but I'm not sure so. I hope the above part of the blog at least serves as a base on how to implement an AXI wrapper and start working with it.


As rest part of this subsection is just working with C, We can skip it and see that the whole match action firewall could then be summarised to work like:

![firewall diagram](/assets/img/other/smart-nic-fpga/firewall.jpeg)

## Ethernet Frame Reception

>  Note: I won't be pasting the entire code files for this or the transmission section as they're pretty huge. Unless there's small script files that needs explanation. You can visit the github link of the project and find it there if needed.

The diagram below is honestly, pretty self explanatory but let's still explore the Ethernet Frame reception Implementation of the project in detail as I wanted to increase the total length of this blog.

![Frame Reception Diagram](/assets/img/other/smart-nic-fpga/det_reception.jpeg)

Since you already know that we're polling for the ethernet frames, We have to take certain decisions whenever the function catches one. Because packets can be of any type can be contained in them. We can't do match action against a non IPV4 /V6 type packet unless we also implement a MAC address based filtering. So what we do is:

1. Wait for the Frame.
2. Check if the ethertype of the ethernet frame is `IPv4` type shi, since we only built a firewall for IPV4. Here, EtherType is a two-octet field in an Ethernet frame. It is used to indicate which protocol is encapsulated in the payload of the frame and is used at the receiving end by the data link layer to determine how the payload is processed. (Source: [wikipedia](https://en.wikipedia.org/wiki/EtherType))
3. so now, if the frame is indeed of type `IPv4`, we extract the IP address and send it to our Firewall Logic, which does it's magic and sends us back a boolean value indicating if it should be dropped. If we have to drop. we just return and poll again.
4. if It's not a `IPv4`, we just skip the part and still proceed.
5. Now in order to send the frame to the host system, we first prepend the size of the frame padded to 2 bytes. So, later the system can know on how much data is to be expected. and also appended it with newline(`\r\n`) so we could just use `ser.readline()`. Only either would have been enough, but caffeine can kick humans into doing many things that they shouldn't, especially when . 
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

7. We have the following scapy based python program running which expects for data from the UART COM Port.

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

In order to test it, let's try pining from an external host(`192.168.1.10`) to the host/fpga(`192.168.1.11`). This won't actually work as we it won't be able to discover the host but we can atleast expect ARP request packets.

Testing it out and this works!!!!!! we can see the ARP packets
![reception output](/assets/img/other/smart-nic-fpga/receptest.jpeg)

## Ethernet Frame/Packet Transmission

Here we go again.

![Frame Transmission Diagram](/assets/img/other/smart-nic-fpga/Transmission.jpg)

Well it's pretty straight forward in transmission as we don't really need to do any kind of processing. The process can be broken down into the following:

1. Sniff for packets with scapy:

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

2. Check if the source Mac Address is same as that of the FPGA/veth0.
3. pad the frame to 1500 bytes will null bytes.
4. append length again.
5. send the packet through the UART serial COM port.

> All the above is done under scapy itself.

6. The interrupt is invoked and we check if the starting bytes of the received data starts from `\xFF` thing as seen in the diagram. which is what a Ethernet frame starts from.
7. **This is particularly important**: Check if reception of a packet is in progress to not interfere with it, Though there are different Rx and Tx buffers for UARTlite, the driver has only one function to reset it, doing so clears both of it and we wouldn't want to interfere.
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

For testing, We try pinging the external hosts(`eth interface`, `192.168.1.11`) from FPGA/Host(`192.168.1.10`). print through uart if we receive a frame in fpga. we sent it via emaclite as well, but I don't have screenshot with tcpdump as of right now. same ARP related situation applies here.

![transmission test](/assets/img/other/smart-nic-fpga/veth2uart.jpeg)

## Combining Everything together

Well we practically started enjoying. These components were separately present and working well, (though I've shown code here is from the combined). we kind of had a night left (yk who needs sleep). Join it together and it should be done, right?

We felt like this:

![winning son meme](/assets/img/other/smart-nic-fpga/winning.png)

After some time spent at night canteen, We actually started this work. Just add all the funcs together and boom.

To test this out, we tried pinging host(`192.168.1.11`) from our veth(`192.168.1.10`).

The output:

![final output](/assets/img/other/smart-nic-fpga/final.jpeg)

Looks like it's working, But it actually wasn't. This is what we got when we ran `ping`, instead of ICMP echos, all we had were repeating ARP packets. 

Upon inspecting with wireshark, we could clearly see that the packets sent from fpga to the host via UART were definitely malformed.
![final output](/assets/img/other/smart-nic-fpga/wireshark_err.jpeg)

After a lot of thoughts and deepfull discussions, considering that we needed to make a report as well as a presentation and that we had to give the FPGA back after it. We successfully **Rage Quitted**. 


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