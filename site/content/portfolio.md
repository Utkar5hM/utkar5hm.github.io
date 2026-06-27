---
title: Portfolio
author: Utkarsh M
---

## Experience

### Applications Developer I · Oracle _(June 2024 - Present)_
- Cut manual document validation by **5-6K hours per year** by designing and deploying an AI-powered document validation workflow on OCI, automating **60%** of routine validations end-to-end.
- Eliminated manual ML deployment hand-offs by building an MLOps pipeline on OCI Data Science with Kafka and Kubernetes, automating model promotion from training to production.
- Built an A2A Gateway governing Fusion AI agent access from enterprise chat through rubric-based tiering, per-user RBAC, and audit logging.
- Implemented Oracle User Assertion (JWT-bearer) grant in an agent-router service to invoke Fusion AI agents as the authenticated end user while preserving per-user RBAC.
- Accelerated analyst workflows by building a summarization tool using LLaMA + RAG, reducing SME dependency for document-heavy processes.
- Accelerated document-validation test generation by developing an MCP server that dynamically creates test documents from dashboard data and user inputs.
- Reduced per-team integration overhead by shipping a reusable MCP server Docker base image that proxies services, handles OAuth 2.0 client-credentials auth, and self-registers with the MCP registry.
- Received **5x Rock Star Gold Awards** for high-impact delivery and contributions across the OAL/SaaS organization.

### Applications Developer Intern · Oracle _(May 2023 - Jul 2023, Remote)_
- Developed an internal HR chatbot for the Oracle HCM platform by designing conversational workflows and implementing custom backend logic in asynchronous Node.js, improving employee access to HR services.
- Implemented error handling and session management to ensure reliable, fault-tolerant user interactions.

### System Engineer · IRIS, NITK _(Jul 2022 - Apr 2024)_
- Secured various services of the student-run IRIS LMS (23K+ users) by implementing key production security measures and hardening system configurations.
- Streamlined deployments by automating builds with GitLab CI/CD and containerizing services to improve consistency and reduce manual errors.
- Refactored the Staging Server, implementing secure Git deployments and an in-browser terminal, while resolving core architectural vulnerabilities.
- Prototyped password-less SSH with HashiCorp Vault and Keycloak for role-based access control.
- Conducted distributed load testing with Locust and Grafana to ensure availability during peak usage such as examinations.
- Automated SSL certificate synchronization and configured data backups to ensure availability and disaster recovery.

### B.Tech, Electronics and Communication · NIT Karnataka, Surathkal _(2020 - 2024)_
- Major project: FPGA-based Smart NIC offloading packet processing with a custom TCAM-IP core and match-action firewall.
- Presented sessions on systems and security topics as part of IEEE-NITK and Web Enthusiasts' Club, NITK.
- Contributed to IRIS-NITK as a System Engineer, securing and maintaining the student LMS for 23K+ users.

## Skills

**Languages:** C/C++, TypeScript, JavaScript, Python, Go, Verilog, Assembly (ARM, x86\_64)

**Frameworks & Technologies:** React, React Native, Node.js, Next.js, Django, Docker, Kubernetes, PostgreSQL, Redis, Linux, OCI, eBPF

**Security:** Web and Network Penetration Testing, Reverse Engineering

**Hardware:** FPGA, Embedded C

## Projects

### Kneed - Knee Recovery Companion App _(2025 - Present)_
A mobile + cloud platform to guide knee rehabilitation with daily tracking, analytics, and AI-powered coaching.
- Built a Node.js/Express REST API with OAuth 2.1, JWT sessions, and Apple Sign-In; secured with rate limiting, CORS, and Helmet.js.
- Designed a React Native (Expo) mobile app with 2D anatomical knee visualization using Three.js for intuitive symptom logging.
- Integrated Apple HealthKit to sync steps, gait speed, sleep, and workouts, enabling data-driven recovery insights.
- Implemented a phased exercise routine engine, pain trend analytics, and medication adherence tracking.
- Shipped an MCP server enabling Claude AI to fully manage recovery programs conversationally: setting up workouts, daily routines, and medications, and analyzing progress, with no manual configuration required.

**Tech:** Node.js, Express, TypeScript, React Native, Expo, SQLite, Docker, Caddy, Apple HealthKit, OAuth 2.1

---

### Clapped - Curated Learning Lists Platform _(2025)_
A full-stack web app for creating, sharing, and progressing through curated video and note-based learning lists.
- Architected SSR-first full-stack app with TanStack Start, file-based routing, and Nitro server runtime for fast page loads.
- Built AI-powered note generation pipeline with Graphile Worker job queue, YouTube transcript extraction, and partial-result handling for resilient async processing.
- Implemented fractional indexing for conflict-free drag-and-drop item reordering at scale.
- Delivered a rich collaborative editor using Tiptap with markdown, code blocks, math rendering, and Excalidraw diagram embedding.
- Designed a social discovery system with public profiles, bookmarking, OG image generation, ratings, and activity visibility controls.

**Tech:** React 19, TanStack Start, TypeScript, PostgreSQL, Drizzle ORM, Tailwind CSS v4, shadcn/ui, Tiptap, Graphile Worker, Docker

