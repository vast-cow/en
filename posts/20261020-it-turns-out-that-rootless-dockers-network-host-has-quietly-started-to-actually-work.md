---
title: "It turns out that Rootless Docker's `--network host` has quietly started to actually work."
description: "Docker Engine 29.5.0 changed rootless mode's --network host to share the host network namespace via RootlessKit's --detach-netns, fixing localhost access."
pubDatetime: 2026-10-20T13:01:00+09:00
---

## Introduction

If you've used `--network host` in Rootless Docker, you might have encountered the following behavior:

```bash
docker run --rm --network host \
  curlimages/curl http://127.0.0.1:8080
```

Even though a service is running on port 8080 on the host, the container cannot connect to it.

This is the behavior where a connection that works in regular Docker (rootful mode) fails in rootless mode.

This was a limitation of the previous Rootless Docker.

However, after investigating, I found that **this limitation was resolved in Docker Engine 29.5.0.**

Previously, specifying `--network host` did not allow sharing the host's network namespace, but now it behaves more like regular Docker.

In this article, I will outline the previous behavior, the changed mechanism, the version boundary, and how to verify the operation.

## In the Previous Rootless Docker, `--network host` Did Not Work as Expected

Docker's `--network host` is an option for containers to share the host's network namespace.

In regular Docker, the behavior is as follows:

- `127.0.0.1` inside the container points to the host's loopback interface.
- Ports that the container listens on can be used directly from the host.
- Communication can be done without NAT or port forwarding.
- Port publishing with `-p` is not necessary.

However, in the previous Rootless Docker, the situation was different.

Rootless Docker uses RootlessKit to run the Docker daemon and containers without using root privileges.

To simplify the previous structure:

```text
Host's Network Namespace
│
└── RootlessKit's Network Namespace
    │
    ├── dockerd
    │
    └── Container (--network host)
```

In this structure, even if you specify `--network host`, the container shares the network namespace on the RootlessKit side.

**Not the host's network namespace.**

Therefore, even if you try to connect to `127.0.0.1:8080` from the container, you cannot connect to the host's `127.0.0.1:8080`.

Also, even if the container listens on a port, that port is not directly exposed to the host.

To expose a service to the host, you usually had to use port forwarding with `-p` instead of `--network host`.

```bash
docker run --rm -p 8080:80 nginx:alpine
```

In other words, in Rootless Docker, the `--network host` specification itself was accepted, but it had a different meaning than in rootful mode.

## The Specification Was Changed in Docker Engine 29.5.0

This behavior was changed in **Docker Engine 29.5.0, released on May 14, 2026.**

The following changes are described in the Networking section of the official release notes:

> Rootless: Properly support `--net=host` and localhost registries.

The related implementation is Moby Pull Request [#47103](https://github.com/moby/moby/pull/47103).

To summarize the changes before and after:

| Item | Before 29.4.3 | After 29.5.0 |
|---|---|---|
| `--network host` specification | Possible | Possible |
| Network namespace shared | RootlessKit side | Host side |
| Access to host's `127.0.0.1` | Generally impossible | Possible |
| Container's listening port | Separate from the host | Shared with the host |
| Port publishing with `-p` | Invalid | Invalid |

The key point is that `--network host` was not newly added, but **the existing option was modified to perform the originally expected behavior.**

## What Changed?

One of the important changes in the implementation is that it now uses RootlessKit's `--detach-netns`.

In Docker Engine 29.5.0's `dockerd-rootless.sh`, there is the following setting:

```bash
: "${DOCKERD_ROOTLESS_ROOTLESSKIT_DETACH_NETNS:=true}"
```

The default for `DOCKERD_ROOTLESS_ROOTLESSKIT_DETACH_NETNS` is now `true`.

With this setting, the Docker daemon is separated from the RootlessKit's network namespace and runs in the host's network namespace.

To simplify the structure after the change:

```text
Host's Network Namespace
│
├── dockerd (rootless)
│
├── Container (--network host)
│
└── Service on the host
```

On the other hand, for containers that use the regular bridge network, the rootless network mechanism is still used.

In other words, rootless mode itself has not been abolished, and not all containers now share the host network.

The change is that when `--network host` is explicitly specified, the host's network namespace can be used.

### You Can Also Revert to the Previous Behavior

Even in Docker Engine 29.5.0 and later, you can disable the new behavior with the following environment variable:

```bash
DOCKERD_ROOTLESS_ROOTLESSKIT_DETACH_NETNS=false
```

Therefore, even after 29.5.0, the previous behavior may occur depending on the startup settings.

## How to Verify the Operation

Let's verify the operation in Rootless Docker on Linux by checking if you can connect to the host's localhost.

### 1. Check the Docker Version

```bash
docker version --format '{{.Server.Version}}'
```

Confirm that it is Docker Engine 29.5.0 or later.

Next, check if it is rootless mode.

```bash
docker info --format '{{json .SecurityOptions}}'
```

If `rootless` is included in the output, it is running in rootless mode.

### 2. Start an HTTP Server on the Host

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

Start an HTTP server on the host's loopback interface.

### 3. Connect from the Container

From another terminal, run the following command:

```bash
docker run --rm --network host \
  curlimages/curl http://127.0.0.1:8080
```

If you are using the default settings for Docker Engine 29.5.0 or later, you should be able to connect to the HTTP server on the host.

In the previous Rootless Docker, this connection would usually fail.

### 4. Compare Network Namespaces

You can also check if the host and container are using the same network namespace.

On the host:

```bash
readlink /proc/self/ns/net
```

On the container:

```bash
docker run --rm --network host alpine \
  readlink /proc/self/ns/net
```

If the same network namespace inode number is displayed in both, it confirms that the network namespace is being shared.

## Security Considerations

Just because `--network host` now behaves more like rootful mode does not mean that the container gains root privileges on the host.

The privilege restrictions of Rootless Docker using the user namespace still exist.

However, network isolation is lost.

For example, be aware of the following:

- You can access services listening on the host's loopback address.
- The container's listening port and the host's service port may conflict.
- Network namespace isolation is lost.
- Privileged port binding still has permission restrictions.

In particular, if you are operating a service that listens on `127.0.0.1` under the assumption that it can only be accessed from processes on the host, be aware that a container with `--network host` specified can also access it.

## Summary

In Docker Engine 29.5.0, the behavior of `--network host` in Rootless Docker has changed.

Previously, it only shared the RootlessKit's internal network namespace, so it could not directly access the host's localhost.

However, in 29.5.0 and later, it can now share the host's network namespace by using RootlessKit's `--detach-netns`.

Personally, I had the perception that `--network host` in Rootless Docker was different from the host, so this change was unexpected.

If you are using a workaround to connect to the host's localhost in Rootless Docker, it may be worth checking the Docker Engine version.

## References

- [Docker Engine v29 Release Notes (29.5.0)](https://docs.docker.com/engine/release-notes/29/#2950)
- [Moby PR #47103 — rootless: properly support --net=host and localhost registries](https://github.com/moby/moby/pull/47103)
- [Docker Engine 29.4.3's dockerd-rootless.sh](https://github.com/moby/moby/blob/docker-v29.4.3/contrib/dockerd-rootless.sh)
- [Docker Engine 29.5.0's dockerd-rootless.sh](https://github.com/moby/moby/blob/docker-v29.5.0/contrib/dockerd-rootless.sh)
- [Docker Official Documentation — Rootless mode](https://docs.docker.com/engine/security/rootless/)
