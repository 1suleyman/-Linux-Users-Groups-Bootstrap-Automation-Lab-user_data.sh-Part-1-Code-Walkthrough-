# 🐧 Linux Users/Groups Bootstrap Automation Lab (user_data.sh) — Part 1 (Code Walkthrough)

> **Bear in mind:** this is **Part 1 of a 2-part lab**.
> **This part is me reading + explaining the script logic.**
> [Part 2 will be me testing it on the EC2 instance and verifying the results in cloud-init logs.](https://github.com/1suleyman/-Linux-Lab-1-Bootstrap-Automation-user_data.sh-Part-2-Testing-Debugging-)

In this lab, I explained how my `user_data.sh` script automates **Lab 1 Linux admin setup** (users, groups, passwords, directory structure, ownership), so **Lab 2 starts in a ready-to-go state**.

---

## 📋 Lab Overview

**Goal:**

* Define reusable lists (arrays) of users + groups
* Add a consistent `log()` function for readable cloud-init output
* Create groups/users **idempotently**
* Set lab-only passwords using `chpasswd` (cloud-init friendly)
* Assign primary + supplementary groups properly
* Create directories/files and apply ownership/group ownership
* Add a **verification block** that prints useful state **without crashing** the script
* End with a “sentinel” log line to confirm the script reached the end

**Learning Outcomes:**

* Use Bash arrays + loops to keep scripts maintainable
* Understand Bash functions and `$*` (pass-through arguments)
* Use exit codes (0 = success, non-zero = failure) for `if` logic
* Understand `>/dev/null` vs `2>&1` and what gets hidden
* Understand `|| true` as a “don’t crash” escape hatch when using `set -e`
* Use `chpasswd` for non-interactive password setting (cloud-init safe)
* Distinguish **user groups (primary/supplementary)** vs **file ownership (owner/group)**

---

## 🛠 Step-by-Step Journey (Part 1 — Script Explanation)

### Step 1: Define Users + Groups as Bash Arrays

**Concept:** Use arrays so you can loop instead of duplicating code.

**Example:**

```bash
users=(user1 user2 user3)
groups=(devops aws)
```

* Creates reusable lists
* Makes the script easier to scale (add/remove users without rewriting logic)

---

### Step 2: Create a Reusable `log()` Function

**Concept:** Instead of repeating `echo` everywhere, define a logging function once.

**Example:**

```bash
log() { echo "[Lab 1] $*"; }
log "Starting Lab 1 bootstrap"
```

* First line **defines** the function
* Second line **calls** the function
* `$*` means “all arguments passed into this function”

✅ **Why it matters:** consistent formatting + cleaner output in cloud-init logs.

---

### Step 3: Create Groups Idempotently (Loop + Exit Codes)

**Example pattern:**

```bash
for g in "${groups[@]}"; do
  if getent group "$g" >/dev/null; then
    log "Group exists: $g"
  else
    log "Creating group: $g"
    groupadd "$g"
  fi
done
```

**Key idea:** `if` checks the **exit code**, not the command output.

* `getent group devops`

  * exit code `0` → group exists → “true”
  * non-zero → group missing → “false”

**Redirection insight:**

* `>/dev/null` **hides stdout** only (throws away normal output)
* It does **not** change exit codes

---

### Step 4: Create Users with Home Directories + Bash Shell

**Example pattern:**

```bash
for u in "${users[@]}"; do
  if id "$u" >/dev/null 2>&1; then
    log "User exists: $u"
  else
    log "Creating user with home: $u"
    useradd -m -s /bin/bash "$u"
  fi
done
```

**Flags:**

* `-m` → create home directory
* `-s /bin/bash` → set login shell to Bash

**Redirection insight:**

* `>/dev/null` hides stdout
* `2>&1` sends stderr to the same place as stdout
  ✅ together: hide **everything**, keep only exit code logic

**Design choice (your decision):**
You noted that hiding stderr can hide useful failures, so you prefer to **keep errors visible** unless they’re expected noise.

---

### Step 5: Set Lab-Only Passwords using `chpasswd`

**Example pattern:**

```bash
for u in "${users[@]}"; do
  echo "$u:${LAB_PASSWORD}" | chpasswd
done
```

**Why `chpasswd` (not `passwd`):**

* `passwd` expects interactive input (doesn’t work well in cloud-init)
* `chpasswd` reads `username:password` from stdin (automation-friendly)

✅ **Lab-safe note:** In real production, you would not hardcode passwords or reuse one password for all users.

---

### Step 6: Set Primary Group for user2 + user3 (devops)

**Example:**

```bash
log "Setting primary group devops for user2 and user3"
usermod -g devops user2
usermod -g devops user3
```

* `-g` (lowercase) = **primary group**
* Silence usually means success (Linux tools “speak” on failure)

You noted you *could* loop here, but with only 2 users, clarity > abstraction.

---

### Step 7: Add Supplementary Group (aws) for user1

**Example:**

```bash
log "Adding supplementary group aws for user1"
usermod -aG aws user1
```

**Flags:**

* `-G` (uppercase) = supplementary groups
* `-a` = append (prevents overwriting existing group memberships)

⚠️ Important nuance:
If `user1` is already logged in, they won’t see new group permissions until:

* log out + log back in, **or**
* run `newgrp aws`

---

### Step 8: Create Directory Structure + Files (Idempotent)

**Example:**

```bash
log "Creating directory structure"
mkdir -p /dir1 /dir7 /dir10

log "Creating files"
touch /dir1/f1 /dir1/f2
```

* `mkdir -p` won’t fail if directories exist
* `touch` creates file if missing, updates timestamp if present
  ✅ safe to rerun

---

### Step 9: Ownership + Group Ownership

**Example:**

```bash
log "Changing group ownership to devops"
chgrp devops /dir1 /dir7 /dir10 /dir1/f2

log "Changing user ownership to user1"
chown user1 /dir1 /dir7 /dir10 /dir1/f2
```

✅ Correction you made (important):

* Users have **primary/supplementary groups**
* Files/dirs do **not** have primary groups
  They have:
* **owner (user)**
* **group owner (group)**

Mental model:

* `chgrp` = “which team owns this?”
* `chown` = “which user owns this?”

---

### Step 10: Verification Block (Using `|| true`)

You explained why this exists:

Because the script uses `set -e` (exit on failure), verification commands could accidentally crash the entire bootstrap. So the verification section uses:

```bash
getent group devops aws || true
getent passwd user1 user2 user3 || true
id user1 || true
ls -ld /dir1 /dir7 /dir10 /dir1/f2 || true
```

**Meaning of `|| true`:**

* Run command
* If it fails, run `true` (which always succeeds)
* This prevents `set -e` from killing the script

✅ Mental model:

> “Best effort check — log what you can, never crash the build.”

---

### Step 11: Final Sentinel Log Line

**Example:**

```bash
log "Lab 1 bootstrap complete"
```

Why it’s powerful:

* If this line appears in `/var/log/cloud-init-output.log`, you know the script reached the end.
* If it’s missing, something failed earlier — and you search above.

---

## ✅ Key Commands Summary

| Concept                        | Command / Pattern              |  
| ------------------------------ | ------------------------------ | 
| Bash array of users            | `users=(user1 user2 user3)`    |   
| Bash array of groups           | `groups=(devops aws)`          | 
| Reusable logging function      | `log() { echo "[Lab 1] $*"; }` |  
| Check if group exists          | `getent group "$g"`            |  
| Create group                   | `groupadd "$g"`                | 
| Check if user exists           | `id "$u"`                      |  
| Create user with home + bash   | `useradd -m -s /bin/bash "$u"` |  
| Hide stdout only               | `>/dev/null`                   | 
| Hide stdout + stderr           | `>/dev/null 2>&1`              |   
| Print last exit code           | `echo $?`                      |  
| Set password non-interactively | `echo "u:pass" \| chpasswd`    | 
| Set primary group              | `usermod -g devops user2`      | 
| Add supplementary group safely | `usermod -aG aws user1`        |  
| Create dirs idempotently       | `mkdir -p ...`                 | 
| Create files idempotently      | `touch ...`                    |  
| Set group owner                | `chgrp devops ...`             |  
| Set user owner                 | `chown user1 ...`              |   
| Prevent `set -e` crash         | `command true`                 | 

---

## 💡 Notes / Tips (From Your Commentary)

* `if` statements in Bash usually check **exit codes**, not printed output.
* `>/dev/null` is a **black hole** for stdout — it doesn’t affect true/false logic.
* `2>&1` means “send stderr (2) to wherever stdout (1) is going”.
* `|| true` is a deliberate “escape hatch” for verification blocks under `set -e`.
* `usermod -aG` is critical — without `-a`, you can overwrite group memberships.
* Files/dirs don’t have primary groups — only **owner + group owner**.
* Cloud-init runs as root → no `sudo` needed in user_data scripts.

---

## 📌 Lab Summary (Part 1)

| Section Explained               | Status | Key Takeaway                          |  
| ------------------------------- | ------ | ------------------------------------- | 
| Arrays for users/groups         | ✅      | Makes loops clean + scalable          |   
| `log()` function                | ✅      | Consistent output + readable logs     |  
| Group creation loop             | ✅      | `getent` + exit codes for idempotency |  
| User creation loop              | ✅      | `id` + `useradd -m -s /bin/bash`      | 
| Redirection + exit codes        | ✅      | Hide output, keep logic via exit code |  
| Password setting                | ✅      | `chpasswd` is cloud-init friendly     |  
| Primary vs supplementary groups | ✅      | `-g` vs `-aG` matters                 |   
| Ownership model                 | ✅      | `chown` vs `chgrp` (user vs team)     |  
| Verification block              | ✅      | `true`prevents`set -e` crash          |  
| Sentinel completion log         | ✅      | Easy “did it finish?” check           |  

---

## ✅ References

* Bash exit codes + `$?`
* `getent` usage
* `chpasswd` for non-interactive passwords
* `usermod` flags (`-g`, `-aG`)
