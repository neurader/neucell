<div align="center">

<br/>

```
███╗   ██╗███████╗██╗   ██╗ ██████╗███████╗██╗     ██╗
████╗  ██║██╔════╝██║   ██║██╔════╝██╔════╝██║     ██║
██╔██╗ ██║█████╗  ██║   ██║██║     █████╗  ██║     ██║
██║╚██╗██║██╔══╝  ██║   ██║██║     ██╔══╝  ██║     ██║
 ██║ ╚████║███████╗╚██████╔╝╚██████╗███████╗███████╗███████╗
╚═╝  ╚═══╝╚══════╝ ╚═════╝  ╚═════╝╚══════╝╚══════╝╚══════╝
```

**Install runtimes in Cells, not on your server.**

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Go Version](https://img.shields.io/badge/go-1.22+-00ADD8.svg)](https://golang.org)
[![Status](https://img.shields.io/badge/status-early%20development-orange.svg)]()
[![Org](https://img.shields.io/badge/org-neurader-1A1A2E.svg)](https://github.com/neurader)

<br/>

</div>

---

## The Problem

Every server you manage looks like this:

```
$ ps aux
java       PID  312MB  idle since 3 days ago
python3    PID  180MB  idle since 6 hours ago
node       PID   95MB  idle since yesterday
terraform  PID  210MB  idle since last week
```

None of them are doing anything. All of them are consuming memory. All of them have config files scattered across `/etc/`, `~/.config/`, and your home directory. Version conflicts accumulate over time. The server becomes harder to reason about.

**Why should Java exist in your server's memory at 2am when nobody is using it?**

The answer today is: because that is how Linux works. You install it, it stays.

NeuCell changes that.

---

## What is NeuCell

NeuCell is a **Cell-based binary runtime** — a single static Go binary that lets you deploy any runtime or tool into an isolated environment called a **Cell**.

A Cell is not a container. It is not a VM. It is a minimal Linux namespace-based isolation primitive.

```
$ neucell deploy java:21
$ neucell deploy python:3.12
$ neucell deploy ansible:2.17

$ ps aux
neucell    PID   8MB   ← only this. nothing else.
```

Activate when you need it:

```
$ neucell activate java

(java-cell) $ java -version
java version "21.0.1" 2023-10-17

(java-cell) $ java -jar myapp.jar

(java-cell) $ exit
→ Cell scales to zero. Java is gone from memory.
```

---

## How it Works

### Scale to Zero

When a Cell is at zero — **the process does not exist**. This is not sleep mode. This is not pause. The process is gone. The kernel has nothing to report. `top`, `ps`, and `free` show nothing.

The **Cell definition** persists on disk (a small `cell.toml` file). When you need it again, the Cell is rebuilt from its blueprint in milliseconds — with all your configs, packages, and environment variables exactly as you left them.

| | Uninstall | Cell at Zero |
|---|---|---|
| Memory freed | ✅ | ✅ |
| Reinstall needed | ✅ | ❌ |
| Config preserved | ❌ | ✅ |
| Ready in milliseconds | ❌ | ✅ |

### Isolation

Each Cell runs in its own Linux namespaces:

- `CLONE_NEWPID` — isolated PID space
- `CLONE_NEWNET` — own network namespace with a `veth` pair
- `CLONE_NEWNS` — mount namespace, bind-mounted configs
- `CLONE_NEWUTS` — own hostname

The tool inside the Cell sees a completely normal Linux environment. It has no idea it is in a Cell. Config files appear exactly where the tool expects them. Network works normally. Nothing is wrapped or proxied.

### No Heavy Dependencies

| | Docker | containerd | NeuCell |
|---|---|---|---|
| Daemon required | dockerd + containerd | containerd | ~8MB daemon only |
| Image format | OCI layers | OCI layers | Raw binary |
| Runtime spec | OCI spec | OCI spec | None |
| Idle memory | ~150MB | ~80MB | **~8MB** |
| Install | `apt install docker-ce` | complex | single binary |

---

## Cell Configuration

Every Cell is defined by a `cell.toml` — its blueprint. This file is all that exists when the Cell is at zero.

```toml
[cell]
name    = "java"
version = "21.0.1"
binary  = "~/.neucell/cells/java/bin/java"

[network]
ip     = "10.10.0.5"
bridge = "neucell0"

[mounts]
"~/.neucell/cells/java/conf" = "/etc/java"
"~/.neucell/cells/java/home" = "/root/.java"

[env]
JAVA_HOME = "/usr/lib/jvm/java-21"
PATH      = "/usr/lib/jvm/java-21/bin:/usr/local/bin:/usr/bin"

[lifecycle]
idle_timeout = 300  # seconds before auto scale-to-zero
```

All Cell files live in one directory:

```
~/.neucell/cells/
  java/
    cell.toml       ← blueprint (survives scale-to-zero)
    bin/            ← java binary
    conf/           ← all java config files
    home/           ← ~/.java equivalent
  python/
    cell.toml
    bin/
    packages/       ← pip installed packages
    conf/
  ansible/
    cell.toml
    bin/
    conf/           ← ansible.cfg, inventory
    collections/
    roles/
```

Backup one folder. Move it to any server. Restore instantly.

---

## CLI

```bash
# Deploy a runtime into a Cell
neucell deploy java:21
neucell deploy python:3.12
neucell deploy ansible:2.17

# Activate / deactivate
neucell activate java
neucell deactivate java

# List all Cells and their state
neucell list

# Edit Cell configuration
neucell config edit ansible

# Portability
neucell backup java          # export as archive
neucell export java          # migrate to another server
neucell import java.tar.gz   # restore from archive

# Remove
neucell remove java
```

---

## NeuRader Integration

NeuCell is designed to work alongside **[NeuRader](https://github.com/neurader/neurader)** — the lightweight Ansible execution monitoring controller.

Together they form a complete stack:

```
NeuRader Controller
  └── monitors all managed nodes
  └── triggers Cell activation automatically
  └── receives alerts from monitoring-cell

NeuCell (on each managed node)
  └── monitoring-cell    ← always on, watches /proc metrics
  └── java-cell          ← scaled to zero
  └── ansible-cell       ← scaled to zero
  └── terraform-cell     ← scaled to zero
```

### Automatic Activation

No human intervention needed. When NeuRader detects that a Cell is required — from an incoming command, a scheduled job, or a threshold breach — it tells NeuCell to activate. The Cell spins up, does its work, and returns to zero.

```
2am — all Cells at zero

Threshold breach: CPU > 90%
  → monitoring-cell detects it
  → NeuRader controller notified
  → Alert sent: Slack / PagerDuty / Jira
  → Cell back to zero
```

---

## What NeuCell Is Not

NeuCell has a clear, intentional scope.

- ❌ **Not an app deployment platform** — NeuCell provides the runtime, not the app
- ❌ **Not a container registry** — no images, no layers, no registry
- ❌ **Not a Kubernetes replacement** — built for single-server and small fleet use
- ❌ **Not a full OCI runtime** — intentionally lighter and more focused

NeuCell does one thing: give you **isolated, versioned, scale-to-zero runtime environments** for your tools.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Go — `CGO_ENABLED=0`, single static binary |
| Cell isolation | Linux namespaces via raw syscalls (`clone`, `unshare`) |
| Networking | `veth` pairs + `netlink` |
| Filesystem | bind mounts via mount namespace |
| State | `cell.toml` — plain TOML on disk |
| CLI | Cobra |
| Controller comms | HTTP + JSON |
| Binary size | Target ~10MB |
| Daemon memory | Target ~8MB idle |

---

## Roadmap

| Version | Scope |
|---|---|
| v0.1 | Core Cell runtime — namespace isolation, veth networking, bind mounts |
| v0.2 | `cell.toml` spec, Cell directory structure, NeuCell daemon |
| v0.3 | CLI — `deploy`, `activate`, `deactivate`, `list`, `config` |
| v0.4 | Scale-to-zero with idle timeout, auto-activation |
| v0.5 | NeuRader monitoring-cell integration, threshold push |
| v0.6 | Cell backup, export, import — portability |
| v0.7 | NeuRader controller integration — remote Cell management |
| v1.0 | Stable release — NeuRader + NeuCell full stack |

---

## Status

NeuCell is in **early design and initial implementation**. The architecture spec is published. Core runtime development is starting.

This is an independent open-source project by [@neurader](https://github.com/neurader). Feedback, ideas, and contributions are welcome.

---

## License

Apache 2.0 — see [LICENSE](LICENSE)

---

<div align="center">

**[neurader.cloud](https://neurader.cloud)** · **[NeuRader](https://github.com/neurader/neurader)** · **[NeuCell](https://github.com/neurader/neucell)**

</div>
