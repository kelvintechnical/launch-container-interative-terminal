# Lab 22-1f: Launch Container with Port Mapping

**RHCSA EX200 Lab** | Series: Container Management (Lab 22)  
**Prerequisite:** Lab 22-1e complete (ubi9 image pulled locally)  
**Time Estimate:** ~10 minutes

---

## 🎯 Objective

Launch a rootful ubi9 container with an interactive terminal and a port mapping, then explore the container environment to confirm it is a fully isolated RHEL 9.8 instance.

---

## 📚 Command Decision Map

| Task | Command |
|---|---|
| Launch interactive container | `sudo podman run -it --name <name> -p <host>:<container> <image> /bin/bash` |
| List files in container | `ls` |
| Print working directory | `pwd` |
| Check disk mounts | `df` |
| Confirm OS version | `cat /etc/redhat-release` |
| View full OS metadata | `cat /etc/os-release` |
| Exit container shell | `exit` |

---

## 🧠 Concept: What Does `podman run` Do?

`podman run` creates a **new container** from a local image and starts it. It is the primary way to go from a stored image to a running process.

```
Local image (podman images)
        ↓
  podman run
        ↓
Running container (isolated process + filesystem)
```

| `podman` subcommand | What it operates on |
|---|---|
| `pull` | Registry → local image storage |
| `run` | Local image → new running container |
| `exec` | Existing running container (enter it) |
| `stop` / `rm` | Manage lifecycle of running/stopped containers |

---

## 🧠 Concept: Port Mapping (`-p`)

Containers run in an isolated network namespace — they can't be reached from the host by default. Port mapping punches a hole between the host network and the container network.

```
Host network         Container network
  port 80      ←→      port 8080
```

| Flag | Syntax | Meaning |
|---|---|---|
| `-p` | `-p <host_port>:<container_port>` | Forward traffic from host port to container port |
| Example | `-p 80:8080` | Requests to host port 80 are forwarded to port 8080 inside the container |

> **Why different ports?** The host port (80) is what external clients connect to. The container port (8080) is what the application inside the container listens on. They don't need to match — the mapping translates between them.

---

## 🔧 Steps

### Step 1 — First attempt (incorrect image path)

```bash
sudo podman run -it --name --rootful-cont-port -p 80:8080 \
  registry.access.redhat.com/ubi:latest /bin/bash
```

**Actual output:**

```
Trying to pull registry.access.redhat.com/ubi:latest...
Error: unable to copy from source docker://registry.access.redhat.com/ubi:latest:
initializing source docker://registry.access.redhat.com/ubi:latest: reading manifest
latest in registry.access.redhat.com/ubi: name unknown: Repo not found
```

#### Output explained

| Part of error | Meaning |
|---|---|
| `Trying to pull registry.access.redhat.com/ubi:latest...` | The image wasn't found locally, so podman tried to pull it from the registry |
| `name unknown: Repo not found` | The path `ubi` does not exist at that registry — the correct path is `ubi9/ubi` |

#### Two bugs in this command

| Bug | What was typed | Correct form |
|---|---|---|
| Wrong image path | `registry.access.redhat.com/ubi:latest` | `registry.access.redhat.com/ubi9/ubi:latest` |
| Bad container name | `--name --rootful-cont-port` | `--name rootful-cont-port` (no leading `--` in the value) |

> `--name` takes a plain string as its value. Writing `--name --rootful-cont-port` passes `--rootful-cont-port` as the name, which starts with `--` and may be misinterpreted. Always use plain alphanumeric names with hyphens: `rootful-cont-port`.

---

### Step 2 — Launch container (corrected command)

```bash
sudo podman run -it --name rootful-cont-port -p 80:8080 \
  registry.access.redhat.com/ubi9/ubi:latest /bin/bash
```

#### Full command breakdown

| Part | Meaning |
|---|---|
| `sudo podman run` | Create and start a new container as root (rootful mode) |
| `-i` | **Interactive** — keep STDIN open so you can type commands |
| `-t` | **TTY** — allocate a pseudo-terminal so the shell renders correctly |
| `-it` | Combined: gives you an interactive shell session inside the container |
| `--name rootful-cont-port` | Assigns a human-readable name — easier than using the container ID |
| `-p 80:8080` | Map host port 80 → container port 8080 |
| `registry.access.redhat.com/ubi9/ubi:latest` | The image to launch from (already stored locally from Lab 22-1e) |
| `/bin/bash` | The process to run inside the container — starts a Bash shell |

