---
title: "Installing Docker CE in CentOS 8"
date: 2020-04-24T08:01:42Z
tags: 
  - tech-notes
  - docker
  - centos
---

## Installation
Let's install the Docker Community Edition. Easy:

```
$ sudo dnf install docker-ce
Error: Unable to find a match: docker-ce
```

Oh.

Installation being harder than it needs to be will be a consistent theme. Why must this be a pain?
It's unclear to me, but it's hard not to be suspicious that container politics isn't playing into it.

Neither Red Hat nor Docker support installing Docker on RHEL/CentOS 8. Red Hat would prefer you
use `podman` and `buildah`, while Docker would prefer you go with Docker Enterprise Edition
or use a different distro. Let's fight against the current and try to shove the Docker CE package for
CentOS 7 into our CentOS 8. There will be rough edges here.

We'll start by configuring `dnf` to add and enable Docker's repository.

```
# make sure you have the config-manager installed
$ sudo dnf install dnf-plugins-core 
$ sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
Adding repo from: https://download.docker.com/linux/centos/docker-ce.repo
```

Great. Now that we're ready to go...

```
$ sudo dnf install docker-ce
...
Error: 
 Problem: package docker-ce-3:19.03.8-3.el7.x86_64 requires containerd.io >= 1.2.2-3, but none of the providers can be installed
  - cannot install the best candidate for the job
  - package containerd.io-1.2.10-3.2.el7.x86_64 is excluded
  - package containerd.io-1.2.13-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.2-3.3.el7.x86_64 is excluded
  - package containerd.io-1.2.2-3.el7.x86_64 is excluded
  - package containerd.io-1.2.4-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.5-3.1.el7.x86_64 is excluded
  - package containerd.io-1.2.6-3.3.el7.x86_64 is excluded
(try to add '--skip-broken' to skip uninstallable packages or '--nobest' to use not only best candidate packages)
```

Why is `containerd.io` cryptically marked as excluded? Some nefarious Red Hat conspiracy? The underlying issue
is that Docker's `containerd.io` package provides `runc`, which is also provided by `runc` package
in Red Hat's `container-tools` module. There is no guarantee that they're compatible.

The usual suggestion is the one suggested by `dnf` itself, which is to use `--nobest`. This installs `docker-ce-3:18.X` instead of `docker-ce-3:19.X`,
which allows falling back to the nonexcluded but older `containerd.io-1.2.0`. Additionally, `--nobest` breaks `dnf update`.

A better alternative is to disable the `container-tools` module. We first install `container-selinux` from it, to satisfy a dependency of `docker-ce`:

```
$ sudo dnf install container-selinux
$ sudo dnf module disable container-tools
```

Disabling `container-tools` is a workaround, but it's a better workaround than `--nobest`.

We can now install `docker-ce` in a (relatively) clean way:

```
$ sudo dnf install docker-ce
```

## Enabling the Docker daemon on systemd

Now, enable and start the `dockerd` service in `systemd`:

```
$ sudo systemctl enable docker
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /usr/lib/systemd/system/docker.service
$ sudo systemctl start docker
```

If you went with the `--nobest` approach above, you may run into issues here. I originally did this,
and I ended up hitting some dependency issues with `containerd`.

```
$ sudo systemctl start docker
A dependency job for docker.service failed. See 'journalctl -xe' for details.
$ journalctl -xe
...
Apr 22 15:56:47 host modprobe[2066]: modprobe: FATAL: Module overlay not found in directory /lib/modules/4.19.98-08076-g24ab33fb8e14
...
Apr 22 15:56:47 host systemd[1]: containerd.service: Control process exited, code=exited status=1
Apr 22 15:56:47 host systemd[1]: containerd.service: Failed with result 'exit-code'.
Apr 22 15:56:47 host systemd[1]: Failed to start containerd container runtime.
...
Apr 22 15:56:47 host systemd[1]: Dependency failed for Docker Application Container Engine.
...
Apr 22 15:56:47 host systemd[1]: docker.service: Job docker.service/start failed with result 'dependency'.
```

Before `systemd` starts `containerd`, it asks the kernel to load the `overlay` module, which fails on my system.

```
$ cat /lib/systemd/system/containerd.service
...
[Service]
ExecStartPre=/sbin/modprobe overlay
...
```

Creating an override file for `containerd` and resetting the `ExecStartPre` entry fixes the issue.

```
$ sudo systemctl edit containerd.service
[Service]
ExecStartPre=
```

More recent versions of `containerd` do not have this issue.

At this point, Docker should be up and running for you. If you have `firewalld` running, it may block DNS.
You may want to add your user to the `docker` group, which allows you to use the `docker` cli without `sudo`.

```
$ usermod -aG docker username
```

Logout and log back in afterwards to see the change take effect, and you're good to go.
