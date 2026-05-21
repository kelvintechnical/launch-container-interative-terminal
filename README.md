# Lab 22-1f — Launch Container with `-it` + Port Map 8080:80

**RHCSA EX200 Lab** | Part of [Linux Ops Mastery](https://github.com/kelvintechnical/linux-ops-mastery)

---

## 📋 Scenario

You are working as **conadm** on Node1. The ubi9 image has already been pulled. Your task is to launch a container interactively with a mapped port so the container's internal port 80 is accessible on host port 8080.

---

## 🎯 Requirements

1. Run the container as **conadm** (non-root)
2. Launch with an interactive terminal (`-it`)
3. Name the container `mycontainer`
4. Map host port `8080` → container port `80`
5. Start a bash shell inside the container

---

## 🧠 Key Concepts

| Flag | Purpose |
|------|---------|
| `podman run` | Create and start a new container |
| `-i` | Keep stdin open (interactive) |
| `-t` | Allocate a pseudo-TTY (terminal) |
| `--name mycontainer` | Assign a referenceable name instead of a random ID |
| `-p 8080:80` | Map host port 8080 → container port 80 (host:container) |
| `/bin/bash` | Command to run inside the container on start |

---

## 🔬 Steps

### Step 1 — Switch to conadm

```bash
su - conadm
```

### Step 2 — Confirm image is available

```bash
podman images
```

**Expected output:**
```
REPOSITORY                                    TAG     IMAGE ID      CREATED      SIZE
registry.access.redhat.com/ubi9/ubi           latest  abc123...     2 weeks ago  214 MB
```

### Step 3 — Launch the container

```bash
podman run -it --name mycontainer -p 8080:80 registry.access.redhat.com/ubi9/ubi /bin/bash
```

**Expected output:**
```
[root@a3f91bc72d10 /]#
```

> ✅ The prompt changes — you are now **inside the container**.  
> The hash after `root@` is the container ID, not your hostname.

---

## 🔍 Command Breakdown

```
podman run -it --name mycontainer -p 8080:80 registry.access.redhat.com/ubi9/ubi /bin/bash
│           │   │                  │           │                                    │
│           │   │                  │           │                                    └─ Command to run inside
│           │   │                  │           └─ Image to use
│           │   │                  └─ Port mapping: host:container
│           │   └─ Human-readable container name
│           └─ Interactive + TTY
└─ Create and start container
```

---

## ⚠️ Pitfalls

- **Port order matters** — `-p 8080:80` means host:container. Reversing it breaks access from the host
- **Forgetting `-it`** — without it, the container starts and immediately exits (no shell to attach to)
- **Running as root** — RHCSA expects rootless containers via `conadm`; always verify with `whoami` before running
- **Name conflicts** — if `mycontainer` already exists, `podman run` fails; remove it first with `podman rm mycontainer`

---

## 🎓 Exam Tip

> On the RHCSA exam, if a question says "launch a container" with a port mapping, the format is always `-p HOST_PORT:CONTAINER_PORT`. Memorize this order — host comes first.

---

## ✅ Lab Checklist

- [ ] Switched to conadm user
- [ ] Confirmed ubi9 image is present with `podman images`
- [ ] Launched container with `-it`, `--name`, and `-p` flags
- [ ] Shell prompt changed to container ID
- [ ] Ready to run commands inside container (Lab 22-1g)

## 🔗 Series Navigation
| Lab | Description |
|-----|-------------|
| [22-1a](https://github.com/kelvintechnical/Create-User-Account-Conadm) | Create conadm user |
| [22-1b](https://github.com/kelvintechnical/Grant-Conadm-Full-Rights) | Grant conadm full sudo rights |
| [22-1c](https://github.com/kelvintechnical/Verify-Sudo-Access-Conadm) | Verify sudo access |
| [22-1d](https://github.com/kelvintechnical/Inspect-ubi9-with-skopeo) | Inspect ubi9 remotely with skopeo |
| [22-1e](https://github.com/kelvintechnical/Pull-ubi9-Image-with-podman) | Pull ubi9 image with podman |
| **22-1f** | **Launch container with -it + port map 80:8080** ← you are here |
| [22-1g](https://github.com/kelvintechnical/run-commands-inside-terminal) | Run basic commands inside container |
| [22-1h](https://github.com/kelvintechnical/verify-port-mapping-from-host) | Verify port mapping from host |

---

## 👤 Author

**Kelvin R. Tobias** — [kelvinintech.com](https://kelvinintech.com) | [GitHub](https://github.com/kelvintechnical) | [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
