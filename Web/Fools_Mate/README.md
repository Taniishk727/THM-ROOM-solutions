# TryHackMe — Fools_mate

## Overview

This write-up documents the approach used to solve the **Fools_mate** room.

The challenge involves a chess-based web application where the board indicates that checkmate is one move away. However, attempting to make the checkmate move through the website results in a message indicating that the move is not allowed.

The investigation involved analyzing the client-side JavaScript, identifying the function responsible for preventing the move, capturing a legitimate chess move request using Burp Suite, and modifying the request through Burp Repeater to bypass the client-side restriction.

The main attack path was:

```text
Chess Board

        ↓

Attempt Checkmate

        ↓

Client-Side Restriction

        ↓

Analyze app.js

        ↓

Identify Move-Blocking Function

        ↓

Capture Valid Move Request

        ↓

Burp Suite Repeater

        ↓

Modify Position Parameters

        ↓

Send Modified Request

        ↓

Checkmate

        ↓

Flag
```

---

# 1. Understanding the Challenge

The challenge provides a chess board and several pieces.

The board indicates that checkmate is one move away. However, when attempting to make the checkmate move using the chess interface, the application displays:

```text
I'll shut down the PC if you play that move
```

This indicates that the application is deliberately preventing the move.

### Reasoning

Since the board indicates that the move should result in checkmate but the application prevents it, I suspected that the restriction might be implemented through client-side JavaScript.

Instead of repeatedly attempting the same move, I decided to inspect the website's source code and JavaScript to understand how the move was being blocked.

---

# 2. Analyzing the Website Source

While inspecting the source code of the website, I found an `app.js` script.

I examined the JavaScript to understand how the application handled chess moves.

During the analysis, I identified the function responsible for preventing the checkmate move.

### Reasoning

Finding the move-blocking function was important because it indicated that the restriction was being handled by the client-side application.

The expected application flow was:

```text
Chess Board
     |
     v
JavaScript
     |
     v
Move Validation
     |
     v
HTTP Request
     |
     v
Server
```

If the JavaScript prevents the move before the request is sent, the restriction may be bypassed by interacting directly with the server.

---

# 3. Capturing a Chess Move Request

I used Burp Suite to intercept the HTTP request generated when making a normal chess move.

The purpose was to understand how the application represented a chess move in the HTTP request.

Instead of attempting the blocked checkmate move directly through the interface, I first captured a move that the application allowed.

### Reasoning

The captured request provided a template that could be modified.

The goal was to determine whether the server itself validated the chess move or whether the restriction existed only within the client-side JavaScript.

The approach was:

```text
Valid Chess Move
       |
       v
Capture HTTP Request
       |
       v
Inspect Request Parameters
       |
       v
Modify Move
       |
       v
Send Request Directly
```

---

# 4. Using Burp Suite Repeater

The captured request was sent to Burp Suite Repeater.

Burp Repeater allows an HTTP request to be modified and resent multiple times without interacting with the web application's interface.

I inspected the request and identified the position values responsible for defining the chess move.

The next step was to modify these position values so that they represented the required checkmate move.

### Reasoning

The original interaction was:

```text
Attempt Checkmate
       |
       v
Client-Side Validation
       |
       v
Move Blocked
```

Instead, I used:

```text
Capture Valid Request
       |
       v
Burp Repeater
       |
       v
Modify Position Values
       |
       v
Send Request Directly
       |
       v
Server Processes Move
```

This allowed the request to bypass the restriction implemented by the client-side interface.

---

# 5. Modifying the Chess Move

In Burp Suite Repeater, I modified the position values in the captured request so that they represented the required checkmate move.

The modified request was then sent directly to the server.

### Reasoning

The important part of the exploitation was that the move was no longer being generated through the normal chess interface.

The browser-side JavaScript was responsible for preventing the move, but the underlying HTTP request could be manipulated independently.

This allowed the server to receive a request representing the checkmate move without going through the same client-side restriction.

