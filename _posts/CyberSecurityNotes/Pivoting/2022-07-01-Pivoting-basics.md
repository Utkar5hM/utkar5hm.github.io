---
title: Pivoting basics - Notes
date: 2022-07-01 00:00:00 +530
categories: [ Cyber Security Notes, Pivoting]
tags: [Notes, Pentesting, Pivoting, Firewall, Proxy, Port Forward, Red Team]     # TAG names should always be lowercase
---

--------------
network penetration testing, and is one of the three main teaching points for this room.

Put simply, by using one of the techniques described in the following tasks (or others!), it becomes possible for an attacker to gain initial access to a remote network, and use it to access other machines in the network that would not otherwise be accessible:

![screenshot](/assets/img/notes/pivoting/img2.png)

--------------

the following Bash one-liner would perform a full ping sweep of the 192.168.1.x network:

`for i in {1..255}; do (ping -c 1 192.168.1.${i} | grep "bytes from" &); done`

The above command generates a full list of numbers from 1 to 255 and loops through it. For each number, it sends one ICMP ping packet to 192.168.1.x as a backgrounded job (meaning that each ping runs in parallel for speed), where i is the current number. Each response is searched for "bytes from" to see if the ping was successful. Only successful responses are shown.

--------------

The equivalent of this command in Powershell is unbearably slow, so it's better to find an alternative option where possible.(very simple beta examples can be found [here](https://github.com/MuirlandOracle/C-Sharp-Port-Scan) for C#, or [here](https://github.com/MuirlandOracle/CPP-Port-Scanner) for C++).

--------------

If you suspect that a host is active but is blocking ICMP ping requests, you could also check some common ports using a tool like netcat.

Port scanning in bash can be done (ideally) entirely natively:

`for i in {1..65535}; do (echo > /dev/tcp/192.168.1.1/$i) >/dev/null 2>&1 && echo $i is open; done`

--------------

NOTE: To start up a TCPDump listener we would use the following command:  
`tcpdump -i tun0 icmp`

--------------

### opening ports Windows
When using this option you will need to open up a port in the Windows firewall to allow the forward connection to be made. The syntax for opening a port using `netsh` looks something like this:  
`netsh advfirewall firewall add rule name="NAME" dir=in action=allow protocol=tcp localport=PORT`

