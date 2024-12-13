# OpenMPI Cluster Setup Documentation
**Version 1.0**
**Date: December 12, 2024**

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Hardware Architecture](#2-hardware-architecture)
3. [Operating System Setup](#3-operating-system-setup)
4. [Network Configuration](#4-network-configuration)
5. [SSH Configuration](#5-ssh-configuration)
6. [NFS Setup](#6-nfs-setup)
7. [OpenMPI Installation](#7-openmpi-installation)
8. [Cluster Configuration](#8-cluster-configuration)
9. [Testing and Validation](#9-testing-and-validation)
10. [Troubleshooting](#10-troubleshooting)
11. [Appendix](#11-appendix)

## 1. System Overview

### 1.1 Purpose
This document provides detailed instructions for setting up an OpenMPI cluster across 5 compute nodes for high-performance computing applications.

### 1.2 Scope
- Initial OS installation
- Network configuration
- Security setup
- Parallel computing environment configuration
- Testing and validation procedures

### 1.3 System Architecture Overview
- 1 Master Node (PDC-Node-1)
- 4 Compute Nodes (PDC-Node-2 to PDC-Node-5)
- High-speed network interconnect
- Shared storage system

## 2. Hardware Architecture

### 2.1 Node Specifications
```plaintext
Master Node (PDC-Node-1):
- CPU: Intel Xeon E5-2680 v4
- RAM: 128GB DDR4
- Storage: 2TB NVMe SSD
- Network: 10GbE

Compute Nodes (PDC-Node-2 to PDC-Node-5):
- CPU: Intel Xeon E5-2680 v4
- RAM: 64GB DDR4
- Storage: 1TB SSD
- Network: 10GbE
```

### 2.2 Network Topology
```plaintext
                    [10GbE Switch]
                          |
     +----------+----------+----------+----------+
     |          |          |          |          |
[PDC-Node-1][PDC-Node-2][PDC-Node-3][PDC-Node-4][PDC-Node-5]
  (Master)    (Compute)   (Compute)   (Compute)   (Compute)
```

## 3. Operating System Setup

### 3.1 Base Installation
```bash
# Install Ubuntu Server 24.04 LTS on all nodes
# Minimal installation with SSH server
```

### 3.2 System Updates
```bash
# On all nodes
sudo apt update
sudo apt upgrade -y
sudo apt install -y build-essential
```

## 4. Network Configuration

### 4.1 Static IP Configuration on all nodes 
Edit `/etc/netplan/00-installer-config.yaml`:
```yaml
network:
  ethernets:
    eno1:
      addresses:
        - <node_ip>/24 
      gateway4: <gatway_ip>
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
```

### 4.2 Hosts Configuration
Edit `/etc/hosts` on all nodes:
```plaintext
127.0.0.1       localhost
192.168.1.101   PDC-Node-1
192.168.1.102   PDC-Node-2
192.168.1.103   PDC-Node-3
192.168.1.104   PDC-Node-4
192.168.1.105   PDC-Node-5
```

## 5. SSH Configuration

### 5.1 Generate SSH Keys
```bash
# On master node (PDC-Node-1)
ssh-keygen -t ed25519 -f ~/.ssh/cluster-key
```

### 5.2 Deploy Keys
```bash
# On master node
for i in {2..5}; do
    ssh-copy-id -i ~/.ssh/cluster-key.pub username@PDC-Node-$i
done
```

### 5.3 SSH Config
Create `~/.ssh/config` on master node:
```plaintext
Host PDC-Node-*
    User username
    IdentityFile ~/.ssh/cluster-key
    StrictHostKeyChecking no
```

### 5.4 Disable firewall or allow openmpi ports on system firewall
```bash
# To disable firewall
sudo ufw disable
```

## 6. NFS Setup

### 6.1 Server Configuration (PDC-Node-1)
```bash
# Install NFS server
sudo apt install -y nfs-kernel-server

# Create shared directory
sudo mkdir -p /shared
sudo chown -R username:username /shared
sudo chmod 755 /shared

# Configure exports
echo "/shared *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

### 6.2 Client Configuration (Compute Nodes)
```bash
# Install NFS client
sudo apt install -y nfs-common

# Create mount point
sudo mkdir -p /shared

# Add to /etc/fstab
echo "PDC-Node-1:/shared /shared nfs defaults 0 0" | sudo tee -a /etc/fstab
sudo mount -a
```

## 7. OpenMPI Installation

### 7.1 Dependencies
```bash
# On all nodes
sudo apt install -y \
    build-essential \
    gfortran \
    openmpi-bin \
    openmpi-common \
    libopenmpi-dev \
    libopenblas-dev
```

## 8. Cluster Configuration

### 8.1 Create Machinefile
Create `/shared/machinefile`:
```plaintext
PDC-Node-1 slots=32
PDC-Node-2 slots=32
PDC-Node-3 slots=32
PDC-Node-4 slots=32
PDC-Node-5 slots=32
```

### 8.2 Environment Setup
Add to `~/.bashrc` on all nodes:
```bash
export PATH="/shared:$PATH"
export LD_LIBRARY_PATH="/shared/lib:$LD_LIBRARY_PATH"
```

## 9. Testing and Validation

### 9.1 Basic Connectivity Test
```bash
# Test all nodes
for i in {1..5}; do
    ssh PDC-Node-$i hostname
done
```

### 9.2 MPI Test Program
Create `/shared/mpi_test.c`:
```c
#include <mpi.h>
#include <stdio.h>
#include <unistd.h>

int main(int argc, char** argv) {
    int world_size, world_rank;
    char processor_name[MPI_MAX_PROCESSOR_NAME];
    int name_len;

    MPI_Init(&argc, &argv);
    MPI_Comm_size(MPI_COMM_WORLD, &world_size);
    MPI_Comm_rank(MPI_COMM_WORLD, &world_rank);
    MPI_Get_processor_name(processor_name, &name_len);

    printf("Process %d of %d on %s\n", 
           world_rank, world_size, processor_name);

    MPI_Finalize();
    return 0;
}
```

### 9.3 Compile and Run Test
```bash
# Compile
mpicc -o /shared/mpi_test /shared/mpi_test.c

# Run on all nodes
mpirun -np 80 --hostfile /shared/machinefile /shared/mpi_test
```

## 10. Troubleshooting

### 10.1 Common Issues
1. SSH Connection Issues
```bash
# Check SSH service
sudo systemctl status ssh

# Test SSH connection
ssh -vv username@PDC-Node-2
```

2. NFS Issues
```bash
# Check NFS mounts
showmount -e PDC-Node-1

# Check NFS service
sudo systemctl status nfs-kernel-server
```

3. MPI Issues
```bash
# Check OpenMPI version
mpirun --version

# Check process limits
ulimit -a
```

## 11. Appendix

### 11.1 Performance Testing Code
```c
#include <stdio.h>
#include <stdlib.h>
#include <mpi.h>
#include <math.h>

#define NUM_POINTS 1000000

int main(int argc, char** argv) {
    int rank, size;
    double pi_local, pi_global;
    double start_time, end_time;

    MPI_Init(&argc, &argv);
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);

    start_time = MPI_Wtime();

    unsigned int seed = rank;
    int count = 0;
    for(int i = 0; i < NUM_POINTS; i++) {
        double x = (double)rand_r(&seed)/RAND_MAX;
        double y = (double)rand_r(&seed)/RAND_MAX;
        if(x*x + y*y <= 1.0) count++;
    }

    pi_local = 4.0 * count / NUM_POINTS;
    MPI_Reduce(&pi_local, &pi_global, 1, MPI_DOUBLE, 
               MPI_SUM, 0, MPI_COMM_WORLD);

    end_time = MPI_Wtime();

    if(rank == 0) {
        pi_global = pi_global/size;
        printf("Pi: %.10f\n", pi_global);
        printf("Time: %f seconds\n", end_time - start_time);
    }

    MPI_Finalize();
    return 0;
}
```

### 11.2 Version Information
- Operating System: Ubuntu 24.04 LTS
- OpenMPI Version: 4.1.6
- GCC Version: 13.2.0

### 11.3 Security Recommendations
1. Configure UFW firewall on all nodes
2. Implement regular security updates
3. Monitor system logs
4. Use strong SSH key encryption
5. Regularly audit user access

### 11.4 Performance Optimization
1. Enable processor-specific optimizations
2. Tune network parameters
3. Optimize MPI parameters
4. Monitor system performance
5. Regular maintenance schedule

This documentation follows the IEEE documentation standards and includes all necessary information for setting up and maintaining an OpenMPI cluster. Regular updates and maintenance procedures should be followed to ensure optimal performance and security.
