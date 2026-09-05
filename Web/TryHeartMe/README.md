# TryHeartMe — Valenflag

## Challenge Description

> The TryHeartMe shop is open for business. Can you find a way to purchase the hidden **“Valenflag”** item?

**Target:** `http://10.48.132.248:5000`

---

## Reconnaissance

After accessing the TryHeartMe web application, the first thing I checked was the cookies associated with the session.

One interesting cookie was:

```text
tryheartme_jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6Inh5ekBnbWFpbC5jb20iLCJyb2xlIjoidXNlciIsImNyZWRpdHMiOjAsImlhdCI6MTc4ODUyNTA4OSwidGhlbWUiOiJ2YWxlbnRpbmUifQ.D2hKz8ymI7MzVNLi6LxGQgNhR2vm4Wb3d-FujAIu0Ro
```

The cookie had the structure of a **JWT (JSON Web Token)**:

```text
HEADER.PAYLOAD.SIGNATURE
```

This immediately made the JWT worth investigating.

---

## Decoding the JWT

Decoding the JWT revealed the following payload:

```json
{
  "email": "xyz@gmail.com",
  "role": "user",
  "credits": 0,
  "iat": 1788525089,
  "theme": "valentine"
}
```

Two fields immediately stood out:

```json
"role": "user",
"credits": 0
```

The application appeared to use these values from the JWT to determine the user's privileges and available credits.

This suggested that if the server trusted the values inside the token without properly validating its signature, modifying the token could potentially change our privileges.

---

## JWT Manipulation

The original payload was modified to:

```json
{
  "email": "xyz@gmail.com",
  "role": "admin",
  "credits": 999999,
  "iat": 1788525089,
  "theme": "valentine"
}
```

The important changes were:

```diff
- "role": "user"
+ "role": "admin"

- "credits": 0
+ "credits": 999999
```

The modified payload was then encoded as a JWT.

The key observation during testing was that the application did not properly validate the JWT signature/key before trusting the modified claims.

---

## Sending the Modified Token

Using **Burp Suite**, the modified JWT was placed back into the `tryheartme_jwt` cookie.

The request was then forwarded to the application.

After the server accepted the modified token, the application treated the session as an administrator with a large number of credits.

---

## Accessing the Hidden Item

I then returned to the shop and inspected the purchase functionality.

With the modified privileges, an additional administrative option became available.

The previously hidden **Valenflag** item could now be accessed and purchased.

Since the modified JWT provided:

```json
"credits": 999999
```

there were sufficient credits to complete the purchase.

---

## Flag

Purchasing the **Valenflag** item revealed the challenge flag.

```text
FLAG: <obtained flag>
```

---

## Vulnerability

The root cause of the challenge was **improper JWT validation**.

The application trusted sensitive authorization information directly from the JWT:

```json
"role": "user",
"credits": 0
```

without properly verifying that the token had been signed using the expected secret/key.

This allowed the token's claims to be manipulated.

### Attack Flow

```text
JWT Cookie
    |
    v
Decode JWT
    |
    v
Modify "role" and "credits"
    |
    v
Re-encode JWT
    |
    v
Replace Cookie using Burp Suite
    |
    v
Server accepts modified claims
    |
    v
Admin privileges + huge credit balance
    |
    v
Access hidden Valenflag item
    |
    v
Purchase item
    |
    v
FLAG
```

## Key Takeaway

JWT payloads should **never be trusted simply because they are structurally valid**.

The server must:

1. Verify the JWT signature using the correct secret/public key.
2. Validate the expected signing algorithm.
3. Avoid trusting client-controlled claims such as `role`, `credits`, or permissions without proper server-side authorization checks.
4. Perform authorization checks server-side rather than relying solely on values supplied inside a client-controlled token.

In this challenge, manipulating the JWT allowed a normal user session to effectively become an administrator and purchase the hidden **Valenflag** item.
