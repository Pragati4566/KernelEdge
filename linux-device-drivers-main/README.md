# Linux Device Driver Lab

[![Language](https://img.shields.io/badge/language-C-00599C?logo=c&logoColor=white)](.)
[![Platform](https://img.shields.io/badge/platform-Linux%20Kernel-FCC624?logo=linux&logoColor=black)](.)
[![Containers](https://img.shields.io/badge/environment-Docker-2496ED?logo=docker&logoColor=white)](.)
[![Virtualization](https://img.shields.io/badge/testing-QEMU-FF6600?logo=qemu&logoColor=white)](.)

A hands-on collection of Linux kernel modules for exploring four important driver areas: character devices, block devices, network interfaces, and PCIe devices. The projects are designed to demonstrate how kernel components communicate with user space and interact with the Linux device, storage, and networking subsystems.

The test workflows use Docker and QEMU so the drivers can be built and exercised in a virtual environment without requiring dedicated physical hardware. Check each module's own documentation for its exact prerequisites and test procedure.

## Driver Overview

| Project | Subsystem | Main concepts |
|---|---|---|
| [`linux-character-device-driver`](linux-character-device-driver/) | Character devices | `file_operations`, mutexes, wait queues, blocking reads and writes, `ioctl` |
| [`linux-block-device-driver`](linux-block-device-driver/) | Block I/O | `gendisk`, blk-mq, bio/request processing, spinlocks, page cache and `O_DIRECT` |
| [`linux-network-device-driver`](linux-network-device-driver/) | Networking | `net_device`, `sk_buff`, descriptor rings, NAPI, transmit-queue backpressure |
| [`linux-pcie-device-driver`](linux-pcie-device-driver/) | PCI / PCIe | BAR-mapped MMIO, MSI interrupts, DMA buffer management |

## Projects at a Glance

### 1. Character Device — `circbuf`

This module exposes a circular buffer through `/dev/circbuf`. Writers add bytes to a fixed-capacity buffer and readers consume them in FIFO order. When the buffer is full, writers can wait for space; when it is empty, readers can wait for data. Synchronization primitives coordinate access by multiple processes.

Concepts covered include:

- Registering device operations through `struct file_operations`.
- Transferring data safely between kernel space and user space with `copy_to_user` and `copy_from_user`.
- Choosing between mutexes and spinlocks according to the execution context.
- Using `wait_queue_head_t` to support blocking I/O.
- Defining `ioctl` commands for device-specific operations.

**Core interface:** `struct file_operations`, which connects operations such as `open`, `read`, `write`, `ioctl`, and `release` to the driver.

### 2. Block Device — `vblk`

The `vblk` module presents a RAM-backed block device at `/dev/vblk0`. Its storage is held in kernel memory, while its interfaces allow user-space tools to treat it like a disk. Operations such as `dd`, `mkfs`, `mount`, `pread`, and `pwrite` can be used to exercise the block layer.

Concepts covered include:

- Registering a disk and its request queue with `gendisk` and blk-mq.
- Handling I/O through `bio` and request segments.
- Protecting shared data in driver execution paths.
- Understanding how buffered I/O uses the page cache and how `O_DIRECT` changes that path.
- Exercising a filesystem workflow, including formatting, mounting, file operations, and unmounting.

**Core interfaces:** `struct gendisk` and `struct blk_mq_tag_set`.

### 3. Network Device — `netdrv`

This module creates a software Ethernet pair named `vnet0` and `vnet1`. Traffic sent through one interface can be received by the other, providing a way to explore networking-driver mechanics without a physical network card.

Concepts covered include:

- Registering a network interface through `struct net_device` and `struct net_device_ops`.
- Managing packets with `struct sk_buff`.
- Using transmit and receive descriptor rings.
- Processing received packets with NAPI polling.
- Applying transmit-queue backpressure with `netif_stop_queue` and `netif_wake_queue`.
- Testing interfaces in isolated network namespaces.

The included end-to-end workflow can use `ping` to verify connectivity between the virtual interfaces.

**Core interfaces:** `struct net_device` and `struct napi_struct`.

### 4. PCIe Device — `pcie-edu-driver`

This module targets QEMU's educational PCI device named `edu`. The emulated device is enumerated on the PCI bus and provides memory-mapped registers, interrupt support, and a DMA engine. This creates a virtual environment for exploring the lifecycle of a PCI driver.

Concepts covered include:

- Enabling and registering a PCI device.
- Mapping a device's BAR region for memory-mapped I/O.
- Reading and writing registers with `ioread32` and `iowrite32`.
- Registering MSI interrupt vectors.
- Allocating coherent DMA buffers and testing data transfers.
- Cleaning up resources when the device is removed or the module is unloaded.
- Exposing user-facing operations through `/dev/pcie_edu` and `ioctl` commands.

**Core interface:** `struct pci_dev`, together with mapped registers, an interrupt vector, and DMA buffers.

## How the Projects Fit Together

The character-device example introduces the kernel/user-space boundary and synchronization. The block-device example applies related ideas to queued storage requests and disk semantics. The network driver explores packet handling and NAPI, while the PCIe example adds bus enumeration, memory-mapped registers, interrupts, and DMA.

Together, the modules provide practical examples of how different Linux subsystems define their own interfaces, execution contexts, and resource-management requirements.

## Repository Layout

```text
linux-device-drivers-main/
├── linux-character-device-driver/   # Circular-buffer character device
├── linux-block-device-driver/       # RAM-backed block device
├── linux-network-device-driver/     # Virtual Ethernet pair
└── linux-pcie-device-driver/        # QEMU educational PCIe device
```

Each driver directory contains its own implementation and build/test files. Refer to its local README for subsystem-specific details and any driver-specific requirements.

## Getting Started

### Prerequisites

| Tool | Requirement | Purpose |
|---|---|---|
| Docker | Version 20.10 or later is recommended | Builds and runs the test environment in a container |
| QEMU | Used by the Docker-based test setup | Boots a virtual machine for end-to-end tests |
| `make` and GCC | Recent versions for native builds | Builds kernel modules outside the container workflow |
| Linux kernel headers | Must match the running kernel for native builds | Provides the kernel interfaces needed for compilation |

The container-based route is intended to reduce host setup. Review the driver-specific documentation before running a workflow, especially if your environment uses a different architecture or kernel version.

### Clone the Repository

```bash
git clone https://github.com/Pragati4566/KernelEdge.git
cd KernelEdge/linux-device-drivers-main
```

### Run the Docker-Based Tests

For the character, block, and PCIe drivers, the general command pattern is:

```bash
cd <driver-directory>
./docker/run.sh test
```

For the network driver, use its Dockerfile:

```bash
cd linux-network-device-driver
docker build -f docker/Dockerfile -t netdrv-e2e .
docker run --rm netdrv-e2e
```

Run these commands from the appropriate driver directory. Exact scripts and available options may differ between modules.

### What the Test Harness Does

Where implemented by the individual driver, the container-based harness can:

1. Build the kernel module against the configured kernel headers.
2. Start a Linux kernel in QEMU with a minimal userspace environment.
3. Load the module, execute driver tests, and unload it.
4. Exercise relevant paths such as device I/O, stress scenarios, filesystem operations, or `ioctl` calls.
5. Check kernel logs for serious errors and return a failure status when a test fails.

Consult the relevant driver's README and scripts to confirm which of these checks are available for that module.

### Native Build Workflow

For a native Linux environment with matching kernel headers, the workflow generally looks like this:

```bash
make                         # Build the module
sudo insmod <module-name>.ko # Load the module
dmesg | tail -20             # Inspect recent kernel messages
sudo ./<test-binary>          # Run the module's test program
sudo rmmod <module-name>     # Unload the module
```

Replace the placeholder names with those used by the selected driver. The PCIe example also requires an appropriately configured QEMU session with the educational `edu` device enabled.

## Test Coverage

The project includes driver-specific checks for loading and unloading modules, basic device operations, and selected edge cases. Depending on the module, tests may cover concurrent circular-buffer access, direct and buffered block I/O, filesystem operations, virtual-network connectivity, interrupt handling, or DMA transfers.

Use the test scripts and latest run output in your checkout as the source of truth for test status. Do not treat a successful build alone as proof that a driver works correctly, and rerun the relevant tests after making changes.

## Troubleshooting

- **Module build fails:** confirm that the kernel headers and compiler configuration match the target kernel.
- **`insmod` fails:** inspect `dmesg` for registration errors, missing symbols, or resource conflicts.
- **A device node is missing:** check whether module initialization succeeded and whether the driver registers the expected device interface.
- **QEMU or Docker tests fail:** check Docker availability, image builds, architecture compatibility, and the driver's documented launch command.
- **Network tests fail:** verify that both virtual interfaces are registered and that the test uses the expected network namespace configuration.

Kernel modules run with high privileges and can affect system stability. Prefer an isolated VM or the documented container/QEMU environment when experimenting.

## Further Reading

- *Linux Device Drivers, 3rd Edition* — Jonathan Corbet, Alessandro Rubini, and Greg Kroah-Hartman.
- *Linux Kernel Development, 3rd Edition* — Robert Love.
- Linux kernel documentation: `Documentation/driver-api/`, `Documentation/block/`, `Documentation/networking/`, and `Documentation/PCI/` in the kernel source tree.

## Contributing

When extending the project, keep each driver focused on its subsystem, document any configuration requirements, and add or update tests for behavioural changes. Include the kernel version and environment when reporting test results so the behaviour can be reproduced.
