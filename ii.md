# Part 1: OS Architecture & System Calls

## 1. Kernel Architecture Investigation

### Identify Your OS Architecture

**OS:** Debian (running in UTM VM)  
**Kernel Version:**
```sh
uname -r
```
_Output:_
```
6.x.x-arch-debian
```
Debian uses the **Linux kernel**, which follows a **monolithic modular** architecture. While the Linux kernel is fundamentally monolithic, it supports dynamically loadable modules, making it **modular** in nature. This allows features to be added or removed at runtime without modifying the core kernel.

### Inspect Loaded Kernel Modules

List currently loaded modules:
```sh
lsmod | head -10
```
Pick a module and inspect:
```sh
modinfo ext4
```
#### Findings:
- `ext4` is the module handling the Ext4 filesystem.
- It has dependencies on `mbcache` and `jbd2`, which help with metadata caching and journaling, respectively.

## 2. System Call Exploration

### Trace a Simple System Call

simple C program:
```c
#include <stdio.h>
#include <unistd.h>
int main() {
    printf("PID: %d\n", getpid());
    return 0;
}
```
Compile and trace:
```sh
gcc getpid.c -o getpid
strace ./getpid
```
#### Observed System Calls:
- `execve()` - Loads the program into memory.
- `brk()` - Adjusts memory allocation.
- `getpid()` - Retrieves process ID.
- `write()` - Outputs text to stdout.

### Directly Invoke a System Call

```c
#include <unistd.h>
#include <sys/syscall.h>
#include <stdio.h>
int main() {
    syscall(SYS_write, STDOUT_FILENO, "Hello\n", 6);
    return 0;
}
```
#### Explanation:
- `syscall()` directly invokes the **write** system call, bypassing the C standard library wrapper.

## 3. Comparative Analysis

3.1 Kernel Architecture Comparison
| Feature | Monolithic | Microkernel | Modular | 
|-------------|------------|-------------|---------|
| Performance | High | Lower | High |
| Security | Low | High | Medium |
| Development | Complex | Easier | Easier |
### Reflection:
- **Embedded Systems:** Microkernel is often preferred for reliability and security.
- **Large-Scale Servers:** Modular monolithic (like Linux) provides performance and extensibility. Microkernel is ideal for embedded due to modularity. Monolithic is better for high-performance server systems. 

## 4. Advanced Activities

### Module Loading and Unloading

```sh
sudo modprobe dummy
lsmod | grep dummy
sudo rmmod dummy
dmesg | tail -5
```
#### Notes:
- `dummy` is a simple module for testing network interfaces.
- `dmesg` logs kernel messages about module insertion/removal.

### Assembly-Level Experiment

```assembly
section .text
global _start
_start:
    mov eax, 4       ; SYS_write
    mov ebx, 1       ; STDOUT
    mov ecx, msg
    mov edx, len
    int 0x80         ; Call kernel
    mov eax, 1       ; SYS_exit
    xor ebx, ebx
    int 0x80         ; Exit
section .data
msg db "Hello, world!", 10
len equ $ - msg
```
Compile and run:
```sh
nasm -f elf32 hello.asm -o hello.o
gcc -m32 hello.o -o hello
d ./hello
```
#### Explanation:
- Uses **int 0x80** for system calls in 32-bit mode.
- Calls `SYS_write` to print text, then `SYS_exit` to terminate.

# Part 2: Process Concepts & Basic Tools

## 1. Process Investigation

### Observing Active Processes
```sh
ps -ef | head -10
```
Key columns:
- `PID` – Process ID
- `PPID` – Parent Process ID
- `CMD` – Command used to start process
- `TIME` – CPU time consumed

### Investigating Process States
Start a process:
```sh
sleep 100 &
ps -C sleep -o pid,stat,cmd
```
Pause & resume:
```sh
kill -SIGSTOP <PID>
kill -SIGCONT <PID>
```
#### Findings:
- `R` (Running)
- `S` (Sleeping)
- `T` (Stopped)

## 2. Threads vs. Processes

### Creating Processes
```c
#include <stdio.h>
#include <unistd.h>
int main() {
    for (int i = 0; i < 3; i++) {
        if (fork() == 0) {
            printf("Child PID: %d\n", getpid());
            return 0;
        }
    }
    return 0;
}
```

### Creating Threads
```c

#include <stdio.h>
#include <pthread.h>

void* print_message(void* arg) {
    printf("Thread ID: %ld is running\n", pthread_self());
    return NULL;
}

int main() {
    pthread_t threads[4];
    for (int i = 0; i < 4; i++) {
        pthread_create(&threads[i], NULL, print_message, NULL);
    }
    for (int i = 0; i < 4; i++) {
        pthread_join(threads[i], NULL);
    }
    return 0;
}
```
## 3. Using OS Commands

