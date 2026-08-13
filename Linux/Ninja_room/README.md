# TryHackMe — Ninja

## Overview

This write-up documents the approach used to solve the **Ninja** room.

The investigation focused primarily on Linux filesystem enumeration and identifying specific files based on different properties such as filename, group ownership, content, cryptographic hash, line count, user ownership, and executable permissions.

Rather than relying on a single search condition, multiple approaches were tested to progressively narrow down the relevant files.

---

# 1. Initial File Enumeration

The first step was to search the filesystem for files matching a specific name:

```bash
find / -name 8V2L 2>/dev/null
```

### Reasoning

The objective was to determine whether a file with the specified name existed anywhere on the system.

The following options were used:

* `/` — search from the root of the filesystem.
* `-name 8V2L` — search for the exact filename.
* `2>/dev/null` — suppress permission-denied and other error messages.

This provides a cleaner result while searching across the entire filesystem.

---

# 2. Finding Files Owned by a Specific Group

The next objective was to identify files belonging to the `best-group` group.

```bash
find / -group best-group 2>/dev/null
```

### Reasoning

Linux files have ownership information associated with both a user and a group.

Searching based on group ownership can be useful when a challenge provides a specific group as a clue. Instead of examining every file manually, `find` can filter the filesystem based on ownership attributes.

---

# 3. Searching for a File Containing an IP Address

The next approach involved searching the files listed in:

```text
/home/new-user/file_list.txt
```

The goal was to identify a file containing an IPv4 address.

The initial command was:

```bash
xargs -a /home/new-user/file_list.txt grep -El "([0-9]{1,3}\.){3}[0-9]{1,3}"
```

### Result

This approach did not produce the expected result.

### Reasoning

The file list contained filenames that needed to be located on the filesystem before their contents could be searched reliably.

This led to a different approach: use `find` to locate each filename and then use `grep` on the resulting files.

---

# 4. Combining `find` and `grep`

The next attempt iterated through every filename in `file_list.txt`:

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -exec grep -El "([0-9]{1,3}\.){3}[0-9]{1,3}" '{}' \; 2>/dev/null; done
```

### Result

This produced the expected output file.

### Reasoning

This approach combined two separate operations:

```text
file_list.txt
     |
     v
Read each filename
     |
     v
Find matching file
     |
     v
Search its contents
     |
     v
Check for IPv4 address
```

The `find` command was responsible for locating each file, while `grep` checked whether the file contained a value matching the IPv4 address pattern.

This was more effective than passing the filenames directly to `grep`.

---

# 5. Searching for a Specific SHA-1 Hash

The next clue involved the following SHA-1 hash:

```text
9d54da7584015647ba052173b84d45e8007eba94
```

I attempted to search for this value within the files:

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -exec grep -El "9d54da7584015647ba052173b84d45e8007eba94" '{}' \; 2>/dev/null; done
```

### Result

This approach did not work.

### Reasoning

Searching for the hash as plain text assumes that the hash itself is stored inside one of the files.

However, a SHA-1 clue can instead mean that the **file's contents should be hashed**, rather than that the hash should appear as text within the file.

This distinction led to the next approach.

---

# 6. Calculating SHA-1 Hashes

Instead of searching for the hash as text, I calculated the SHA-1 hash of each candidate file:

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -exec sha1sum '{}' \; 2>/dev/null; done
```

### Result

This worked and produced SHA-1 hashes for the files.

### Reasoning

The important difference between the previous approach and this one is:

```text
Previous approach:
Search file contents for the hash

Current approach:
Calculate the hash of each file
```

The second approach directly tests whether a candidate file corresponds to the provided SHA-1 value.

---

# 7. Comparing File Line Counts

The next clue involved the number of lines contained within the files.

I used:

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -exec wc -l '{}' \; 2>/dev/null; done
```

### Result

The command showed only 12 files, and each contained 209 lines.

However, `file_list.txt` contained 13 filenames.

### Reasoning

This was significant because the expected number of files and the number of files returned by the search did not match.

The discrepancy suggested that one filename from `file_list.txt` did not correspond to a matching file found by the command.

This provided another property that could be used to narrow down the candidates.

---

# 8. Searching by User Ownership

The next clue was related to file ownership.

