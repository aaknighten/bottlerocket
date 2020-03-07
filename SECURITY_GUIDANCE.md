# Security Guidance

## Overview

Bottlerocket adheres to the [Shared Responsibility Model](https://aws.amazon.com/compliance/shared-responsibility-model/) which defines security and compliance as a shared responsibility between the OS vendor and the customer.

We provide these recommendations, along with [details](#details) and [examples](#examples), to help you create a configuration that meets your security and compliance requirements.

| Recommendation                                                                                      | Priority  |
| :-------------------------------------------------------------------------------------------------- | :-------- |
| [Enable automatic updates](#enable-automatic-updates)                                               | Critical  |
| [Restrict access to the host API socket](#restrict-access-to-the-host-api-socket)                   | Critical  |
| [Restrict access to the container runtime socket](#restrict-access-to-the-container-runtime-socket) | Critical  |
| [Design for host replacement](#design-for-host-replacement)                                         | Important |
| [Limit use of host containers](#limit-use-of-host-containers)                                       | Important |
| [Limit use of privileged SELinux labels](#limit-use-of-privileged-selinux-labels)                   | Important |
| [Limit access to system mounts](#limit-access-to-system-mounts)                                     | Important |
| [Do not run containers in host namespaces](#do-not-run-containers-in-host-namespaces)               | Moderate  |
| [Do not run containers with elevated privileges](#do-not-run-containers-with-elevated-privileges)   | Moderate  |
| [Do not run containers as UID 0](#do-not-run-containers-as-uid-0)                                   | Moderate  |

## Details

### Enable automatic updates

Bottlerocket includes many [security features](SECURITY_FEATURES.md) to mitigate software vulnerabilities.
These countermeasures serve to reduce the reliability of exploits and to raise their cost.
However, it is always better to patch vulnerabilities than to rely on mitigations alone.

We provide [a Kubernetes operator](https://github.com/bottlerocket-os/bottlerocket-update-operator) for automated updates to Bottlerocket.
We recommend deploying it on your Kubernetes clusters.

### Restrict access to the host API socket

The Bottlerocket API server listens for requests on a Unix domain socket.
The canonical location of this socket is `/run/api.sock`.
It is owned by UID 0 (`root`) and GID 274 (`api`).

Write access to this socket will grant full control over system configuration.
This includes the ability to define an arbitrary source for a host container, and to run that container with "superpowers" that bypass other restrictions.
These "superpowers" are described [below](#limit-use-of-host-containers).

We recommend blocking access to the API socket from containers managed by the orchestrator.
The "control" host container can be used to modify settings when needed.

### Restrict access to the container runtime socket

Different [variants](variants/) of Bottlerocket may have different container runtimes installed.
Each container runtime will have its own API and will listen for requests on a Unix domain socket.
The socket will usually be owned by UID 0 (`root`) and GID 0 (`root`).

Some potential locations of container runtime sockets are:
* `/run/dockershim.sock`
* `/run/containerd/containerd.sock`
* `/run/host-containerd/host-containerd.sock`

Write access to any of these sockets will grant full control over container execution.
This includes the ability to run containers with elevated privileges and with access to all filesystem locations.

One common use case for mounting the container runtime socket is to perform container image builds.
Instead of mounting the socket, you can use an image build tool that does not require additional privileges.

We recommend blocking access to the container runtime socket from containers managed by the orchestrator.

### Design for host replacement

One of the main security objectives of Bottlerocket is to harden the OS against persistent threats.
This is closely related to the support for automated, in-place updates.
Applying updates to the same host makes sense if you are confident that the underlying software can still be trusted.

However, containers share the same kernel with the host.
The exposed kernel interface can be minimized through techniques such as seccomp filters, but it cannot be eliminated.
If the kernel is ever compromised through a local exploit, then other defenses may break down.

We recommend designing for periodic host replacement even with automated updates enabled.

### Limit use of host containers

Bottlerocket offers host containers to provide out-of-band access to the underlying host OS.

Host containers can be configured with an optional `superpowered` flag.
This causes the container to run with extra privileges, an unrestricted SELinux label, and additional mounts.
The current implementation can be found in [host-ctr](sources/host-ctr/cmd/host-ctr/main.go).

Two host containers are defined in the default configuration.
The ["control" host container](README.md#control-container) is enabled by default unless otherwise specified.
It provides remote connectivity through the AWS SSM [Session Manager](https://console.aws.amazon.com/systems-manager/session-manager/sessions).
The ["admin" host container](README.md#admin-container) is disabled by default unless otherwise specified.
It can be enabled through the "control" host container, through instance user data, or by accessing the host API socket.

We recommend leaving the "admin" host container disabled until it is necessary to use it.
The "control" host container can also be disabled if you are confident you will not need it.
**This could leave you with no way to access the API and change settings on an existing node!**

If you define your own host container, avoid using `superpowered = true` unless your use case requires an extremely high level of privilege, such as loading an out-of-tree kernel module.

### Limit use of privileged SELinux labels

Bottlerocket enables SELinux in enforcing mode by default.
SELinux works by associating labels with subjects (processes) and objects (such as files).

Labels are "sticky" by default: processes will receive the label of their parent process, and files will receive the label of the directory where they are created.
A process can change its own label or the label of a child process under certain circumstances.
These changes are called "transitions".
The SELinux policy for Bottlerocket defines special transition rules for container runtimes.

A container runtime can transition a child processes to either of these labels:
* `container_t` (the default, for ordinary containers)
* `super_t` (for "superpowered" containers)

Some orchestrators such as Kubernetes allow SELinux labels to be defined in the container specification.
If `super_t` is specified in this way, it will override the default transition rules and the container will run with elevated privileges.

We recommend limiting access to any SELinux label other than `container_t`.

### Limit access to system mounts

Bottlerocket provides a read-only root filesystem and ephemeral mounts for system directories such as `/etc` and `/run`.

The `/etc` directory contains system configuration files generated by the API.
These are regenerated when a setting changes, but otherwise not monitored.
If the contents of this directory are mounted into a privileged container, they can be modified in unexpected ways.
This is not supported and may interfere with the reliability of automated updates.

The `/run` directory contains ephemeral files such as Unix domain sockets used by the API server and the container runtime.
If the contents of this directory are mounted into a privileged container, they can be used to bypass security protections.

We recommend limiting access to all system mounts.

### Do not run containers in host namespaces

Namespaces are one of the key building blocks for Linux containers.

Network namespaces provide isolation for network resources such as IP addresses, ports, and routing tables.
Containers that share the host network namespace can connect to services listening on the host loopback addresses `127.0.0.1` and `::1`.
These services are not otherwise reachable from the network.

PID namespaces provide isolation for the process ID number space.
Containers that share the host PID namespace can interact with processes running on the host.
This includes the ability to send signals to those processes, which may interfere with system functionality.

We recommend that you do not run containers that share any of the host namespaces.

### Do not run containers with elevated privileges

Containers can be made more secure by limiting the capabilities they have, and by filtering syscalls they can make.

Capabilities are a way to split up the traditional powers of the `root` user so that a subset of the permissions can be granted instead.
For example, `CAP_NET_BIND_SERVICE` can be granted to allow binding to a low-numbered port.
Bottlerocket uses `runc` to execute containers with [a subset of Linux capabilities](https://github.com/opencontainers/runc/blob/master/libcontainer/SPEC.md#security).

Syscalls are a way for userspace programs to request services from the kernel.
Seccomp filters can be used to allow access to a subset of syscalls.
Bottlerocket uses `containerd` as the container runtime which provides [a default seccomp profile](https://github.com/containerd/containerd/blob/master/contrib/seccomp/seccomp_default.go).

Orchestrators provide ways to disable these protections.
For example, Docker can run containers with the `--privileged` flag, and Kubernetes can run pods with `privileged: true` set in the pod definition.
By default, Kubernetes also runs pods with no seccomp filter applied.
Pods can specify a seccomp profile, or you can apply a default profile using a [Pod Security Policy](https://kubernetes.io/docs/concepts/policy/pod-security-policy/).

We recommend that you do not run containers with elevated privileges.
The default set of capabilities and the default seccomp filter should be used where possible.

### Do not run containers as UID 0

Bottlerocket does not currently support user namespaces.
This means that UID 0 (`root`) inside the container is the same as UID 0 on the host.

A process in a container that runs as UID 0 will have nearly unlimited access to the host if all of these are true:
* it uses a privileged SELinux label
* it has access to system mounts
* it shares the host namespaces
* it has elevated privileges, with all capabilities and no seccomp filter

This is essentially the configuration that is used for a host container with "superpowers", where `superpowered = true` is set.

We recommend that you do not run containers as UID 0.

## Examples

### EC2

These settings can passed as [user data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html) on EC2 instance launch.
Note that some orchestrator-specific settings are required but not shown.

```
# The admin host container provides SSH access and runs with "superpowers".
# It is disabled by default, but can be disabled explicitly.
[settings.host-containers.admin]
enabled = false

# The control host container provides out-of-band access via SSM.
# It is enabled by default, and can be disabled if you do not expect to use SSM.
# This could leave you with no way to access the API and change settings on an existing node!
[settings.host-containers.control]
enabled = false
```

### Kubernetes

The following [Pod Security Policy](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) is based on our recommendations.
It can be used as a starting point for your own policy.

```
---
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted-psp

  # Ensure that the default seccomp filter is used.
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: 'runtime/default'
    seccomp.security.alpha.kubernetes.io/defaultProfileName: 'runtime/default'

spec:
  # Do not allow containers to run as privileged.
  privileged: false

  # Do not allow containers to gain new privileges.
  allowPrivilegeEscalation: false

  # Remove all capabilities from the default set.
  requiredDropCapabilities:
    - ALL

  # Run all containers with the less privileged container_t label.
  seLinux:
    rule: 'MustRunAs'
    seLinuxOptions:
      user: system_u
      role: system_r
      type: container_t
      level: s0

  # Do not allow containers to run as any system user.
  runAsUser:
    rule: 'MustRunAs'
    ranges:
      - min: 1000
        max: 65535

  # Do not allow containers to run as any system group.
  runAsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1000
        max: 65535

  # Do not allow containers to add other system groups.
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1000
        max: 65535

  # Do not allow containers to use other system groups for volumes.
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1000
        max: 65535

  # Do not allow containers to share host namespaces.
  hostNetwork: false
  hostIPC: false
  hostPID: false

  # Do not allow containers to use or write to host paths.
  allowedHostPaths:
    - pathPrefix: "/tmp"
      readOnly: true

  # Allow minimal set of core volume types.
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
```