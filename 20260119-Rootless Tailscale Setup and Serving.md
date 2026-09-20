---
title: "Rootless Tailscale Setup and Serving"
description: "1) Assumption: rootless mode requires userspace networking   Without root, you generally..."
pubDatetime: 2026-01-19T15:33:21.445Z
---

## 1) Assumption: rootless mode requires userspace networking

Without root, you generally cannot use a TUN device, so you run `tailscaled` in **userspace networking** mode.



## 2) Start `tailscaled` as a normal user (no sudo)

Set `XDG_DATA_HOME` so state is written under the current directory:

```bash
export XDG_DATA_HOME="$PWD/.xdg"
mkdir -p "$XDG_DATA_HOME"
```

Store the control socket in the current directory as well:

```bash
./tailscaled --tun=userspace-networking --socket="./tailscaled.sock" --verbose=1
```

Check it’s running:

```bash
./tailscale --socket="./tailscaled.sock" status
```



## 3) Connect to your tailnet

Browser-based login:

```bash
./tailscale --socket="./tailscaled.sock" up
```

Or using an auth key:

```bash
./tailscale --socket="./tailscaled.sock" up --authkey tskey-auth-XXXX
```


<!-- 
## 4) `tailscale serve` examples

### 4-1) HTTP serving with NO SSL (plain HTTP)

Use `http` (not `https`):

```bash
./tailscale --socket="./tailscaled.sock" serve http / http://127.0.0.1:3000
```

Access from another device in the same tailnet:

* `http://<your-hostname>/`

(Use `http://` explicitly so your browser doesn’t switch to HTTPS.)

 -->

## 5) Raw TCP (not HTTP) with `tailscale serve`

TCP forwarding:

```bash
./tailscale --socket="./tailscaled.sock" serve --tcp 11434 tcp://127.0.0.1:11434
```

If you are forwarding the same port on localhost, a shorter form may work:

```bash
./tailscale --socket="./tailscaled.sock" serve --tcp 11434 11434
```

Test from another tailnet device:

```bash
nc -vz <hostname-or-tailnet-ip> 11434
```



## 6) Check status / disable serving

```bash
./tailscale --socket="./tailscaled.sock" serve status
./tailscale --socket="./tailscaled.sock" serve off
```