---

# 6. Retrieving the Flag

After modifying the position values in the request, I sent the request through Burp Repeater.

The server accepted the modified checkmate move and returned the challenge flag.

This confirmed that the restriction preventing the move was implemented on the client side and could be bypassed by directly manipulating the HTTP request.

---

# Attack Chain

```text
Target Web Application

        |

        v

Chess Board

        |

        v

Attempt Checkmate

        |

        v

Client-Side Restriction

        |

        v

Inspect app.js

        |

        v

Identify Move-Blocking Function

        |

        v

Capture Normal Move Request

        |

        v

Burp Suite

        |

        v

Burp Repeater

        |

        v

Modify Position Parameters

        |

        v

Send Modified Request

        |

        v

Checkmate

        |

        v

Flag
```

---

# Commands / Requests Used

## Capturing a Normal Move

A normal chess move was performed through the web interface while Burp Suite was running.

The resulting HTTP request was intercepted and sent to Burp Repeater.

## Modifying the Move

The position parameters in the captured request were modified in Burp Repeater to represent the checkmate move.

The modified request was then sent directly to the server.

---

# Tools Used

| Tool                    | Purpose                                   |
| ----------------------- | ----------------------------------------- |
| Browser Developer Tools | Inspect the website source and JavaScript |
| Burp Suite              | Intercept HTTP requests                   |
| Burp Repeater           | Modify and resend the chess move request  |

---

# Techniques Used

* Client-side JavaScript analysis

* Client-side validation bypass

* HTTP request interception

* HTTP request manipulation

* Parameter manipulation

* Burp Suite Repeater

---

# Failed Attempts

## Attempting the Checkmate Through the Interface

The initial approach was to perform the checkmate move directly through the chess interface.

The application prevented the move and displayed:

```text
I'll shut down the PC if you play that move
```

This indicated that the move was being deliberately blocked.

Instead of continuing to interact only through the interface, I moved to analyzing the JavaScript responsible for handling the move.

---

# Key Takeaways

## 1. Client-Side Validation Is Not a Security Boundary

The main lesson from this room is that client-side restrictions should not be treated as a reliable security control.

A user can inspect the JavaScript and interact with the backend independently of the application's interface.

Security-sensitive actions should therefore be validated on the server side.

## 2. Inspect the HTTP Request

When an application prevents an action through the user interface, inspecting the underlying HTTP request can reveal how the application communicates with the server.

A legitimate request can then be captured and tested for parameter manipulation.

## 3. Burp Repeater Is Useful for Testing Application Logic

Burp Repeater made it possible to modify the chess move without interacting with the chess interface.

This allowed the client-side restriction to be bypassed and the server's handling of the modified request to be tested.

## 4. Understand the Difference Between Client and Server Logic

The important distinction in this room was:

```text
Client-Side Logic
        |
        v
Prevents the Move
```

versus:

```text
Server-Side Logic
        |
        v
Processes the HTTP Request
```

By communicating directly with the server, it was possible to determine that the client-side restriction was not sufficient to prevent the checkmate move.

---

# Conclusion

The Fools_mate room demonstrated how client-side restrictions can sometimes be bypassed by directly manipulating the HTTP requests sent to a server.

The application prevented the checkmate move through its client-side JavaScript. After inspecting the `app.js` file, I identified the function responsible for blocking the move.

I then used Burp Suite to capture a legitimate chess move request and sent it to Burp Repeater. The position parameters were modified to represent the required checkmate move, and the modified request was sent directly to the server.

The server accepted the modified request and returned the flag.

The overall methodology was:

```text
Analyze
    ↓
Identify Client-Side Restriction
    ↓
Capture Request
    ↓
Modify Parameters
    ↓
Bypass UI Validation
    ↓
Send Request Directly
    ↓
Retrieve Flag
```

---

# References

* TryHackMe — Fools_mate
* Burp Suite
