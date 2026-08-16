**Docker works by using containers to run applications in isolated, lightweight environments that share the host operating system’s kernel.** Instead of creating a full virtual machine with its own OS, Docker packages the app and its dependencies into a container that can run anywhere.

## 🧩 OS Kernel Sharing in Docker

- **Shared kernel**: Unlike virtual machines, Docker containers don’t need their own operating system. They all share the **host OS kernel**.
    
- **Process isolation**: Each container runs as an isolated process, with its own filesystem, libraries, and environment variables, but still relies on the host kernel for system calls.
    
- **Lightweight design**: Because containers don’t carry a full OS, they start up in seconds and use fewer resources.
    
- **Namespaces**: The kernel uses namespaces to give each container its own view of the system (like separate process IDs, network interfaces, and mounts).
    
- **Control groups (cgroups)**: The kernel enforces resource limits (CPU, memory, I/O) so containers don’t interfere with each other.