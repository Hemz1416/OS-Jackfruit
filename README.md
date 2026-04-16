# OS-Jackfruit: Multi-Container Runtime

A lightweight Linux container runtime written in C, featuring a persistent user-space supervisor daemon and a kernel-space memory monitor (LKM) for strict resource enforcement. 

This project demonstrates core Operating System concepts including namespace isolation, bounded-buffer concurrency, IPC, physical memory enforcement from Ring 0, and Completely Fair Scheduler (CFS) behavior.

---

## 1. Team Information
* **Team Member 1:** HITHA SHREE SURESH (SRN: PES2UG24CS196)
* **Team Member 2:** HEMANT RAM NELAPATI (SRN: PES2UG24CS192)

---

## 2. Build, Load, and Run Instructions

Follow these steps to reproduce the environment and run the runtime on a fresh Ubuntu 22.04 or 24.04 VM (Secure Boot disabled).

### Phase 1: Setup and Compilation
```bash
# Install dependencies
sudo apt update && sudo apt install -y build-essential linux-headers-$(uname -r)

# Enter directory and compile user-space and kernel module
cd boilerplate
make clean
make

# Load the memory monitor kernel module
sudo insmod monitor.ko

# Prepare isolated Alpine root filesystems
mkdir -p rootfs-base
wget -nc https://dl-cdn.alpinelinux.org/alpine/v3.20/releases/x86_64/alpine-minirootfs-3.20.3-x86_64.tar.gz
sudo tar -xzf alpine-minirootfs-3.20.3-x86_64.tar.gz -C rootfs-base
sudo cp -a ./rootfs-base ./rootfs-alpha
sudo cp -a ./rootfs-base ./rootfs-beta
```

### Phase 2: Running the Supervisor
Open **Terminal 1** and start the long-running supervisor daemon:
```bash
sudo ./engine supervisor ./rootfs-base
```

### Phase 3: Client Commands & Teardown
Open **Terminal 2** to issue commands via the CLI:
```bash
# Start containers
sudo ./engine start alpha ./rootfs-alpha "echo 'Hello'; sleep 100"
sudo ./engine start beta ./rootfs-beta "echo 'Hello'; sleep 100"

# Check status and logs
sudo ./engine ps
cat logs/alpha.log

# Stop containers and cleanup
sudo ./engine stop alpha
sudo ./engine stop beta
sudo rmmod monitor
```

---

## 3. Demo with Screenshots

Below is the visual evidence of the successful implementation of all project requirements:

1. **Multi-container supervision:** <img width="940" height="122" alt="image" src="https://github.com/user-attachments/assets/afb2e42b-3034-40b2-9f83-75246e096762" />
       <img width="940" height="91" alt="image" src="https://github.com/user-attachments/assets/01071fb2-2d7f-4c16-9515-c1e06b119815" />

 - *Shows the supervisor managing 'alpha' and 'beta' concurrently.*
2. **Metadata tracking:** <img width="913" height="234" alt="image" src="https://github.com/user-attachments/assets/2ac7dbc5-20d0-45fc-8308-b61d9413e185" />
 - *Output of `engine ps` displaying host PIDs and running states.*
3. **Bounded-buffer logging:** <img width="936" height="195" alt="image" src="https://github.com/user-attachments/assets/b5f3677a-8599-4129-9f25-d6b0e970dd76" />
 - *Shows log file contents successfully captured via the concurrent logging pipeline.*
4. **CLI and IPC:**  - *Demonstrates a CLI command being sent via UNIX sockets and the supervisor acknowledging it.*
5. **Soft-limit warning:**  - *`dmesg` output showing the kernel LKM triggering a SOFT LIMIT warning.*
6. **Hard-limit enforcement:** <img width="940" height="271" alt="image" src="https://github.com/user-attachments/assets/bbfc14c5-437e-4d2e-b5a9-184ad3184fd1" />
 - *`dmesg` output showing the kernel issuing a SIGKILL after the HARD LIMIT is breached.*
7. **Scheduling experiment:** <img width="940" height="481" alt="image" src="https://github.com/user-attachments/assets/b57d4538-473b-4428-abf5-4844516ae846" />
 - *Side-by-side terminal output showing the fast completion of the I/O task vs the CPU task.*
