# Multi-Container Runtime

## 1. Team Information

| Name | SRN |
|------|-----|
| Sriram K | PES1UG24CS468 |
| Soumik Sinha | PES1UG24CS463 |

---

## 2. Build, Load, and Run Instructions

### Prerequisites
- Ubuntu 22.04 or 24.04 VM
- Secure Boot OFF
- No WSL

### Install Dependencies
```bash
sudo apt update
sudo apt install -y build-essential linux-headers-$(uname -r)
```

### Build
```bash
cd boilerplate
make
```

### Prepare Root Filesystem
```bash
mkdir rootfs-base
wget https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
cp -a ./rootfs-base ./rootfs-alpha
cp -a ./rootfs-base ./rootfs-beta
```

### Load Kernel Module
```bash
sudo insmod monitor.ko
ls -l /dev/container_monitor
sudo dmesg | tail -5
```

### Start Supervisor
```bash
sudo ./engine supervisor ./rootfs-base
```

### Launch Containers (in a second terminal)
```bash
sudo ./engine start alpha ./rootfs-alpha "/bin/sh -c 'while true; do echo hello from alpha; sleep 2; done'"
sudo ./engine start beta ./rootfs-beta "/bin/sh -c 'while true; do echo hello from beta; sleep 2; done'"
```

### Use the CLI
```bash
# List containers
sudo ./engine ps

# View logs
sudo ./engine logs alpha

# Stop a container
sudo ./engine stop alpha

# Run a container and wait for it to exit
sudo ./engine run alpha ./rootfs-alpha "/bin/sh -c 'echo done'"
```

### Run Memory Test
```bash
cp memory_hog ./rootfs-alpha/
sudo ./engine start memtest ./rootfs-alpha "/memory_hog" --soft-mib 10 --hard-mib 20
sleep 10
sudo dmesg | grep container_monitor
```

### Run Scheduling Experiment
```bash
cp cpu_hog ./rootfs-alpha/
cp cpu_hog ./rootfs-beta/
sudo ./engine start cpuhi ./rootfs-alpha "/cpu_hog" --nice -10
sudo ./engine start cpulo ./rootfs-beta "/cpu_hog" --nice 10
top
```

### Cleanup
```bash
sudo ./engine stop alpha
sudo ./engine stop beta
sudo rmmod monitor
sudo dmesg | tail -5
```

---

## 3. Demo Screenshots

### Screenshot 1 — Multi-container supervision
![Multi-container](boilerplate/Screenshot1a.png)
![Multi-container](boilerplate/Screenshot1b.png)
*Two containers alpha and beta running concurrently under one supervisor process*

### Screenshot 2 — Metadata tracking
![Metadata](boilerplate/Screenshot1b.png)
*Output of engine ps showing container IDs, host PIDs, and current states*

### Screenshot 3 — Bounded-buffer logging
![Logging](boilerplate/Screenshot2.png)
*Log file contents captured through the bounded-buffer logging pipeline*

### Screenshot 4 — CLI and IPC
![CLI](boilerplate/Screentshot1b.png)
*CLI stop command issued and supervisor responding, demonstrating UNIX domain socket IPC*

### Screenshot 5 — Soft-limit warning
![Soft limit](boilerplate/Screentshot5.png)
*dmesg showing SOFT LIMIT warning when container RSS exceeds soft threshold*

### Screenshot 6 — Hard-limit enforcement
![Hard limit](boilerplate/Screenshot6.png)
*dmesg showing container killed after exceeding hard limit, supervisor metadata updated to killed state*

### Screenshot 7 — Scheduling experiment
![Scheduling](boilerplate/Screenshot7a.png)
![Scheduling](boilerplate/Screenshot7b.png)
*top output showing cpuhi (nice -10) receiving more CPU time than cpulo (nice +10)*

### Screenshot 8 — Clean teardown
![Teardown](boilerplate/Screenshot8.png)
*No zombie processes after shutdown, kernel module unloaded cleanly*

---

