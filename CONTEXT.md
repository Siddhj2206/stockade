# Stockade

A Linux-only educational container runtime project. It provides a controlled environment for starting and managing isolated processes.

## Language

**Host distribution**:
The Linux distribution on which Stockade is installed and runs.
_Avoid_: Host container, base image

**Guest root filesystem**:
The filesystem tree presented to a process running in a Stockade container.
_Avoid_: Host filesystem, container image

**Guest-root switch**:
The change from the host filesystem view to a container's guest root filesystem, performed with `chroot` in an isolated mount namespace.
_Avoid_: Virtual machine boot, `pivot_root`

**Container pseudo-filesystems**:
The isolated `/proc` and minimal `/dev` filesystem views mounted in a container; `/sys` is intentionally absent.
_Avoid_: Host `/proc`, host `/dev`, mounted `/sys`

**Bootstrap**:
The creation of a guest root filesystem using the `stockade init` command and `debootstrap`.
_Avoid_: Image pull, setup script

**Container**:
An isolated process together with its guest root filesystem and allocated kernel resources.
_Avoid_: Virtual machine, application sandbox

**Trusted workload**:
A command whose caller is trusted not to intentionally escape or damage the host; Stockade isolates it for education and resource control, not as a security sandbox.
_Avoid_: Untrusted sandbox workload

**Rootful runtime**:
A runtime invoked with privileges sufficient to create and configure container isolation on behalf of its caller.
_Avoid_: Rootless runtime

**Private network namespace**:
A container network stack isolated from the host network stack, with its loopback interface enabled but no required external connectivity.
_Avoid_: Host network

**Resource policy**:
The CPU, memory, and process-count limits applied to a container through its cgroup v2.
_Avoid_: Resource reservation, container size