8. **Clean teardown:** <img width="940" height="271" alt="image" src="https://github.com/user-attachments/assets/7c3694af-d6f1-4aa7-9267-a4c58a7e54a8" />
 - *`ps aux | grep engine` output proving no zombie processes remain after shutdown.*

---

## 4. Engineering Analysis

### 4.1 Isolation Mechanisms
Our runtime achieves process isolation using the `clone()` system call with `CLONE_NEWPID`, `CLONE_NEWNS`, and `CLONE_NEWUTS` flags. Filesystem isolation is enforced via `chroot()`, effectively "jailing" the process within its specific `rootfs-*` directory. The host kernel still shares the underlying hardware drivers, network stack, and core kernel memory with all containers.

### 4.2 Supervisor and Process Lifecycle
A long-running parent supervisor is essential for tracking container metadata and preventing resource leaks. We manage the process lifecycle by maintaining a linked list of container records. A `SIGCHLD` signal handler was implemented to asynchronously reap exited child processes using `waitpid(..., WNOHANG)`, which updates metadata states safely without blocking the main event loop.

### 4.3 IPC, Threads, and Synchronization
We utilized two distinct IPC mechanisms: UNIX domain sockets (`/tmp/mini_runtime.sock`) for the CLI control plane, and anonymous pipes for container stdout/stderr logging. To prevent race conditions during logging, we implemented a **Bounded Buffer**. Without synchronization, concurrent writes from multiple pipes could cause data corruption or deadlocks. We chose a `pthread_mutex_t` combined with two condition variables (`not_empty` and `not_full`) to safely block and wake producer and consumer threads.

### 4.4 Memory Management and Enforcement
The Resident Set Size (RSS) measures the portion of memory held in physical RAM, ignoring swapped pages. We enforce these limits in kernel space via a Linux Kernel Module (LKM) rather than user space because user-space polling is not atomic; a rogue process could crash the system between user-space checks. The soft limit policy acts as an early warning system (logged to `dmesg`), while the hard limit policy guarantees system stability by dispatching an immediate, uncatchable `SIGKILL` from Ring 0.

### 4.5 Scheduling Behavior
We ran controlled experiments comparing a CPU-bound workload (`cpu_hog`) against an I/O-bound workload (`io_pulse`). By manipulating their `nice` values, we observed that the Linux Completely Fair Scheduler (CFS) prioritizes responsiveness. Even under load, the I/O-bound task completed its file-write bursts rapidly because the CFS rewards tasks that frequently yield their CPU timeslice, giving them a larger share of virtual runtime compared to the CPU-hogging process.

---

## 5. Design Decisions and Tradeoffs

* **Subsystem: Supervisor Architecture**
  * *Choice:* A single monolithic supervisor process handling both control IPC and logging.
  * *Tradeoff:* Simpler state management, but introduces a single point of failure (if the supervisor crashes, logging stops and containers become orphaned).
  * *Justification:* Ideal for a lightweight runtime where simplifying the metadata synchronization lock scope is prioritized over high availability.

* **Subsystem: Kernel List Synchronization**
  * *Choice:* Using a Spinlock (`spin_lock_irqsave`) to protect the kernel's process tracking list.
  * *Tradeoff:* Consumes CPU cycles by "spinning" while waiting for the lock, rather than putting the thread to sleep.
  * *Justification:* The list iteration occurs inside a kernel timer callback (an atomic interrupt context) where putting the kernel to sleep (e.g., using a Mutex) is strictly prohibited and would cause a kernel panic.

* **Subsystem: Logging IPC**
  * *Choice:* Bounded Buffer with fixed-size chunks.
  * *Tradeoff:* Can drop logs or block container execution if the disk write speed is drastically slower than the container output speed.
  * *Justification:* Provides necessary backpressure to protect the supervisor's heap memory from expanding infinitely during a logging flood.

---

## 6. Scheduler Experiment Results

**Workloads Executed Concurrently:**
1. `io_pulse` (High Priority: `nice -10`) - I/O Bound
2. `cpu_hog` (Low Priority: `nice 10`) - CPU Bound

**Observations:**
As seen in the log outputs, `io_pulse` completed its 10 iterations of writing to the filesystem while `cpu_hog` was still processing its early accumulators. This validates that the Linux CFS actively penalizes tasks that saturate the CPU and rewards tasks that sleep/wait for disk I/O, ensuring the overall system remains highly responsive to interactive events.