### Monitoring Tools
```sh
top
htop
```

### Process Tree
```sh
pstree -p | less
```

### Adjusting Priorities
```sh
renice -n 10 -p <PID>
```

# Part 3: Scheduling

This part aligns with the topics covered in the previous slides of the section, emphasizing practical application of scheduling algorithms and hands-on kernel exploration. Use this assignment to solidify your understanding of how scheduling works in theory and how it’s actually implemented in the Linux kernel source code.

## 1. Scheduling Simulation

### 1.1 Set Up a Simple Scheduling Simulator

We implemented a simple scheduling simulator in C. The process set consists of five processes, each with arrival time, burst time, and priority.

Example Process Set:

| Process | Arrival Time | Burst Time | Priority |
|---------|--------------|------------|----------|
| P1      | 0            | 5          | 2        |
| P2      | 1            | 3          | 1        |
| P3      | 2            | 8          | 3        |
| P4      | 3            | 6          | 2        |
| P5      | 4            | 2          | 1        |

### 1.2 Input Scheduling Algorithms

We simulated the following algorithms:
- FCFS (First-Come-First-Serve)
- SJF (Shortest Job First)
- Round Robin (Time Quantum = 2)
- Priority Scheduling

For each, we recorded:
- Waiting Time
- Turnaround Time
### 1.3 Compare Outcomes

| Algorithm | Avg Waiting Time | Avg Turnaround Time |
|-----------|------------------|---------------------|
| FCFS      | X                | Y                   |
| SJF       | X                | Y                   |
| RR        | X                | Y                   |
| Priority  | X                | Y                   |

Interpretation:
- SJF gives the lowest average waiting time in batch processing.
- Round Robin is best suited for time-sharing and fairness in multi-user systems.

## 2. Kernel Tour

### 2.1 Navigate the Linux GitHub Repository

We explored the Linux kernel repository on GitHub, focusing on scheduling-related code.

Key Directories:
- `kernel/` — System calls and core logic.
- `fs/` — File systems like ext4.
- `drivers/` — Device drivers.
- `kernel/sched/` — CPU Scheduling source code.
### 2.2 Identify Scheduling Files

In `kernel/sched/` we found important files:
- `sched_fair.c` — Implements the Completely Fair Scheduler (CFS).
- `sched_rt.c` — Real-Time scheduling policies.
- `sched_core.c` — Core scheduling logic.

We identified functions like `schedule()` and data structures such as `task_struct` and `rq` (run queues) responsible for managing processes.


### 2.3 Document Your Findings

| Directory     | Purpose                              | Notable Files/Functions       |
|---------------|-------------------------------------|--------------------------------|
| kernel/sched/ | CPU scheduling implementation      | `sched_fair.c`, `sched.c`     |
| fs/           | Filesystem handling                | ext4, btrfs code              |
| drivers/      | Hardware interfacing               | Driver implementations        |

## 3. Scenario Discussion

### Round Robin Advantage:

A large multi-user system (e.g., university lab servers) benefits from Round Robin because it ensures fair CPU time-sharing among all users, preventing starvation.

### SJF Advantage:

In a batch processing system where short tasks dominate (e.g., automated testing jobs), SJF minimizes average waiting time and turnaround time.

## 4. Short Scheduling Problems

Designed Process Set:

| Process | Arrival Time | Burst Time | Priority |
|---------|--------------|------------|----------|
| P1      | 0            | 7          | 3        |
| P2      | 1            | 5          | 2        |
| P3      | 2            | 1          | 1        |
| P4      | 3            | 2          | 4        |
| P5      | 4            | 3          | 1        |

Manually Calculated Waiting and Turnaround Times for FCFS, SJF, RR.

Deliverable: Attached handwritten calculations showing detailed steps.
## 6. Reflection & Group Deliverables 

### Group Summary
Working together on this assignment strengthened our technical understanding and collaboration skills. We combined our findings from the scheduling simulation, Linux kernel exploration, and system call tracing. Linux's CFS and RT scheduling policies relate closely to theoretical algorithms like SJF and Round Robin. Exploring the open-source kernel bridged the gap between textbook scheduling algorithms and their real-world implementations. Open-source exploration is valuable because it allows us to see how complex scheduling policies are practically implemented and maintained over time. 

### Group Members' Reflections

- Sammin Ahamed: Learned how kernel architecture impacts performance and why open-source exploration is crucial. Found the exploration of Linux's `sched_fair.c` particularly interesting, realizing how balancing fairness and efficiency is critical in scheduling.

- Chenjing Zhuang: Found hands-on scheduling simulations very useful for understanding how Linux manages processes in real time. The comparison between different scheduling algorithms clarified the practical trade-offs between throughput, fairness, and response time. Tracing system calls deepened my appreciation of how user programs interact with the kernel.


