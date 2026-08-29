# Cache Me Outside — CTF Write-Up

> **Category:** OSINT
> **Difficulty:** Easy 
> **Platform:** THM
> **Room:** Cache Me Outside

---

## Introduction

Years after walking away from the scene, a retired hacker has left pieces of his identity scattered across the open internet.

At first glance, the investigation appears to involve nothing more than a leaked conversation screenshot. However, hidden within that starting point is the first clue in a much larger trail.

Public profiles, forgotten information, exposed details, and small mistakes gradually connect together.

> **Someone wanted this person found.**

The objective is to identify the retired hacker and follow his digital footprint until the final location can be determined.

---

## Objective

The assignment is to:

1. Identify the retired hacker.
2. Find the email address he accidentally exposed.
3. Discover his phone number.
4. Determine the city where he is located.
5. Identify the tram station where he got off on 7 May 2026.

The investigation begins with a publicly available Komoot profile.

---

## Starting Point

The challenge provides the following link:

**Komoot Profile**

https://www.komoot.com/user/5667624959835

The first step is to investigate the profile and look for identifying information.

---

## Q1 — What is the retired hacker's full name?

### Approach

The first clue can be found directly through the provided **Komoot profile**.

By examining the profile information, the retired hacker's identity can be established.

---

## Q2 — What email address did he accidentally expose?

### Approach

Once the hacker's identity is known, the next step is to search for his other online accounts.

One useful lead is his **GitHub account**.

GitHub commit pages can expose additional information through their `.patch` representation.

Examining a commit using the `.patch` endpoint can reveal information that may not be immediately visible on the normal GitHub interface.

### Steps

1. Find the hacker's GitHub profile.
2. Examine his repositories and commits.
3. Identify suspicious or relevant commits.
4. Append `.patch` to the commit URL.
5. Inspect the patch contents.
6. Look for accidentally exposed information such as an email address.



---

## Q3 — What is his phone number?

### Approach

The email address discovered in the previous step provides another avenue for investigation.

The challenge requires contacting the hacker using the exposed email address.

Send an email asking for his phone number.

The response provides the phone number required for the next stage of the investigation.

### Steps

1. Take the email address obtained from the GitHub commit.
2. Send an email to the address.
3. Ask the hacker for his phone number.
4. Use the response to obtain the phone number.


---

## Q4 — In which city is he located?

### Approach

The phone number initially appears to provide a geographical clue.

The number is associated with **Bucharest**, but this turns out not to be the hacker's actual location.

Therefore, the phone number alone is not enough.

The next step is to search for accounts using the username:

```text
jiml33t
```

Searching for this username across different platforms leads to a **Threads** account.

---

## Investigating the Threads Account

The Threads account contains an image that provides another important geographical clue.

The image can be downloaded and analyzed using **Google Lens**.

### Steps

1. Search for the username `jiml33t`.
2. Locate the corresponding Threads account.
3. Examine the posts and images.
4. Identify the relevant image.
5. Run the image through Google Lens.
6. Use the visual matches and location information to identify the city.

This reveals the hacker's actual city.



---

## Q5 — What tram station did he get off at on 7 May 2026?

### Approach

The final question asks for the name of the tram station where the hacker got off on:

```text
7 May 2026
```

The image discovered during the previous stage is important here because it is also associated with the same date.

The goal is to determine the location shown in the image and then identify the nearest tram station.

---

## Investigation

Using the location identified from the image:

1. Determine the exact area shown in the photograph.
2. Locate the relevant point on a map.
3. Identify nearby tram stations.
4. Compare the stations with the location shown in the image.
5. Determine the station closest to the photographed location.
6. Submit the station's name as the final answer.

---

## Complete Investigation Chain

```text
Komoot Profile
      |
      v
Retired Hacker's Name
      |
      v
GitHub Account
      |
      v
GitHub Commit
      |
      v
.patch File
      |
      v
Exposed Email Address
      |
      v
Email the Hacker
      |
      v
Phone Number
      |
      v
Search Username: jiml33t
      |
      v
Threads Account
      |
      v
Image
      |
      v
Google Lens
      |
      v
Actual City
      |
      v
Locate Photograph
      |
      v
Nearest Tram Station
      |
      v
Final Answer
```

---

## Tools Used

| Tool            | Purpose                                   |
| --------------- | ----------------------------------------- |
| Komoot          | Initial profile investigation             |
| GitHub          | Finding the hacker's online activity      |
| GitHub `.patch` | Extracting the accidentally exposed email |
| Email           | Obtaining the phone number                |
| Threads         | Following the `jiml33t` username          |
| Google Lens     | Identifying the location from the image   |
| Maps            | Finding the nearest tram station          |

---

## Key Takeaways

This challenge demonstrates how small pieces of publicly available information can be combined to build a much larger picture of someone's online identity.

Important OSINT techniques used in this challenge include:

* Investigating public profiles
* Pivoting between different online platforms
* Examining GitHub commit metadata
* Finding accidentally exposed contact information
* Username enumeration
* Reverse image searching
* Geolocation
* Mapping locations to nearby public transport

The important lesson is that information that appears harmless individually can become highly revealing when multiple sources are correlated.

---

## Conclusion

The investigation begins with a simple Komoot profile and gradually expands across multiple online platforms.

By following each clue carefully, the trail progresses from:

**Komoot → GitHub → Email → Phone Number → Threads → Image → Geolocation → Tram Station**

The challenge highlights the power of OSINT investigations and demonstrates how publicly available information can be chained together to identify a person and determine their movements.

> **One small clue can lead to an entire digital footprint.**
