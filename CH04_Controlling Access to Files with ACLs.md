# 📘 Chapter 4 — Access Control Lists (ACLs)

## 🔍 Difference Between ACLs and Traditional Permissions

`Access Control Lists` (ACLs) extend the traditional Linux permission model by providing fine-grained access control.\
**The table below summarizes the key differences**:

|**Feature** | **Traditional Permissions** | **ACL (Access Control List)**|
| ------------------------ | --------------------------------------------------------------------------------- | ----------------- |
|**Definition**|**Standard** Linux permission |Advanced, **flexible** method for **fine-grained permission** control|
|**Control Scope**|Only **owner** (user), **group**, **others**|Any **specific user**, **specific group**, **mask**, basic permissions|
|**Flexibility**|**Limited** — cannot set different permissions for multiple users at the same time |Highly **flexible** — you can grant **custom permissions to any user or group** on the same file|
|**Use Case Example**|`chmod 640 file.txt`<br>**user**: RW, **group**: R, **others**: none |`setfacl -m u:john:r file.txt` **only John** has read access without modifying the file owner or group|
|**Kernel Support Check**|Not required; Supported by default|Requires ACL support in the kernel: <br>     `grep -i acl /boot/config*`     <br> Must have `CONFIG_FS_POSIX_ACL=Y`|
| **Tools Used**|`chmod`, `chown`, `ls -l`| `setfacl`, `getfacl`|


## 🔐 ACL Indicators and Commands:
### 1️⃣ Detecting ACLs
Use `ls -l` to check if a file has ACLs.\
If you see a + after the permission bits → ACL exists.

#### Example:
```ini
-rw-r--r--+ 1 user user 1234 Oct 23 file.txt
```
> The + indicates that file has an extended ACL.

### 2️⃣ Viewing ACLs
```bash
# Displays full and detailed ACL entries
getfacl <file>
```

### 3️⃣ Managing ACLs (setfacl)
| **Option**            | **Description**                                            | **Example Command**                          |
| --------------------- | ---------------------------------------------------------- | -------------------------------------------- |
| **-m**                | Add or modify ACL entries (user/group)                     | `setfacl -m u:ali:r file.txt`                |
| **-x**                | Remove specific ACL entries                                | `setfacl -x g:developers file.txt`           |
| **-b**                | Remove **all** ACLs and return to basic permissions        | `setfacl -b file.txt`                        |
| **-R**                | Apply ACLs **recursively** to directories and all children | `setfacl -m g:admins:rwX -R /shared/data/`   |
| **-d**                | Set or modify **default ACLs** for a directory             | `setfacl -d -m g:devs:rwX /shared/`          |
| **-n**                | Do **not** recalculate the mask automatically              | `setfacl -n -m u:toka:rwx project.txt`       |
| **--set-file=<file>** | Replace ACLs from a file created via `getfacl`             | `setfacl --set-file=acl_backup.txt file.txt` |
***---***

## 🧩 Types of ACL Entries (from getfacl outputt)
### 1️⃣ Comment Entries (Metadata)
Appear at the top of the `getfacl` output:
-   **Filename** (`# file: filename`) 
-   **User owner** (`# owner: username`)
-   **Group owner** called operatores (`# group: groupname`)
-   **Flags** (Optional line) \
    Shown only if the file has special permission bits:
    | File                      | Flags | Meaning        |
    | ------------------------- | ----- | -------------- |
    | `getfacl /usr/bin/passwd` | `s--` | SUID set       |
    | `getfacl /usr/bin/locate` | `-s-` | SGID set       |
    | `getfacl /tmp`            | `--t` | Sticky bit set |

### 2️⃣ User Entries     
Lists permissions for enteries started with `user`.
#### Examples: 
```ini
    user::rwx       → file owner permissions
    user:toka:---   → named user (by username)
    user:1005:rwx   → named user (by UID)
```    
### 3️⃣ Group Entries
Lists permissions for enteries started with `group`.
#### Examples: 
```ini
    group::rwx → group owner permissions
    group:toka:--- → named group (group name)
    group:1005:rwx → named group (GID)
```
### 4️⃣ Mask Entry
Defines the maximum permissions allowed for:
-   Group owner
-   Named users
-   Named groups

Does NOT affect:
-   file owner
-   others 

#### Example: 
```ini
    mask::rw-
    user:toka:rwx    #effective:rw-
```
> The mask reduces the effective permissions.

### 5️⃣ Other Entry 
Defines permissions for everyone not included in:
-   Owner
-   Group owner
-   Named users
-   Named groups

#### Example: 
```ini
    other::---
```

## ACL Permission Precedence (Order of Evaluation)
Linux checks ACL entries in this order:
1.  **User owner** -- Owner permissions apply (mask not applied).
2.  **Named user** -- Explicit user ACL, limited by mask.
3.  **Group owner** -- Group owner permissions, limited by mask.
4.  **Named group** -- Explicit named group permissions, limited by mask.
5.  **Other** -- Applies if none of the above match.


