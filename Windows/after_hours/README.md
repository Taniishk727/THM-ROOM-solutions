# TryHackMe — After Hours

## Overview

This write-up documents the approach used to solve the **After Hours** room. The investigation involved analyzing provided system artifacts, identifying hidden configuration data, extracting an embedded payload, decoding the payload, and analyzing the resulting executable to recover the flag.

The main focus of the investigation was understanding how to move from raw artifacts to useful indicators and then follow those indicators through multiple layers of encoded and compressed data.

---

## Objectives

The room had three primary objectives:

1. Parse the provided system artifacts for hidden custom configuration data.
2. Locate the malicious class and extract its embedded payload.
3. Decode the payload and recover the flag.

---

# 1. Installing Detect It Easy

The first tool used during the investigation was **Detect It Easy (DIE)**. It can be used to analyze executable files and identify useful information about their structure.

I installed it using:

```bash
sudo apt install detect-it-easy
```

At this stage, the tool was installed for later analysis of the executable extracted from the artifacts.

---

# 2. Extracting Strings from the Artifacts

The room provided several system artifact files:

```text
INDEX.BTR
MAPPING1.MAP
MAPPING2.MAP
MAPPING3.MAP
OBJECTS.DATA
```

Instead of manually inspecting each file, I used the `strings` command to extract readable ASCII content.

I first searched for references to `autoruns`:

```bash
strings INDEX.BTR MAPPING1.MAP MAPPING2.MAP MAPPING3.MAP OBJECTS.DATA | grep "autoruns"
```

I then searched for PowerShell-related content:

```bash
strings INDEX.BTR MAPPING1.MAP MAPPING2.MAP MAPPING3.MAP OBJECTS.DATA | grep "powershell"
```

### Reasoning

The objective was to identify useful indicators within the provided artifacts.

`strings` is useful when working with binary or structured files because it can expose human-readable text embedded within them. Searching the resulting output with `grep` makes it possible to quickly identify specific terms without manually reviewing the complete output.

The terms `autoruns` and `powershell` were useful indicators because they could potentially point toward persistence mechanisms or malicious execution activity.

---

# 3. Identifying the Encoded Data

During the artifact analysis, Base64-encoded data was identified.

The next step was to convert the encoded content into a readable form.

Base64 is an encoding mechanism that represents binary data using ASCII characters. It is commonly encountered during security investigations because encoded data can be embedded inside configuration files, scripts, or other artifacts.

The extracted Base64 data was therefore preserved for further analysis.

---

# 4. Searching for `Win32_HardwareTelemetry`

The next useful indicator identified during the investigation was:

```text
Win32_HardwareTelemetry
```

I searched for this string within the extracted strings from the artifact files:

```bash
strings INDEX.BTR MAPPING1.MAP MAPPING2.MAP MAPPING3.MAP OBJECTS.DATA | grep "Win32_HardwareTelemetry"
```

### Reasoning

At this point, searching for generic terms such as `powershell` was no longer sufficient. A specific string such as `Win32_HardwareTelemetry` provided a more targeted way to locate the relevant malicious class or configuration data.

This allowed the investigation to move from general artifact enumeration toward identifying the specific data containing the embedded payload.

---

# 5. Decoding the Payload

The Base64 data associated with the discovered content was copied into CyberChef.

The following operations were used:

```text
From Base64
    ↓
Raw Inflate
```

The first operation decoded the Base64 representation.

The resulting data was then processed using **Raw Inflate** to decompress the compressed payload.

The overall process was:

```text
Base64-encoded data
        |
        v
    Base64 Decode
        |
        v
Compressed Data
        |
        v
    Raw Inflate
        |
        v
Decoded Payload
```

### Reasoning

The important point here was that the data was not simply Base64 encoded. After Base64 decoding, the resulting content still required decompression.

Recognizing the difference between encoding and compression was necessary to correctly recover the embedded payload.

---

# 6. Extracting `download.exe`

After decoding and decompressing the payload, an executable was obtained.

The executable was saved as:

