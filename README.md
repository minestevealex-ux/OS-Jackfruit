Multi-Container Runtime (C + Linux Kernel Module)
Overview

This project implements a lightweight container runtime in C along with a kernel-space memory monitoring module. The system demonstrates core operating system concepts including process isolation, inter-process communication, scheduling, and kernel-level resource management.

The system consists of two major components:

A user-space runtime (engine.c) that manages containers
A kernel module (monitor.c) that enforces memory limits
System Architecture

The system follows a client–server model.

CLI (engine commands)
→ UNIX domain socket (/tmp/mini_runtime.sock)
→ Supervisor (long-running process)
→ Containers (isolated processes)
→ Kernel module (memory monitoring)

The supervisor acts as the central control unit, while containers are executed as isolated processes. The kernel module independently monitors memory usage.

Component 1: User-Space Runtime (engine.c)

Reference:

Responsibilities
Parses CLI commands
Starts and manages containers
Maintains container metadata
Handles logging
Communicates with kernel module using ioctl
Acts as a supervisor process
Supervisor Design

The supervisor is started using:

./engine supervisor <base-rootfs>

It performs the following:

Creates a UNIX domain socket at /tmp/mini_runtime.sock
Listens for client requests
Handles container lifecycle operations
Spawns a logging thread
Tracks container states
Supported Commands
start: launches a container in background
run: launches container similar to start
ps: lists running containers
logs: displays container logs
stop: terminates a container
Container Creation

Containers are created using the clone() system call:

clone(CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS)

This enables:

PID namespace: isolates process IDs
UTS namespace: separate hostname per container
Mount namespace: separate filesystem view
Container Setup

Inside the child process:

sethostname() sets container identity
chroot() isolates filesystem
chdir("/") sets working directory
/proc is mounted for process visibility
execv() runs the given command
Scheduling with Nice Values

Containers support CPU scheduling control via:

--nice N
Lower value (e.g., -10) gives higher priority
Higher value (e.g., 10) gives lower priority

This demonstrates Linux scheduler behavior.

Component 2: Logging System
Design

The logging system follows a producer-consumer model.

Producer: container output (stdout and stderr)
Consumer: logging thread
Implementation Details
Uses a bounded buffer
Synchronization via mutex and condition variables
Each container has a separate log file in the logs/ directory
Flow

Container output
→ pipe
→ log reader thread
→ bounded buffer
→ logging thread
→ file write

Advantages
Prevents blocking on I/O
Supports concurrent containers
Ensures ordered logging
Component 3: Kernel Module (monitor.c)

Reference:

Purpose

The kernel module monitors memory usage of containers and enforces limits.

It operates independently of user-space and uses kernel APIs.

Data Structures

Each container is represented by:

struct monitored_entry {
    pid_t pid;
    container_id;
    soft_limit_bytes;
    hard_limit_bytes;
}

All entries are stored in a linked list protected by a mutex.

Memory Monitoring

The module periodically checks memory usage using a timer.

Interval: 1 second
Function: get_mm_rss() retrieves RSS
Soft and Hard Limits

Two types of limits are enforced:

Soft limit:

Logs a warning when exceeded
Does not terminate the process

Hard limit:

Sends SIGKILL to terminate the process
Removes entry from monitoring list
Kernel Logging

Messages are printed using printk() and can be viewed using:

sudo dmesg | grep container_monitor
IOCTL Interface

The user-space runtime communicates with the kernel module using ioctl:

MONITOR_REGISTER: register container
MONITOR_UNREGISTER: remove container
Container Lifecycle
User issues command using CLI
Request sent to supervisor via socket
Supervisor creates container using clone
Container executes command inside isolated environment
Supervisor tracks container state
Kernel module monitors memory usage
Logs are collected asynchronously
Demonstration Steps
Start Supervisor
sudo ./engine supervisor ../rootfs-base
Start Container
sudo ./engine start alpha ../rootfs-alpha /cpu_hog
List Containers
sudo ./engine ps
View Logs
sudo ./engine logs alpha
Scheduling Demo
sudo ./engine start fast ../rootfs-alpha /cpu_hog --nice -10
sudo ./engine start slow ../rootfs-alpha /cpu_hog --nice 10
Memory Limit Demo
sudo ./engine start memtest ../rootfs-alpha /memory_hog --soft-mib 5 --hard-mib 50

Check kernel logs:

sudo dmesg | grep container_monitor
Cleanup
sudo pkill engine
ps aux | grep defunct
Key Concepts Demonstrated
Process isolation using namespaces
Inter-process communication using UNIX sockets
Producer-consumer synchronization
Kernel-user space interaction via ioctl
Memory monitoring using RSS
CPU scheduling using nice values
Conclusion

This project provides a simplified implementation of a container runtime, combining user-space control with kernel-level monitoring. It demonstrates how operating system concepts can be applied to build a system similar in principle to container technologies like Docker, but at a much lower level.