**Actual output — prompt change:**

```
[root@57cdc653dd8a /]#
```

| Part | Meaning |
|---|---|
| `root` | You are the root user **inside** the container — not the host root |
| `57cdc653dd8a` | The container's hostname — automatically set to the first 12 chars of the container ID |
| `/` | Current directory is `/` (container root filesystem) |
| `#` | Root shell prompt — confirms elevated access inside the container |

> **You are now inside the container.** Every command you run executes in an isolated environment — separate filesystem, separate process table, separate network namespace. Changes here do not affect the host.

---

### Step 3 — Explore the container filesystem

```bash
ls
```

**Actual output:**

```
afs  boot  etc   lib    media  opt   root  sbin  sys  usr
bin  dev   home  lib64  mnt    proc  run   srv   tmp  var
```

#### Output explained

This is the **root filesystem of a minimal RHEL 9.8 container** — not your EC2 host. The directory structure follows the Filesystem Hierarchy Standard (FHS):

| Directory | Purpose |
|---|---|
| `bin`, `sbin` | Essential binaries and system binaries |
| `etc` | Configuration files |
| `lib`, `lib64` | Shared libraries |
| `usr` | User programs, docs, and libraries |
| `var` | Variable data (logs, spool, runtime) |
| `tmp` | Temporary files |
| `root` | Root user's home directory |
| `proc`, `sys`, `dev` | Virtual filesystems — kernel/hardware interfaces |
| `home` | Empty — no regular users created in this base image |
| `afs` | AFS (Andrew File System) mount point — present in RHEL base, empty here |

> **Why no application files?** UBI is a **base image** — it contains only the OS layer. Application code, configs, and dependencies are added on top in derived images or bind mounts.

---

### Step 4 — Confirm working directory

```bash
ls pwd
```

**Actual output:**

```
ls: cannot access 'pwd': No such file or directory
```

> This was a typo — `ls pwd` tried to list a file named `pwd` rather than running the `pwd` command. The correct command is just `pwd`.

```bash
pwd
```

**Actual output:**

```
/
```

You are at the root of the container's filesystem. This is the default starting directory for a `/bin/bash` entrypoint with no `WORKDIR` set in the image.

---

### Step 5 — Check disk mounts

```bash
df
```

**Actual output:**

```
Filesystem     1K-blocks    Used Available Use% Mounted on
overlay         10213356 2191080   8022276  22% /
tmpfs              65536       0     65536   0% /dev
tmpfs             185756    3696    182060   2% /etc/hosts
shm                64000       0     64000   0% /dev/shm
devtmpfs          427636       0    427636   0% /proc/keys
```

#### Output explained

| Filesystem | Mounted on | Meaning |
|---|---|---|
| `overlay` | `/` | The container's root filesystem uses **OverlayFS** — layers the read-only image on top of a writable layer. Changes you make are stored in the writable layer only. |
| `tmpfs` on `/dev` | `/dev` | Device filesystem — virtual, exists only in memory |
| `tmpfs` on `/etc/hosts` | `/etc/hosts` | Host-injected hostname resolution file — podman mounts this from the host |
| `shm` | `/dev/shm` | Shared memory — 64MB allocated for inter-process communication |
| `devtmpfs` | `/proc/keys` | Kernel key retention service interface |

> **OverlayFS** is the key concept here. The base image layers are **read-only**. When you write a file inside the container, it goes into a thin **writable layer** on top. When the container is deleted, that writable layer is discarded — the original image is untouched. This is how podman can run hundreds of containers from a single image without copying it each time.

---

### Step 6 — Confirm OS version

```bash
cat /etc/redhat-release
```

**Actual output:**

```
Red Hat Enterprise Linux release 9.8 (Plow)
```

| Part | Meaning |
|---|---|
| `Red Hat Enterprise Linux` | Confirms this is a genuine RHEL userland, not CentOS or Fedora |
| `release 9.8` | RHEL minor version — matches the `version: 9.8` from `skopeo inspect` in Lab 22-1d ✅ |
| `(Plow)` | RHEL 9.8 codename |

---

### Step 7 — View full OS metadata

```bash
cat /etc/os-release
```

**Actual output:**