[Live](https://clapped.space/)

---

### Total Overdose: RTX Remix _(2025)_
Reverse-engineered the 2004 Direct3D 9 game *Total Overdose* to run under NVIDIA RTX Remix path tracing.
- Authored a C++ D3D9 vtable hook (ASI shim) to resend camera transforms, redirect render targets, and shadow managed textures for valid Remix texture hashes.
- Reverse-engineered game binary with Ghidra to locate skybox draw markers and HUD render calls, enabling correct sky/HUD separation in the path tracer.
- Resolved fog, atmosphere, and lighting artifacts by tuning RTX Remix config and implementing an optional injected sun light with auto-aim.
- Produced AI-upscaled PBR texture mod (diffuse/normal/roughness) for Mission 2.

**Tech:** C++, Direct3D 9, RTX Remix, Ghidra, DXVK, ASI Loader

[GitHub](https://github.com/Utkar5hM/TotalOverDoseRTXRemix)

---

### Execstasy - IAM for Linux Instances _(May 2025 - June 2025)_
A modern Identity & Access Management platform for Linux servers.
- Developed a robust Golang REST API for role-based access control (RBAC).
- Engineered OAuth 2.0 Device Authorization Flow (RFC 8628) from scratch, integrating Redis and Postgres.
- Built a Linux PAM module in C for seamless API-driven authentication.
- Designed a sleek React/Next.js interface using shadcn UI components.
- Dockerized microservices for scalable deployment.

**Tech:** Go, C, PostgreSQL, Redis, Linux, Docker, React.js, Next.js

[GitHub](https://github.com/Utkar5hM/Execstasy) / [PAM](https://github.com/Utkar5hM/execstasy-pam)

---

### Staging-Server _(IRIS-NITK, Feb 2023 - Mar 2024)_
Served as a maintainer and contributed to the development of a Django-based platform for testing containerized apps, used by NITK developers.
- Refactored architecture and UI for modularity and maintainability.
- Implemented real-time log monitoring and a web terminal with WebSockets and xterm.js.
- Extended deployment to any Git-based repo, improving flexibility.

**Tech:** Django, WebSockets, MySQL, Redis, Celery, Docker, Nginx, xterm.js

[GitHub](https://github.com/IRIS-NITK/Staging-Server)

---

### MySQL/MariaDB Query Profiler _(Aug 2023 - Sep 2023)_
An eBPF-powered Prometheus exporter for real-time query tracing and latency monitoring.
- Benchmarked against Performance Schema using sysbench.
- Enabled probe attachment to live Docker containers using Linux namespaces.
- Live Grafana dashboard for visualizing query performance metrics.

**Tech:** Go, C, Libbpfgo, Bash, Docker, Make

[GitHub](https://github.com/Utkar5hM/mariadb-ebpf-exporter) / [Writeup](https://utkar5hm.github.io/posts/ebpf-vs-perf-schema/)

---

### Smart NIC on FPGA _(Feb 2024 - Mar 2024)_
A custom Network Interface Card with a match-action firewall, built on FPGA.
- Designed and implemented packet processing logic with Emaclite drivers and Microblaze processor.
- Created virtual Ethernet interfaces and Scapy-based communication for real NIC emulation.
- Developed a custom TCAM-IP core with AXI interface and drivers.

**Tech:** Embedded C, Python, Verilog, Linux, Wireshark

[GitHub](https://github.com/Utkar5hM/fpga-based-packet-processing-unit) / [Writeup](https://utkar5hm.github.io/posts/smart-nic-on-fpga/)

---

### Password Cracker RTL-2-GDS-2 _(Jan 2023, Mar 2023)_
Verilog implementation of a hashcat-like SHA256 cracker, synthesized to GDS-2 for VLSI design coursework.

**Tech:** Verilog HDL, RTL Design

[GitHub](https://github.com/BenzeneAlcohol/Password-Cracker)

---

### DockMagic [CTF Challenge / Vulnerable VM] _(Feb 2023)_
A CTF challenge VM focused on Docker breakout, Linux privilege escalation, and web pentesting. Attempted by 825+ users.

**Tech:** Docker, Linux, Web Security

[TryHackMe](https://tryhackme.com/jr/dockmagic) / [Writeup](https://drive.google.com/file/d/1ZLRQgAnoT-Do0gO4i-HeAarvPaXVy_cM/view?usp=sharing)

---

### Reddit 2 Frontends _(Jan 2023)_
Chrome extension to auto-redirect Reddit links to alternative frontends, bypassing campus network blocks.

**Tech:** JavaScript

[GitHub](https://github.com/Utkar5hM/Reddit-2-Frontends)

---

### Bug Tracking System _(Jan 2022)_
Full-stack platform for team-based bug management, with user hierarchy, discussion, and image uploads. 4th place at TRI-NIT hackathon (2000+ registrations).

**Tech:** Node.js, Express.js, eJS, MongoDB, passport.js

[GitHub](https://github.com/BenzeneAlcohol/TRINIT_SCRIPTKIDDIES_DEV)

---

### CTF Hosting Platform _(IEEE-NITK, Jan 2022)_
Platform for hosting digital electronics CTFs with challenge solving, points, and leaderboard.

**Tech:** Node.js, Express.js, eJS, MongoDB, passport.js, Particle.js

[GitHub](https://github.com/IEEE-NITK/ieee-eureka-22)

---

### wa-song _(Oct 2021)_
Node.js bot that updates WhatsApp status with your currently playing Spotify song.

**Tech:** Node.js

[GitHub](https://github.com/Utkar5hM/wa-song)

---

### YelpCamp _(June 2021)_
A full-stack campground review site built as part of a web dev bootcamp. Learned security best practices and integration with MongoDB.

**Tech:** Node.js, Express.js, eJS, MongoDB, passport.js, Bootstrap

[Live](https://protected-basin-08290.herokuapp.com/)