I searched for files from the list that were owned by user ID `502`:

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -user 502 2>/dev/null; done
```

### Reasoning

Linux file ownership can be represented using either usernames or numeric user IDs.

Using:

```text
-user 502
```

allowed the search to target files belonging specifically to UID `502`.

This is useful when a challenge provides a numeric UID rather than a username.

---

# 9. Searching for Executable Files

The final search condition in the notes focused on executable permissions:

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -executable 2>/dev/null; done
```

### Reasoning

The `-executable` predicate filters files that are executable by the current user.

This provides another way to distinguish candidate files when multiple files share similar names or contents.

The investigation therefore used several different filesystem properties:

```text
Filename
   ↓
Group Ownership
   ↓
Content
   ↓
SHA-1 Hash
   ↓
Line Count
   ↓
User Ownership
   ↓
Executable Permission
```

---

# Investigation Flow

The overall methodology was:

```text
Initial File Search
        |
        v
Group Ownership
        |
        v
Content Search
        |
        v
IP Address Detection
        |
        v
SHA-1 Investigation
        |
        v
Line Count Comparison
        |
        v
User Ownership
        |
        v
Executable Permission
```

---

# Commands Used

### Search for a specific filename

```bash
find / -name 8V2L 2>/dev/null
```

### Search by group ownership

```bash
find / -group best-group 2>/dev/null
```

### Search candidate files for an IPv4 address

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -exec grep -El "([0-9]{1,3}\.){3}[0-9]{1,3}" '{}' \; 2>/dev/null; done
```

### Search for a specific SHA-1 value

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -exec grep -El "9d54da7584015647ba052173b84d45e8007eba94" '{}' \; 2>/dev/null; done
```

### Calculate SHA-1 hashes

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -exec sha1sum '{}' \; 2>/dev/null; done
```

### Check line counts

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -exec wc -l '{}' \; 2>/dev/null; done
```

### Search by UID

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -user 502 2>/dev/null; done
```

### Search for executable files

```bash
for i in $(cat /home/new-user/file_list.txt); do find / -name "$i" -executable 2>/dev/null; done
```

---

# Techniques Used

* Linux filesystem enumeration
* `find` command
* File ownership enumeration
* Group ownership enumeration
* Content-based file searching
* Regular expressions
* SHA-1 hashing
* File metadata analysis
* Permission enumeration
* Command chaining with `find`, `grep`, `sha1sum`, and `wc`

---

# Key Takeaways

## 1. `find` Can Filter on Multiple File Properties

Linux `find` is much more powerful than simply searching by filename.

It can search based on:

```text
Name
Group
User
Permissions
File type
Size
Time
```

This makes it particularly useful during CTF filesystem enumeration.

## 2. Distinguish Between Searching for a Hash and Calculating a Hash

The initial attempt searched for the SHA-1 value as plain text:

```bash
grep -El "9d54da7584015647ba052173b84d45e8007eba94"
```

That failed because the hash was not necessarily stored inside the file.

Calculating the SHA-1 value of each candidate file was the more appropriate approach:

```bash
sha1sum <file>
```

This is an important distinction when a challenge provides a cryptographic hash as a clue.

## 3. Failed Attempts Are Useful

The failed IP-address search and direct hash search helped refine the methodology.

Instead of repeatedly searching the same way, the approach changed from:

```text
Search directly
```

to:

```text
Locate candidate
        ↓
Inspect property
        ↓
Compare result
```

## 4. File Lists Can Be Used as a Controlled Search Set

Rather than blindly examining every file on the system, `/home/new-user/file_list.txt` was used as the candidate set.

Each filename was then processed individually using a loop:

```bash
for i in $(cat /home/new-user/file_list.txt); do
    ...
done
```

This made it possible to apply different filters to the same group of candidate files.

---

# Conclusion

The Ninja room focused heavily on Linux filesystem investigation and the ability to identify files using multiple properties.

The solution process required testing different search conditions, including filenames, group ownership, file contents, SHA-1 hashes, line counts, user ownership, and executable permissions.

The main lesson was that when a CTF provides several clues about a file, each clue should be treated as a filter that progressively reduces the number of possible candidates.

The overall approach can be summarized as:

```text
Enumerate
    ↓
Filter
    ↓
Validate
    ↓
Compare
    ↓
Narrow Down
```