```
NAME="Red Hat Enterprise Linux"
VERSION="9.8 (Plow)"
ID="rhel"
ID_LIKE="fedora"
VERSION_ID="9.8"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Red Hat Enterprise Linux 9.8 (Plow)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:redhat:enterprise_linux:9::baseos"
HOME_URL="https://www.redhat.com/"
BUG_REPORT_URL="https://issues.redhat.com/"
REDHAT_BUGZILLA_PRODUCT="Red Hat Enterprise Linux 9"
REDHAT_SUPPORT_PRODUCT_VERSION="9.8"
```

#### Key fields explained

| Field | Value | Meaning |
|---|---|---|
| `ID="rhel"` | `rhel` | Machine-readable OS identifier — scripts use this to detect RHEL |
| `ID_LIKE="fedora"` | `fedora` | RHEL is derived from Fedora — shares package format, tooling conventions |
| `VERSION_ID="9.8"` | `9.8` | Programmatic version string — used by package managers and CI tools |
| `PLATFORM_ID="platform:el9"` | `el9` | Enterprise Linux 9 platform — determines available module streams in dnf |
| `CPE_NAME` | `cpe:/o:redhat:...` | Common Platform Enumeration — standardized identifier used in CVE/security databases |
| `HOME_URL` | redhat.com | Official documentation source |

> `/etc/os-release` is the **standard way for scripts and tools to detect the OS** inside a container or VM. `cat /etc/redhat-release` is human-readable; `/etc/os-release` is machine-readable. Know both for the exam.

---

## ✅ Lab Checklist

- [ ] First `podman run` failed with `Repo not found` — bugs identified and fixed
- [ ] Corrected command launches container successfully
- [ ] Prompt changes to `[root@<container-id> /]#` ✅
- [ ] `ls` shows standard RHEL FHS directory structure ✅
- [ ] `pwd` returns `/` ✅
- [ ] `df` shows `overlay` as root filesystem ✅
- [ ] `cat /etc/redhat-release` confirms RHEL 9.8 ✅
- [ ] `cat /etc/os-release` confirms `VERSION_ID="9.8"` ✅

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| Wrong image path (`ubi` instead of `ubi9/ubi`) | `Repo not found` | Use full path: `registry.access.redhat.com/ubi9/ubi:latest` |
| `--name --value` with leading dashes | Container name error | Use plain names: `--name rootful-cont-port` |
| Missing `-it` | Container exits immediately | `/bin/bash` needs `-it` to stay open interactively |
| `ls pwd` instead of `pwd` | `No such file or directory` | These are separate commands — run `pwd` alone |
| `exit` not run | Still inside container | Type `exit` to return to host shell |

---

## 📌 RHCSA Exam Strategy

- `-it` is almost always required when using `/bin/bash` as the container entrypoint — missing it causes the container to exit immediately.
- **Container hostname = container ID** — the prompt `[root@57cdc653dd8a /]#` is normal, not an error.
- `overlay` in `df` output confirms OverlayFS is active — the expected storage driver for rootful podman.
- `/etc/os-release` is the authoritative machine-readable OS identifier — preferred over `/etc/redhat-release` for scripting.
- Port mapping is **host:container** — left side is always the host port.

---

## ➡️ Next Lab

**[Lab 22-1g: Run commands inside container](https://github.com/kelvintechnical/run-commands-inside-terminal)**

---

## 🔗 Series Index

| Lab | Topic |
|---|---|
| ✅ [22-1a](https://github.com/kelvintechnical/Create-User-Account-Conadm) | Create user `conadm` |
| ✅ [22-1b](https://github.com/kelvintechnical/Grant-Conadm-Full-Rights) | Grant `conadm` full sudo rights |
| ✅ [22-1c](https://github.com/kelvintechnical/Verify-Sudo-Access-Conadm) | Verify sudo access |
| ✅ [22-1d](https://github.com/kelvintechnical/Inspect-ubi9-with-skopeo) | Inspect ubi9 image with skopeo |
| ✅ [22-1e](https://github.com/kelvintechnical/Pull-ubi9-Image-with-podman) | Pull ubi9 image with podman |
| 👉 **22-1f** | Launch container with port mapping ← *you are here* |
| [22-1g](https://github.com/kelvintechnical/run-commands-inside-terminal) | Run commands inside container |
| [22-1h](https://github.com/kelvintechnical/verify-port-mapping-from-host) | Verify port mapping from host |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