## 4. Engineering Analysis

### 1. Isolation Mechanisms

The runtime achieves isolation using three Linux namespaces created via the `clone()` system call with `CLONE_NEWPID`, `CLONE_NEWUTS`, and `CLONE_NEWNS` flags.

**PID namespace** gives each container its own process ID space. The first process inside the container sees itself as PID 1, completely unaware of host processes. This prevents containers from seeing or signaling each other's processes.

**UTS namespace** allows each container to have its own hostname, set via `sethostname()` inside `child_fn`. This is why running `hostname` inside a container returns the container ID rather than the host machine name.

**Mount namespace** isolates the filesystem view. Combined with `chroot()`, the container sees only its assigned rootfs directory as `/`. The host kernel still mounts `/proc` inside the container after `chroot()` so tools like `ps` work correctly inside the container.

What the host kernel still shares with all containers: the same kernel, the same network stack (we don't use `CLONE_NEWNET`), and the same system time. Namespaces virtualize the view, but the underlying kernel resources are shared.

### 2. Supervisor and Process Lifecycle

A long-running supervisor is necessary because containers are child processes — when a child exits, only its parent can reap it with `wait()`. Without a persistent parent, exited children become zombies consuming PID table entries indefinitely.

The supervisor uses `clone()` instead of `fork()` to create containers with namespace isolation. It maintains a linked list of `container_record_t` structs tracking each container's ID, host PID, start time, state, memory limits, and log path.

`SIGCHLD` is delivered to the supervisor whenever a container exits. The handler calls `waitpid(-1, &status, WNOHANG)` in a loop to reap all exited children without blocking. It then updates the container's state to `CONTAINER_EXITED` or `CONTAINER_KILLED` depending on whether it exited normally or was signaled.

`SIGINT` and `SIGTERM` to the supervisor trigger orderly shutdown: the bounded buffer drain is initiated, the logger thread is joined, all file descriptors are closed, and the UNIX socket file is removed.

### 3. IPC, Threads, and Synchronization

The project uses two distinct IPC mechanisms:

**Path A — Logging (pipes):** Each container's stdout and stderr are redirected to the write end of a pipe. A dedicated pipe-reader thread per container reads from the read end and pushes chunks into the bounded buffer. This is the producer side. The logger thread is the consumer — it pops chunks and writes them to per-container log files.

The bounded buffer uses a `pthread_mutex_t` to protect the shared buffer state (head, tail, count) and two `pthread_cond_t` variables (`not_empty`, `not_full`) to block producers when full and consumers when empty. A mutex is the right choice here because the critical sections involve multiple field updates that must be atomic together. A spinlock would waste CPU since producers and consumers may block for extended periods.

**Path B — Control (UNIX domain socket):** CLI client processes connect to `/tmp/mini_runtime.sock`, send a `control_request_t` struct, and receive a `control_response_t`. The supervisor's event loop calls `accept()` in a loop. The container metadata linked list is protected by a separate `pthread_mutex_t` (`metadata_lock`) since it is accessed by both the event loop and the SIGCHLD handler.

### 4. Memory Management and Enforcement

RSS (Resident Set Size) measures the amount of physical RAM currently occupied by a process's pages. It does not measure memory that has been allocated but not yet touched (due to lazy allocation), memory mapped but swapped out, or shared library pages counted multiply.

Soft and hard limits implement two different enforcement policies. The soft limit triggers a warning — it signals that a container is approaching its budget without killing it, giving the operator time to react. The hard limit enforces an absolute ceiling by sending `SIGKILL` to the container process.

Enforcement belongs in kernel space because user-space cannot reliably measure or act on another process's memory usage. A user-space monitor could be starved of CPU, delayed by scheduling, or simply too slow — a container could blow past its limit between checks. The kernel timer runs at a fixed interval regardless of user-space scheduling, and `send_sig(SIGKILL, task, 1)` from kernel space is immediate and cannot be caught or ignored by the target process.

### 5. Scheduling Behavior

Linux uses the Completely Fair Scheduler (CFS) which assigns CPU time based on a virtual runtime counter. Processes with lower nice values (higher priority) accumulate virtual runtime more slowly, so CFS picks them more often to maintain fairness in virtual time.

In our experiment, `cpuhi` (nice -10) consistently received significantly more CPU time than `cpulo` (nice +10) as observed in `top`. This is because CFS applies a weight multiplier based on nice value — nice -10 has roughly 9x the weight of nice 0, while nice +10 has roughly 0.11x the weight. The difference in CPU share between -10 and +10 is therefore very large.

For the CPU-bound vs I/O-bound comparison, the I/O-bound container (`io_pulse`) received CPU time quickly whenever it became runnable because CFS rewards processes that sleep frequently with a shorter scheduling lag. This demonstrates CFS's built-in preference for interactive and I/O-bound workloads over pure CPU hogs.

---

## 5. Design Decisions and Tradeoffs

### Namespace Isolation
**Choice:** Used `CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS` without network namespace.
**Tradeoff:** Containers share the host network stack, so port conflicts are possible.
**Justification:** Network namespace requires additional veth pair setup which is outside the project scope. PID, UTS, and mount isolation are sufficient for the required demonstrations.

### Supervisor Architecture
**Choice:** Single-threaded event loop using blocking `accept()` for the control plane.
**Tradeoff:** One slow CLI command blocks others from being processed simultaneously.
**Justification:** Simplifies synchronization significantly. Container workloads are the performance-critical path — the control plane handles infrequent human commands where millisecond latency is acceptable.

### IPC and Logging
**Choice:** UNIX domain socket for control, pipes for logging, bounded buffer with mutex/condvar.
**Tradeoff:** Pipes are unidirectional so a separate socket is needed for the control channel.
**Justification:** UNIX domain sockets provide reliable, ordered, connection-oriented communication with built-in backpressure. The bounded buffer with condvars avoids busy-waiting and handles backpressure naturally when the logger falls behind.

### Kernel Monitor
**Choice:** Mutex instead of spinlock for the monitored list.
**Tradeoff:** Mutex has slightly higher overhead than a spinlock for very short critical sections.
**Justification:** The timer callback iterates the entire list and calls `get_mm_rss()` per entry — this is not a short critical section. A spinlock would hold off other CPUs for too long. A mutex allows the kernel to schedule other work while waiting.

### Scheduling Experiments
**Choice:** Used `setpriority()` with nice values rather than cgroups CPU shares.
**Tradeoff:** Nice values affect the whole process tree; cgroups would give finer per-container control.
**Justification:** Nice values are directly observable in `top` and map cleanly to CFS weight theory, making the experiment results easier to explain and verify.

---

## 6. Scheduler Experiment Results

### Experiment 1 — CPU-bound vs CPU-bound with different priorities

| Container | Nice value | Observed CPU% |
|-----------|-----------|---------------|
| cpuhi | -10 | ~75-80% |
| cpulo | +10 | ~20-25% |

**Analysis:** CFS weight for nice -10 is approximately 9544, while nice +10 is approximately 110. The ratio is roughly 87:1 in weight, but since both processes compete on a single core the actual split converges toward the weight ratio up to the physical limit. The high-priority container receives approximately 3-4x more CPU time, consistent with CFS theory.

### Experiment 2 — CPU-bound vs I/O-bound

| Container | Workload | Observed behavior |
|-----------|----------|-------------------|
| alpha | cpu_hog | Consistently high CPU%, long scheduling intervals |
| beta | io_pulse | Low average CPU%, but gets CPU immediately when I/O completes |

**Analysis:** CFS tracks `vruntime` — processes that sleep accumulate less virtual runtime, so they are scheduled ahead of CPU hogs when they wake up. The I/O-bound container had lower average CPU usage but better responsiveness, demonstrating CFS's built-in preference for interactive workloads. This is the intended behavior of CFS to balance throughput (CPU hog) with responsiveness (I/O bound).
