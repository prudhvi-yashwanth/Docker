# VM vs Container

## 1. Deployment Evolution

### Traditional / Bare Metal / Legacy
- One physical server runs one main service, e.g. a database.
- No virtualization layer.
- Pros: full hardware access, no hypervisor overhead, strong physical isolation.
- Cons: poor hardware utilization, expensive, slow provisioning, hard to scale, dependency conflicts.

### Virtual Machine (VM)
- A hypervisor runs on bare metal or on a host OS.
- The hypervisor creates virtual hardware: vCPU, vRAM, virtual disk, virtual NIC.
- Each VM installs its own guest OS, kernel, libraries, and application.
- Resources are allocated as virtual hardware. They are usually reserved, though hypervisors can overcommit.
- Examples: VMware ESXi, KVM, Hyper-V, VirtualBox.
- Hypervisor types:
  - Type 1: bare-metal, e.g. ESXi, KVM, Hyper-V.
  - Type 2: hosted, e.g. VirtualBox, VMware Workstation.

**Pros:** strong isolation, can run different OS kernels, good for legacy apps, compliance, and untrusted workloads.  
**Cons:** heavy, full OS per VM, slower boot, more RAM/disk, slower scaling.

### Container
- Host OS + container engine + containers.
- Container engine examples: Docker, Podman, containerd, CRI-O.
- A container image packages the application code, runtime, libraries, and config.
- A container is a running instance of an image.
- Containers share the host OS kernel. They do **not** include a full OS kernel.
- Isolation is provided by Linux namespaces and cgroups:
  - **Namespaces** isolate what a process can see: PID, network, mount, user, etc.
  - **cgroups** limit and account resources: CPU, memory, I/O.
- Resources are shared but can be limited per container.

**Pros:** lightweight, portable, fast start, high density, CI/CD friendly, good for microservices.  
**Cons:** weaker isolation than VMs, must match host kernel/architecture, storage and networking can be more complex.

## 2. VM vs Container Comparison

| Feature | VM | Container |
|---|---|---|
| Virtualizes | Hardware | OS |
| Guest OS | Required per VM | Not required |
| Kernel | Own kernel per VM | Shares host kernel |
| Startup | Seconds to minutes | Milliseconds to seconds |
| Size | GBs | MBs to few GBs |
| Resource usage | High | Low |
| Density | Few per host | Many per host |
| Isolation | Stronger by default | Weaker by default |
| Portability | Less portable | Highly portable as image |
| Best for | Legacy, different OS, strong isolation, compliance | Microservices, CI/CD, cloud-native, rapid scaling |

**Simple analogy:**
- Bare metal = own house.
- VM = apartment with its own kitchen and bathroom.
- Container = room in a shared apartment with a shared kitchen.

## 3. Key Concepts to Remember
- VM = virtualized hardware. Container = virtualized OS.
- Containers share the host kernel, so a Linux container needs a Linux kernel. On Windows/macOS, Docker usually runs a Linux VM in the background.
- Containers are not always more secure. Harden them: non-root user, minimal images, seccomp/AppArmor, image scanning, network policies.
- Docker uses a daemon and runs as root by default. Podman is daemonless and rootless-friendly. Both are OCI-compliant.
- VMs and containers are often used together. In cloud, containers usually run inside VMs.

## 4. Practical Commands
```bash
# Check host kernel
uname -r

# Run an Nginx container
docker run -d --name web -p 8080:80 nginx

# Limit container resources
docker run --memory=512m --cpus=1 nginx

# View live container resource usage
docker stats

# Run a rootless container with Podman
podman run --rm -it alpine sh
```

## 5. Interview Points

### Common Questions & Answers
**Q: What is the main difference between a VM and a container?**  
A: A VM virtualizes hardware and has its own guest OS kernel. A container virtualizes the OS, shares the host kernel, and packages only the app and dependencies.

**Q: Why do containers start faster than VMs?**  
A: Containers do not boot a full OS. They start a process in isolated namespaces.

**Q: Are containers secure?**  
A: Less isolated than VMs by default. Security depends on hardening: rootless mode, non-root users, minimal images, seccomp, AppArmor, scanning, and network policies.

**Q: Can containers replace VMs?**  
A: Not always. VMs are better for different OS kernels, legacy apps, strong isolation, and compliance. Containers are better for microservices, CI/CD, and cloud-native scaling. Many systems use both.

**Q: What is a hypervisor?**  
A: Software that creates and runs VMs. Type 1 runs on bare metal. Type 2 runs on a host OS.

**Q: What are namespaces and cgroups?**  
A: Namespaces isolate what a process can see. cgroups limit and measure CPU, memory, and I/O.

**Q: Docker vs Podman?**  
A: Docker uses a daemon and root by default. Podman is daemonless and rootless-friendly. Both run OCI containers.

### Product-Based Company Scenarios
**Scenario: You need to run 50 microservices on one 16GB/4vCPU server. VMs or containers?**  
Use containers with CPU/memory limits. They give higher density and faster scaling. Use orchestration later when scale grows.

**Scenario: A container is consuming too much memory and affecting others. What do you do?**  
Set memory limits with cgroups: `docker run --memory=512m --memory-swap=512m`. Monitor with `docker stats`. The kernel may OOM-kill the container if it exceeds limits.

**Scenario: You need to run untrusted code from users. Containers or VMs?**  
Prefer VMs or sandboxed containers like gVisor/Kata for stronger isolation. Plain containers share the kernel, so a kernel exploit can affect the host.

### Service-Based Company Scenarios
**Scenario: Client has a legacy Windows app and a Linux app. How do you deploy?**  
Use VMs because each needs a different OS kernel. Containers cannot share different kernels on the same host.

**Scenario: Client asks why not use only VMs?**  
Explain cost, density, startup speed, and portability benefits of containers. Use VMs where strong isolation, different OS, or legacy support is required.

**Scenario: CI/CD pipeline needs fast, consistent build and test environments. What do you use?**  
Use containers. They are disposable, reproducible, and start quickly.

**Scenario: Stateful database deployment. VM or container?**  
VMs are often preferred for predictable I/O and strong isolation. Containers are possible with persistent volumes and careful resource limits, but need more operational care.

## 6. Quick Takeaway
- Bare metal: one service per server, poor utilization.
- VM: virtualized hardware, own OS, strong isolation, heavy.
- Container: virtualized OS, shared kernel, lightweight, fast, portable, weaker isolation.
- Choose based on isolation, OS needs, density, speed, and operational cost.
