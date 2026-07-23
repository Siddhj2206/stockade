# Fedora and WSL2 runtime prerequisites

**Scope.** Stockade is a rootful, trusted-workload educational runtime. It requires
Linux cgroup v2 `cpu`, `memory`, and `pids` controls. Current Fedora is the
authoritative demo host; Ubuntu under WSL2 is compatibility-tested only.

## Decision

Support a host only when the preflight below succeeds. Do not treat “Linux” or
“Ubuntu on Windows” alone as sufficient: cgroup-controller availability,
delegation, and mount-namespace privileges are runtime properties.

| Concern            | Current Fedora                                                                                                                                                                                                                                                           | Ubuntu under WSL2                                                                                                                                                                                                                                                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| cgroup hierarchy   | Fedora has used cgroup v2 by default since Fedora 31; current Fedora is the expected demo target. [Fedora cgroups-v2 change][fedora-v2]                                                                                                                                  | Require **WSL 2**, not WSL 1. WSL 2 uses a managed VM with a full Linux kernel and system-call compatibility; WSL 1 does not support systemd. [Microsoft: compare WSL versions][wsl-compare]                                                                                                                                        |
| Controllers        | `cpu`, `memory`, and `pids` must appear in `cgroup.controllers` _and_ be delegated/enabled along Stockade’s parent path. Available controllers are not enabled by default and distribution is top-down. [Linux cgroup v2][kernel-cgv2]                                   | Same Linux cgroup-v2 requirement. Microsoft ships the WSL kernel independently of the Ubuntu userspace, so probe this WSL instance rather than pinning an Ubuntu release or kernel version. [Microsoft: WSL kernel release notes][wsl-kernel]                                                                                       |
| systemd/delegation | systemd owns the root cgroup tree. A runtime must use a delegated unit/subtree rather than write arbitrary cgroups below `/sys/fs/cgroup`. `Delegate=` grants a unit a private subtree and can enable selected controllers. [systemd resource control][systemd-resource] | Require systemd as PID 1. Microsoft requires WSL `>= 0.67.6`; enable it with `/etc/wsl.conf`, `[boot] systemd=true`, then `wsl.exe --shutdown`. Ubuntu/Debian also need `systemd` and `systemd-sysv`. [Microsoft: systemd in WSL][wsl-systemd]                                                                                      |
| Rootful isolation  | Run as host root (or equivalent initial-user-namespace privilege). Creating mount/PID/network/cgroup namespaces and mounting filesystems requires `CAP_SYS_ADMIN`; `mount(2)` also requires it. [Linux `unshare(2)`][unshare] [Linux `mount(2)`][mount]                  | Same. WSL 2’s full kernel compatibility is necessary, but successful privilege/mount probes are still required. Keep runtime state and root filesystems on the Linux filesystem (for example `~/…`), not `/mnt/c`; Microsoft documents slower cross-OS filesystem access with WSL 2. [Microsoft: compare WSL versions][wsl-compare] |
| `debootstrap`      | Available from Fedora repositories (`debootstrap` package). [Fedora Packages][fedora-debootstrap]                                                                                                                                                                        | Available in supported Ubuntu releases, including 24.04 LTS (`apt install debootstrap`). [Ubuntu Packages][ubuntu-debootstrap]                                                                                                                                                                                                      |

## Cgroup-v2 and delegation implications

1. A single cgroup-v2 hierarchy may be mounted at `/sys/fs/cgroup`; a
   controller present in `cgroup.controllers` is **available**, not necessarily
   enabled for child cgroups. Enablement is written to
   `cgroup.subtree_control` and can only flow from parent to child. [Linux
   cgroup v2][kernel-cgv2]
2. A cgroup with enabled controllers must not also contain the runtime’s
   processes if it is to have controlled child cgroups: first create/move into
   children, then enable controllers. [Linux cgroup v2][kernel-cgv2]
3. With systemd, do not make Stockade a second writer of the host hierarchy.
   Launch it in a unit delegated with exactly `cpu memory pids` (or use a
   systemd API/unit designed by the runtime). `Delegate=` establishes ownership
   below that unit and permits the delegatee to build the subtree. [systemd
   resource control][systemd-resource]
4. A fresh mount namespace initially copies the parent’s mount list; mount
   events can propagate through shared mounts. Before bind mounts, `pivot_root`,
   or teardown, make the namespace recursively private. `unshare(1)` defaults
   to doing so for a new mount namespace, but the runtime should explicitly
   check/enforce its own intended behavior. [Linux mount namespaces][mount-ns]

## Actionable preflight