```text
download.exe
```

This executable was then analyzed using Detect It Easy.

```bash
die download.exe
```

### Reasoning

The investigation had now moved from system artifacts to an executable payload.

Rather than immediately attempting to execute the file, I used a binary analysis tool to inspect it and identify useful information contained within the executable.

---

# 7. Analyzing the Executable

Within Detect It Easy, I inspected the **Strings** section of `download.exe`.

One of the interesting strings identified was:

```text
netuserpatch
```

This provided another indicator that could be used to locate the relevant data within the executable.

The investigation therefore followed the same approach used earlier with the system artifacts: identify meaningful strings and use them as pivots for further analysis.

---

# 8. Extracting the Flag

The data associated with `netuserpatch` was extracted and passed into CyberChef for decoding.

The decoded output contained the final flag.

```text
THM{FLAG}
```

The actual flag should be inserted above after completing the room.

---

# Attack Chain

The complete investigation can be summarized as follows:

```text
System Artifacts
       |
       v
     strings
       |
       +---- autoruns
       |
       +---- powershell
       |
       +---- Win32_HardwareTelemetry
                    |
                    v
             Base64 Payload
                    |
                    v
              Base64 Decode
                    |
                    v
               Raw Inflate
                    |
                    v
               download.exe
                    |
                    v
             Detect It Easy
                    |
                    v
              String Analysis
                    |
                    v
              netuserpatch
                    |
                    v
             CyberChef Decode
                    |
                    v
                  FLAG
```

---

# Commands Used

### Install Detect It Easy

```bash
sudo apt install detect-it-easy
```

### Search for `autoruns`

```bash
strings INDEX.BTR MAPPING1.MAP MAPPING2.MAP MAPPING3.MAP OBJECTS.DATA | grep "autoruns"
```

### Search for PowerShell

```bash
strings INDEX.BTR MAPPING1.MAP MAPPING2.MAP MAPPING3.MAP OBJECTS.DATA | grep "powershell"
```

### Search for `Win32_HardwareTelemetry`

```bash
strings INDEX.BTR MAPPING1.MAP MAPPING2.MAP MAPPING3.MAP OBJECTS.DATA | grep "Win32_HardwareTelemetry"
```

### Analyze the extracted executable

```bash
die download.exe
```

---

# Tools Used

| Tool           | Purpose                                              |
| -------------- | ---------------------------------------------------- |
| `strings`      | Extract readable strings from the provided artifacts |
| `grep`         | Search extracted data for specific indicators        |
| Detect It Easy | Analyze the extracted executable                     |
| CyberChef      | Decode Base64 and decompress the payload             |

---

# Techniques Used

* Artifact analysis
* String extraction
* Indicator-based searching
* Base64 decoding
* Raw DEFLATE decompression
* Executable analysis
* Payload extraction

---

# Key Takeaways

The main lesson from this room was the importance of following useful indicators through multiple layers of data.

The investigation followed a relatively simple chain:

```text
Artifact
    ↓
Interesting String
    ↓
Encoded Data
    ↓
Decoded Payload
    ↓
Executable
    ↓
Interesting String
    ↓
Encoded Data
    ↓
Flag
```

Rather than attempting to analyze everything within the provided artifacts, targeted searches were used to identify relevant strings and progressively narrow down the investigation.

The room also demonstrated the importance of recognizing different forms of data transformation. The payload required both Base64 decoding and Raw Inflate decompression before the executable could be recovered.

---

# Conclusion

The After Hours room provided practical experience with analyzing system artifacts and following embedded data through multiple stages of encoding and compression.

Starting with the provided artifact files, I used `strings` and `grep` to identify relevant indicators, including `Win32_HardwareTelemetry`. This led to an encoded payload that was decoded using Base64 and Raw Inflate, resulting in `download.exe`.

The executable was then analyzed using Detect It Easy, where the `netuserpatch` string provided the final pivot toward the flag.

The overall process reinforced an important approach in security investigations:

```text
Identify → Extract → Decode → Analyze → Pivot → Repeat
```
