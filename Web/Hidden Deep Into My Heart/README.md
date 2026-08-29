# Hidden Deep Into My Heart — CTF Write-Up

> **Category:** Web / Enumeration
> **Difficulty:** Easy
> **Platform: **THM
> **Room:** Hidden Deep Into My Heart

---

## Introduction

Cupid's Vault was designed to protect secrets meant to stay hidden forever. Unfortunately, Cupid underestimated how determined attackers can be.

Intelligence indicates that Cupid may have unintentionally left vulnerabilities in the system. With the holiday deadline approaching, you've been tasked with uncovering what's hidden inside the vault before it's too late.

---

## Objective

The objective of this challenge is to:

* Discover hidden directories and files.
* Investigate the `robots.txt` file.
* Find the secret vault.
* Enumerate the vault for additional hidden pages.
* Discover the login page.
* Identify the credentials required to access the vault.
* Retrieve the flag.

---

## Starting Point

The target was:

```text
http://10.48.164.138:5000
```

The first step was to perform directory enumeration using **Gobuster**.

---

## Step 1 — Directory Enumeration

I used Gobuster with the common Dirb wordlist:

```bash
gobuster dir -u http://10.48.164.138:5000 -w /usr/share/wordlists/dirb/common.txt
```

The scan revealed a `robots.txt` file.

---

## Step 2 — Investigating robots.txt

Navigating to:

```text
http://10.48.164.138:5000/robots.txt
```

revealed the following:

```text
User-agent: *
Disallow: /cupids_secret_vault/*
# cupid_arrow_2026!!!
```

The `Disallow` entry revealed an interesting directory:

```text
/cupids_secret_vault/
```

More importantly, the comment contained what appeared to be a password:

```text
cupid_arrow_2026!!!
```

---

## Step 3 — Accessing the Secret Vault

Using the directory discovered in `robots.txt`, I navigated to:

```text
http://10.48.164.138:5000/cupids_secret_vault/
```

The page displayed:

```text
You've found the secret vault, but there's more to discover...
```

This indicated that the discovered directory was only the beginning.

---

## Step 4 — Enumerating the Vault

I ran Gobuster against the newly discovered directory:

```bash
gobuster dir -u http://10.48.164.138:5000/cupids_secret_vault/ -w /usr/share/wordlists/dirb/common.txt
```

The enumeration revealed a **login page**.

---

## Step 5 — Finding the Credentials

Initially, I attempted to fuzz possible credentials using FFUF, including:

```text
admin
passwd
```

However, these attempts did not reveal anything useful.

Looking back at the information already discovered in `robots.txt`, there was an interesting comment:

```text
# cupid_arrow_2026!!!
```

This looked like a password.

Using:

```text
Username: admin
Password: cupid_arrow_2026!!!
```

I was able to successfully authenticate.

---

## Step 6 — Retrieving the Flag

After logging in with the discovered credentials, the vault was successfully accessed and the flag was obtained.

```text
Username: admin
Password: cupid_arrow_2026!!!
```

---

## Complete Attack Chain

```text
Target
  |
  v
Gobuster
  |
  v
robots.txt
  |
  +--> /cupids_secret_vault/
  |
  +--> cupid_arrow_2026!!!
  |
  v
Secret Vault
  |
  v
Gobuster Again
  |
  v
Login Page
  |
  v
admin : cupid_arrow_2026!!!
  |
  v
Authenticated
  |
  v
FLAG
```

---

## Tools Used

| Tool       | Purpose                                          |
| ---------- | ------------------------------------------------ |
| Gobuster   | Directory and file enumeration                   |
| FFUF       | Credential fuzzing attempts                      |
| Browser    | Accessing discovered endpoints                   |
| robots.txt | Discovering the hidden vault and leaked password |

---

## Key Takeaways

This challenge demonstrates the importance of checking seemingly insignificant files such as `robots.txt`.

The key vulnerability was the exposure of sensitive information inside a comment:

```text
# cupid_arrow_2026!!!
```

Although the comment was not intended to be displayed as page content, it was publicly accessible and contained the password required to access the login page.

The main techniques used were:

* Directory enumeration
* `robots.txt` enumeration
* Hidden endpoint discovery
* Recursive directory enumeration
* Credential discovery
* Authentication bypass through exposed credentials

---

## Conclusion

The challenge was solved by chaining together multiple enumeration steps.

First, Gobuster discovered `robots.txt`. The file revealed the hidden `/cupids_secret_vault/` directory as well as a password contained in a comment.

The vault was then enumerated again, revealing a login page. Using `admin` as the username and the password found in `robots.txt` resulted in successful authentication and access to the flag.

The final attack path was:

**Gobuster → robots.txt → Secret Vault → Gobuster → Login → Exposed Credentials → Flag**