## Mask and Effective Permissions
The **mask** limits maximum permissions for: 
-   Named users
-   Group owner
-   Named groups

#### Example:
```ini
    # file: example.txt
    user:user1:rwx       #effective:rw-
    user:user2:r--
    mask::rw-
```
`user1` has `rwx`, but mask `rw-` reduces effective permissions to
`rw-`.

### 🔹 Mask behavior:

-   Automatically recalculated when ACL entries are added/modified/removed.
-   Inherited by new files created in a directory with default ACLs.
-   Acts as a *permission ceiling* for applicable entries.

### 🔍 Full ACL Example (Step-by-Step)

#### 1️⃣ Start — File with no ACL entries
```bash 
# Create file & Check ACL
touch file.txt
getfacl file.txt
```
```ini 
# file: file.txt
# ...
user::rw-
group::r--
other::r--
```

#### 2️⃣ Add a named user with read-write access
```bash
setfacl -m u:ali:rw file.txt
getfacl file.txt
```
```ini
user::rw-
user:ali:rw-          
group::r--
mask::rw-
other::r--
```
> ✔ Mask appears automatically because you added ACL entries.

#### 3️⃣ Add write-only permission for user owner
```bash
setfacl -m u::w file.txt
getfacl file.txt
```

#### 4️⃣ Add mask manually
```bash
setfacl -m m::r file.txt
```
```ini
user::-w-
user:ali:rw-       #effective:r--   
group::r--
mask::r--
other::r--
```

#### 5️⃣ Add multiple ACL changes in one command (comma separated)
```bash
setfacl -m u:ali:rwx,g:devs:r--,m::rw file.txt
```
```ini
user::-w-
user:ali:rwx       #effective:rw-   
group::r--
group:devs:r--
mask::rw-
other::r--
```


### 🔹 Example --- Multiple group memberships
```bash
    setfacl -m g:group01:r,g:group02:rw reports.txt
```
> ✔ If a user belongs to both groups → gets most permissive allowed by mask (rw).

### 🔹 Mask Recalculation Options

    setfacl -m u:ali:r file.txt       # mask recalculated (default)
    setfacl -n -m u:ali:r file.txt    # no mask recalculation


## ACL Backup and Restore>
### Backup:

    getfacl test.txt > acl.txt

### Restore:

    setfacl --set-file=acl.txt file1

### Single pipeline:

    getfacl test.txt | setfacl --set-file=- file1
    
> **Note:**
>- **Copy normally (ACL is LOST)**\
    `setfacl -m u:ali:rw file1` \
    `cp file1 file2` \
    `getfacl file2` \
❌ file2 will not have u:ali:rw because normal cp does NOT copy ACLs.
>- **Correct way: Preserve ACLs** \
    `cp -p file1 file2`\
    `getfacl file2`\
    ✔ You will see u:ali:rw\
    ✔ Permissions, owner, timestamps, and ACLs preserved.
## Removing ACLs

    setfacl -x u:ali test.txt                # remove specific entry
    setfacl -x u:ali,g:group1 test.txt       # remove multiple entries
    setfacl -b test.txt                      # remove ALL ACLs (rollback to default)

## Default ACLs (`-d`)
-   Default ACLs = Inheritance rules for NEW files and directories.
-   They do NOT affect the directory itself, and they do NOT affect existing files inside it.

### Example:

    setfacl -d -m u:ali:rw,g:devs:rw,o::r project/
    getfacl project/

```ini
    # file: project/
    # owner: root
    # group: root
    user::rwx
    group::r-x
    other::r-x
    default:user::rwx
    default:user:ali:rw-
    default:group::r-x
    default:group:devs:rw-
    default:mask::rwx
    default:other::r--
```

✔ It adds default ACLs (inheritance rules) to the directory `project/` \
✔ It affects NEW files created inside `project/`:
-    Ali will get: rw
-    devs group gets: rw
-    others get: r

✔ It affects NEW directories created inside `project/`:
-    Ali → rwx  (auto-add x directories always need x to enter)
-    devs → rwx
-   others → rx

❌ It does NOT give permissions to Ali on the `project/` directory itself \
Ali cannot:
-   enter the directory
-   read inside it
-   write inside it

> Unless you ALSO run a normal ACL: 

    setfacl -m u:ali:rwx project/

❌ It does NOT give permissions to devs group on `project/` itself.\
❌ It does NOT modify any existing files inside the directory.\
> Only newly created files/folders inherit these permissions.

## Recursive ACLs (`-R`)

Applies ACLs to directories *and all existing contents*.

### Example:

    setfacl -R -m u:ali:rwx project/

### Effect: 
- Updates ACL for `project/` and everything inside it.
- recursive affects only existing files and directories, not future ones.
------------------------------------------------------------------------
