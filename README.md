# Go Container Runtime

A minimal Linux container runtime implemented in Go, inspired by [Liz Rice's Containers From Scratch](https://github.com/lizrice/containers-from-scratch).

This project demonstrates how Linux kernel primitives can be combined to build a small container runtime without Docker, containerd, or a high-level container library.

## Features

- UTS namespace isolation for an independent hostname
- PID namespace isolation
- Mount namespace isolation
- `chroot` filesystem isolation
- `proc` filesystem mounting
- `tmpfs` mounting at `/mytemp`
- cgroup process limiting using cgroup v2
- `pids.max` enforcement
- cleanup of temporary mounts after container exit

## How It Works

```text
./container run /bin/bash
        |
        v
      run()
        |
        | exec /proc/self/exe child ...
        | CLONE_NEWUTS
        | CLONE_NEWPID
        | CLONE_NEWNS
        v
     child()
        |
        +--> set hostname
        +--> configure cgroup
        +--> chroot(rootfs)
        +--> mount /proc
        +--> mount tmpfs on /mytemp
        +--> execute requested command
        +--> unmount /proc and /mytemp
        v
      exit
```

The runtime re-executes itself through `/proc/self/exe`, allowing the child process to enter the requested namespaces before setting up the container environment.

## Project Structure

```text
.
├── main.go
├── rootfs/
├── go.mod
├── .gitignore
└── README.md
```

`rootfs/` is generated locally and is intentionally excluded from Git.

## Requirements

This project targets Linux and requires:

- Go
- Linux namespaces
- root privileges
- `debootstrap`
- cgroup v2 with the `pids` controller enabled

Check the cgroup hierarchy with:

```bash
mount | grep cgroup
```

## Creating the Root Filesystem

Create a Debian Bookworm root filesystem:

```bash
sudo debootstrap bookworm rootfs https://deb.debian.org/debian
```

Verify:

```bash
sudo ls -l rootfs/bin/bash
```

## Build

```bash
go mod tidy
go build -o container main.go
```

## Run

```bash
sudo ./container run /bin/bash
```

Inside the container:

```bash
hostname
```

Expected:

```text
container
```

Check the PID namespace:

```bash
cat /proc/1/status | grep NSpid
```

Expected:

```text
NSpid:  1
```

Check the container mounts:

```bash
mount | grep -E 'proc|tmpfs'
```

The runtime mounts:

```text
proc   -> /proc
tmpfs  -> /mytemp
```

## cgroups

The original Containers From Scratch implementation uses a cgroup v1 `pids` hierarchy. The original v1 implementation is retained in the source as a reference, while the current runtime uses cgroup v2 for modern Fedora systems.

The cgroup v2 implementation creates:

```text
/sys/fs/cgroup/go-container/
```

and configures:

```text
pids.max = 20
```

The container process is placed into:

```text
/sys/fs/cgroup/go-container/cgroup.procs
```

While the container is running, verify from another terminal:

```bash
sudo cat /sys/fs/cgroup/go-container/pids.max
```

Expected:

```text
20
```

Check assigned processes:

```bash
sudo cat /sys/fs/cgroup/go-container/cgroup.procs
```

## Testing Process Limits

Inside the container:

```bash
for i in $(seq 1 30); do sleep 100 & done
```

Once the cgroup reaches its process limit, additional process creation fails with an error similar to:

```text
bash: fork: retry: Resource temporarily unavailable
```

The exact number of child processes that can be created is lower than 20 because the container's existing processes also count toward `pids.max`.

### Cleaning Up the cgroup

After the container has exited and no processes remain in the cgroup, the temporary cgroup can be removed:

```bash
sudo cat /sys/fs/cgroup/go-container/cgroup.procs
```

If the file is empty, remove the cgroup with:

```bash
sudo rmdir /sys/fs/cgroup/go-container
```

`rmdir` is preferred over `rm -rf` because it will refuse to remove the cgroup if it is still in use.

## Implementation Details

### Namespaces

The runtime creates:

```go
syscall.CLONE_NEWUTS
syscall.CLONE_NEWPID
syscall.CLONE_NEWNS
```

The mount namespace is also unshared before container setup.

### Filesystem

The child process:

1. Sets the hostname.
2. Configures the cgroup.
3. Calls `chroot()` to change the container root.
4. Changes the working directory to `/`.
5. Mounts `proc`.
6. Mounts `tmpfs` at `/mytemp`.
7. Executes the requested command.

### Cleanup

After the command exits, the runtime unmounts:

```text
/proc
/mytemp
```

before returning to the host.

## Limitations

This is an educational container runtime rather than a production container engine.

It currently does not provide:

- image management
- layered filesystems
- networking
- port forwarding
- capabilities/seccomp configuration
- user namespaces
- persistent container management
- OCI image/runtime compatibility
- daemon or orchestration functionality

The goal is to expose the Linux primitives behind container isolation rather than reproduce the full functionality of Docker or an OCI runtime.

## Acknowledgement

This project was inspired by Liz Rice's **Containers From Scratch**, which demonstrates how Linux namespaces, `chroot`, mounts, and cgroups can be combined to construct a basic container.

Reference:

- [Liz Rice — Containers From Scratch](https://github.com/lizrice/containers-from-scratch)

The implementation here is adapted for a modern Fedora Linux environment, particularly the transition from cgroup v1 to cgroup v2.

# Go Container Runtime

A minimal Linux container runtime implemented in Go, inspired by [Liz Rice's Containers From Scratch](https://github.com/lizrice/containers-from-scratch).

This project demonstrates how Linux kernel primitives can be combined to build a small container runtime without Docker, containerd, or a high-level container library.

## Features

- UTS namespace isolation for an independent hostname
- PID namespace isolation
- Mount namespace isolation
- `chroot` filesystem isolation
- `proc` filesystem mounting
- `tmpfs` mounting at `/mytemp`
- cgroup process limiting using cgroup v2
- `pids.max` enforcement
- cleanup of temporary mounts after container exit

## How It Works

```text
./container run /bin/bash
        |
        v
      run()
        |
        | exec /proc/self/exe child ...
        | CLONE_NEWUTS
        | CLONE_NEWPID
        | CLONE_NEWNS
        v
     child()
        |
        +--> set hostname
        +--> configure cgroup
        +--> chroot(rootfs)
        +--> mount /proc
        +--> mount tmpfs on /mytemp
        +--> execute requested command
        +--> unmount /proc and /mytemp
        v
      exit
```

The runtime re-executes itself through `/proc/self/exe`, allowing the child process to enter the requested namespaces before setting up the container environment.

## Project Structure

```text
.
├── main.go
├── rootfs/
├── go.mod
├── .gitignore
└── README.md
```

`rootfs/` is generated locally and is intentionally excluded from Git.

## Requirements

This project targets Linux and requires:

- Go
- Linux namespaces
- root privileges
- `debootstrap`
- cgroup v2 with the `pids` controller enabled

Check the cgroup hierarchy with:

```bash
mount | grep cgroup
```

## Creating the Root Filesystem

Create a Debian Bookworm root filesystem:

```bash
sudo debootstrap bookworm rootfs https://deb.debian.org/debian
```

Verify:

```bash
sudo ls -l rootfs/bin/bash
```

## Build

```bash
go mod tidy
go build -o container main.go
```

## Run

```bash
sudo ./container run /bin/bash
```

Inside the container:

```bash
hostname
```

Expected:

```text
container
```

Check the PID namespace:

```bash
cat /proc/1/status | grep NSpid
```

Expected:

```text
NSpid:  1
```

Check the container mounts:

```bash
mount | grep -E 'proc|tmpfs'
```

The runtime mounts:

```text
proc   -> /proc
tmpfs  -> /mytemp
```

## cgroups

The original Containers From Scratch implementation uses a cgroup v1 `pids` hierarchy. The original v1 implementation is retained in the source as a reference, while the current runtime uses cgroup v2 for modern Fedora systems.

The cgroup v2 implementation creates:

```text
/sys/fs/cgroup/go-container/
```

and configures:

```text
pids.max = 20
```

The container process is placed into:

```text
/sys/fs/cgroup/go-container/cgroup.procs
```

While the container is running, verify from another terminal:

```bash
sudo cat /sys/fs/cgroup/go-container/pids.max
```

Expected:

```text
20
```

Check assigned processes:

```bash
sudo cat /sys/fs/cgroup/go-container/cgroup.procs
```

## Testing Process Limits

Inside the container:

```bash
for i in $(seq 1 30); do sleep 100 & done
```

Once the cgroup reaches its process limit, additional process creation fails with an error similar to:

```text
bash: fork: retry: Resource temporarily unavailable
```

The exact number of child processes that can be created is lower than 20 because the container's existing processes also count toward `pids.max`.

### Cleaning Up the cgroup

After the container has exited and no processes remain in the cgroup, the temporary cgroup can be removed:

```bash
sudo cat /sys/fs/cgroup/go-container/cgroup.procs
```

If the file is empty, remove the cgroup with:

```bash
sudo rmdir /sys/fs/cgroup/go-container
```

`rmdir` is preferred over `rm -rf` because it will refuse to remove the cgroup if it is still in use.

## Implementation Details

### Namespaces

The runtime creates:

```go
syscall.CLONE_NEWUTS
syscall.CLONE_NEWPID
syscall.CLONE_NEWNS
```

The mount namespace is also unshared before container setup.

### Filesystem

The child process:

1. Sets the hostname.
2. Configures the cgroup.
3. Calls `chroot()` to change the container root.
4. Changes the working directory to `/`.
5. Mounts `proc`.
6. Mounts `tmpfs` at `/mytemp`.
7. Executes the requested command.

### Cleanup

After the command exits, the runtime unmounts:

```text
/proc
/mytemp
```

before returning to the host.

## Limitations

This is an educational container runtime rather than a production container engine.

It currently does not provide:

- image management
- layered filesystems
- networking
- port forwarding
- capabilities/seccomp configuration
- user namespaces
- persistent container management
- OCI image/runtime compatibility
- daemon or orchestration functionality

The goal is to expose the Linux primitives behind container isolation rather than reproduce the full functionality of Docker or an OCI runtime.

## Acknowledgement

This project was inspired by Liz Rice's **Containers From Scratch**, which demonstrates how Linux namespaces, `chroot`, mounts, and cgroups can be combined to construct a basic container.

Reference:

- [Liz Rice — Containers From Scratch](https://github.com/lizrice/containers-from-scratch)

The implementation here is adapted for a modern Fedora Linux environment, particularly the transition from cgroup v1 to cgroup v2.