Run these checks as the account that will start Stockade. They are deliberately
capability checks, not a distro/version allow-list.

```sh
# 1. Rootful identity and unified cgroup v2.
test "$(id -u)" = 0 || { echo 'Stockade requires root'; exit 1; }
test "$(stat -fc %T /sys/fs/cgroup)" = cgroup2fs || {
  echo 'cgroup v2 is not mounted at /sys/fs/cgroup'; exit 1;
}
for controller in cpu memory pids; do
  grep -qw "$controller" /sys/fs/cgroup/cgroup.controllers || {
    echo "missing cgroup-v2 controller: $controller"; exit 1;
  }
done

# 2. Required namespace syscalls and private mount propagation.
unshare --mount --pid --fork --mount-proc true || {
  echo 'cannot create required rootful namespaces'; exit 1;
}
unshare --mount sh -ceu '
  mount --make-rprivate /
  test -r /proc/self/mountinfo
' || { echo 'cannot establish a private mount namespace'; exit 1; }

# 3. Bootstrap tool (install with: dnf install debootstrap; or apt install debootstrap).
command -v debootstrap >/dev/null || {
  echo 'debootstrap is not installed'; exit 1;
}
```

On a systemd host, the runtime also needs a **delegated** parent. The following
is a manual smoke test for a sufficiently recent systemd; it keeps the
supervising command in `payload`, leaving the delegated unit cgroup available
for controlled children. `DelegateSubgroup=` is documented as the mechanism
for this placement. [systemd resource control][systemd-resource]

```sh
sudo systemd-run --quiet --wait --collect --scope \
  --property=Delegate=cpu,memory,pids \
  --property=DelegateSubgroup=payload \
  sh -ceu '
    child=$(awk -F: "$1 == 0 { print \$3 }" /proc/self/cgroup)
    parent=${child%/payload}
    base=/sys/fs/cgroup$parent
    for controller in cpu memory pids; do
      grep -qw "$controller" "$base/cgroup.subtree_control" || {
        echo "+$controller" > "$base/cgroup.subtree_control"
      }
    done
    mkdir "$base/stockade-preflight"
    test -w "$base/stockade-preflight/cgroup.procs"
    rmdir "$base/stockade-preflight"
  '
```

Failure here means the runtime must not claim CPU, memory, or PID enforcement
on that host. On Fedora, investigate the systemd unit/delegation arrangement;
on WSL, first confirm the distro is WSL 2 and systemd is PID 1:

```powershell
wsl.exe --version
wsl.exe -l -v       # the Ubuntu distro must report VERSION 2
```

```sh
ps -p 1 -o comm=    # expected: systemd
systemctl is-system-running --wait
```

WSL systemd services do not keep the WSL VM alive after interactive use ends;
this is expected WSL behavior and is another reason WSL is compatibility-only,
not the authoritative demo host. [Microsoft: systemd in WSL][wsl-systemd]

## Sources

- [Fedora cgroups-v2 change][fedora-v2]
- [Microsoft: compare WSL versions][wsl-compare]
- [Microsoft: systemd in WSL][wsl-systemd]
- [Microsoft: WSL kernel release notes][wsl-kernel]
- [Linux kernel: Control Group v2][kernel-cgv2]
- [systemd: resource control and `Delegate=`][systemd-resource]
- [Linux manual pages: `unshare(2)`][unshare], [`mount(2)`][mount], and [mount namespaces][mount-ns]
- [Fedora `debootstrap` package][fedora-debootstrap] and [Ubuntu 24.04 `debootstrap` package][ubuntu-debootstrap]

[fedora-v2]: https://fedoraproject.org/wiki/Changes/CGroupsV2
[wsl-compare]: https://learn.microsoft.com/en-us/windows/wsl/compare-versions
[wsl-systemd]: https://learn.microsoft.com/en-us/windows/wsl/systemd
[wsl-kernel]: https://learn.microsoft.com/en-us/windows/wsl/kernel-release-notes
[kernel-cgv2]: https://docs.kernel.org/admin-guide/cgroup-v2.html
[systemd-resource]: https://www.freedesktop.org/software/systemd/man/latest/systemd.resource-control.html
[unshare]: https://man7.org/linux/man-pages/man2/unshare.2.html
[mount]: https://man7.org/linux/man-pages/man2/mount.2.html
[mount-ns]: https://man7.org/linux/man-pages/man7/mount_namespaces.7.html
[fedora-debootstrap]: https://packages.fedoraproject.org/pkgs/debootstrap/debootstrap/
[ubuntu-debootstrap]: https://packages.ubuntu.com/noble/debootstrap
