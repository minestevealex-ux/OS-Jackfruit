# Multi-Container Runtime (C + Linux Kernel Module)

## Overview

This project implements a lightweight container runtime in C along with a kernel-space memory monitoring module. The system demonstrates core operating system concepts such as process isolation, inter-process communication, scheduling, and kernel-level resource management.

The system consists of two major components:

- A user-space runtime (`engine.c`) that manages containers
- A kernel module (`monitor.c`) that enforces memory limits

---

## System Architecture

The system follows a client-server model.

CLI (engine commands)  
→ UNIX domain socket (/tmp/mini_runtime.sock)  
→ Supervisor (long-running process)  
→ Containers (isolated processes)  
→ Kernel module (memory monitoring)  

The supervisor acts as the central control unit, while containers are executed as isolated processes. The kernel module independently monitors memory usage.

---

## Component 1: User-Space Runtime (`engine.c`)

### Responsibilities

- Parses CLI commands  
- Starts and manages containers  
- Maintains container metadata  
- Handles logging  
- Communicates with kernel module using ioctl  
- Acts as a supervisor process  

---

### Supervisor

Start the supervisor using:
sudo ./engine supervisor ../rootfs-base

The supervisor:

- Creates a UNIX domain socket at `/tmp/mini_runtime.sock`
- Listens for client requests
- Handles container lifecycle operations
- Tracks container state

---

### Supported Commands

- `start` — launch container in background  
- `run` — run container  
- `ps` — list containers  
- `logs` — view logs  
- `stop` — stop container  

---

### Container Creation

Containers are created using:
clone(CLONE_NEWPID | CLONE_NEWUTS | CLONE_NEWNS)


This provides:

- PID namespace for process isolation  
- UTS namespace for hostname isolation  
- Mount namespace for filesystem isolation  

---

### Container Setup

Inside the container:

- `sethostname()` sets container identity  
- `chroot()` isolates filesystem  
- `chdir("/")` sets working directory  
- `/proc` is mounted  
- `execv()` runs the command  

---

### Scheduling with Nice Values

sudo ./engine start fast ../rootfs-alpha /cpu_hog --nice -10
sudo ./engine start slow ../rootfs-alpha /cpu_hog --nice 10


- Lower nice value → higher priority  
- Higher nice value → lower priority  

---

## Component 2: Logging System

### Design

- Producer: container output  
- Consumer: logging thread  

---

### Flow

Container → pipe → log reader → buffer → logging thread → file

---

### Features

- Uses a bounded buffer  
- Uses mutex and condition variables  
- Each container has a separate log file  

---

## Component 3: Kernel Module (`monitor.c`)

### Purpose

The kernel module monitors memory usage of containers and enforces limits.

---

### Memory Monitoring

- Uses RSS (Resident Set Size)  
- Checks memory periodically using a timer  

---

### Soft and Hard Limits

Soft limit:
- Logs a warning when exceeded  
- Does not kill the process  

Hard limit:
- Terminates the process using `SIGKILL`  

---

### Viewing Kernel Logs

sudo dmesg | grep container_monitor

---

## Demonstration

### Start Supervisor

sudo ./engine supervisor ../rootfs-base

---

### Start Container

sudo ./engine start alpha ../rootfs-alpha /cpu_hog

---

### List Containers

sudo ./engine ps

---

### View Logs

sudo ./engine logs alpha

---

### Scheduling Demo

sudo ./engine start fast ../rootfs-alpha /cpu_hog --nice -10
sudo ./engine start slow ../rootfs-alpha /cpu_hog --nice 10

---

### Memory Limit Demo

sudo ./engine start memtest ../rootfs-alpha /memory_hog --soft-mib 5 --hard-mib 50

Check kernel output:
sudo dmesg | grep container_monitor

---

### Cleanup

sudo pkill engine
ps aux | grep defunct

---

## Key Concepts Demonstrated

- Process isolation using namespaces  
- Inter-process communication using UNIX sockets  
- Producer-consumer synchronization  
- Kernel-user communication using ioctl  
- Memory monitoring using RSS  
- CPU scheduling using nice values  

---

## Comparison with Docker

This project is a simplified version of Docker.

- Uses namespaces for isolation (same as Docker)  
- Uses a custom kernel module for memory control  
- Docker uses cgroups instead  

---

## Conclusion

This project demonstrates how container runtimes work internally by combining user-space process management with kernel-level monitoring. It provides a low-level understanding of how systems like Docker operate.
