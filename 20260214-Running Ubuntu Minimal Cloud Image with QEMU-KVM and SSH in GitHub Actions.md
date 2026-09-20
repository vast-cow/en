---
title: "Running Ubuntu Minimal Cloud Image with QEMU-KVM and SSH in GitHub Actions"
description: "This article explains a GitHub Actions workflow named qemu-kvm-ubuntu-minimal-cloudimg-ssh. The..."
pubDatetime: 2026-02-14T13:52:02.437Z
updatedDate: 2026-02-14T13:52:43.020Z
---

This article explains a GitHub Actions workflow named **`qemu-kvm-ubuntu-minimal-cloudimg-ssh`**. The workflow provisions an Ubuntu 24.04 Minimal Cloud Image inside a virtual machine using QEMU with KVM acceleration, enables SSH access, validates the boot process, and performs cleanup.

The workflow is triggered manually using `workflow_dispatch` and runs on `ubuntu-latest`.

---

## Overview of the Workflow

The job `boot-and-ssh` performs the following high-level steps:

1. Install virtualization and networking dependencies.
2. Ensure access to `/dev/kvm`.
3. Download and cache the Ubuntu minimal cloud image.
4. Generate cloud-init configuration.
5. Create an overlay disk image.
6. Boot the VM with KVM acceleration.
7. Wait for SSH and cloud-init readiness.
8. Connect via SSH and verify the OS.
9. Shut down the VM and upload logs.

---

## Installing Dependencies

The workflow installs required packages:

* `qemu-system-x86`, `qemu-kvm`, `qemu-utils`
* `cloud-image-utils`, `genisoimage`
* `openssh-client`, `netcat-openbsd`, `curl`
* `util-linux` (for `sg` command)

It verifies:

* `sg` availability
* QEMU version
* Presence of `/dev/kvm`

This ensures the runner supports hardware virtualization.

---

## Enabling KVM Access

Access to `/dev/kvm` is required for hardware acceleration.

The workflow:

* Checks that `/dev/kvm` exists.
* Adds the current user to the `kvm` group.
* Uses `sg kvm -c 'command'` to execute commands with KVM group permissions inside the CI environment.

Because group changes do not affect the current shell session immediately, `sg` is used to enforce group membership at runtime.

---

## Preparing the Ubuntu Minimal Image

The workflow uses the Ubuntu 24.04 (Noble) Minimal Cloud Image:

* URL:

  ```
  https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img
  ```

### Caching Strategy

The image is cached using `actions/cache@v4`:

* Prevents repeated downloads.
* Reduces CI execution time.
* Uses a key based on OS and image name.

If the cache is missed, the image is downloaded with `curl`.

---

## Cloud-Init Configuration

Cloud-init is used to provision the VM automatically.

### SSH Key Generation

An ED25519 SSH keypair is generated:

* Private key: `id_ed25519`
* Public key inserted into cloud-init config

### user-data

Defines:

* A user `ubuntu`
* Passwordless sudo
* SSH key authentication only
* Root login disabled
* A marker file `/var/tmp/cloud-init-ready` created via `runcmd`

This marker file is later used to detect that initialization is complete.

### meta-data

Defines:

* `instance-id`
* `local-hostname`

### Creating Seed ISO

`cloud-localds` generates a `seed.iso` containing:

* `user-data`
* `meta-data`

This ISO is attached as a virtual CD-ROM to the VM.

---

## Creating the Overlay Disk

Instead of modifying the base image directly:

* A QCOW2 overlay image (`ubuntu-minimal.qcow2`) is created.
* It uses the base cloud image as backing storage.
* The overlay is sized at 20GB.

This approach preserves the original image and allows ephemeral usage in CI.

---

## Booting the VM with QEMU and KVM

The VM is started with:

* `-machine accel=kvm`
* `-cpu host`
* 2 vCPUs
* 2 GB RAM
* Virtio disk and network devices
* User-mode networking with port forwarding:

  ```
  hostfwd=tcp::2222-:22
  ```

This maps:

```
localhost:2222 → guest:22
```

The process is started in the background with `nohup`, and the PID is stored.

If QEMU exits unexpectedly, logs are printed and the workflow fails.

---

## Waiting for SSH Availability

Two readiness checks are performed:

### 1. Port Check

Using `nc`:

* Polls `127.0.0.1:2222`
* Confirms SSH port is open

### 2. Cloud-Init Completion Check

Using SSH:

* Attempts login with generated private key
* Checks for existence of:

  ```
  /var/tmp/cloud-init-ready
  ```

This ensures:

* The system booted.
* Cloud-init finished execution.
* The user account is ready.

---

## Verifying the OS

Once SSH is available, the workflow runs:

```
cat /etc/os-release
```

This confirms the running OS is Ubuntu 24.04 Minimal.

---

## Cleanup Procedure

A cleanup step always runs:

* Reads the QEMU PID.
* Sends `kill`.
* Escalates to `kill -9` if needed.
* Ensures no orphan processes remain.

This prevents resource leakage on the GitHub runner.

---

## Log Upload

The workflow uploads `vm/qemu.log` as an artifact using:

```
actions/upload-artifact@v4
```

This allows inspection of:

* Boot logs
* Kernel output
* QEMU errors

Even if the job fails, logs remain accessible.

---

## Key Design Characteristics

| Feature                       | Implementation                 |
| ----------------------------- | ------------------------------ |
| Hardware Acceleration         | `/dev/kvm` + `sg kvm`          |
| Immutable Base Image          | QCOW2 overlay disk             |
| Automated Provisioning        | Cloud-init seed ISO            |
| Secure SSH Access             | ED25519 key, password disabled |
| Deterministic Boot Validation | Marker file from `runcmd`      |
| CI Efficiency                 | GitHub Actions cache           |
| Failure Observability         | Log upload                     |

---

## Conclusion

This workflow demonstrates a reproducible method to:

* Boot an Ubuntu Minimal cloud image inside GitHub Actions.
* Use KVM acceleration safely in CI.
* Configure the VM non-interactively using cloud-init.
* Validate readiness over SSH.
* Perform controlled cleanup.

It provides a compact and production-grade pattern for testing virtualization, OS provisioning, or infrastructure workflows directly within a CI pipeline.

```yml
name: qemu-kvm-ubuntu-minimal-cloudimg-ssh

on:
  workflow_dispatch:

defaults:
  run:
    shell: /usr/bin/bash --noprofile --norc -e -o pipefail {0}

jobs:
  boot-and-ssh:
    runs-on: ubuntu-latest

    env:
      UBUNTU_MINIMAL_IMG_URL: https://cloud-images.ubuntu.com/minimal/releases/noble/release/ubuntu-24.04-minimal-cloudimg-amd64.img
      UBUNTU_MINIMAL_IMG_NAME: ubuntu-24.04-minimal-cloudimg-amd64.img

    steps:
      - name: Install deps (qemu/kvm + cloud-init + sg)
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            util-linux \
            qemu-system-x86 qemu-utils qemu-kvm \
            cloud-image-utils genisoimage \
            openssh-client netcat-openbsd curl

          command -v sg
          qemu-system-x86_64 --version

      - name: Ensure KVM access via kvm group + sg
        run: |
          test -e /dev/kvm
          ls -l /dev/kvm
          id

          # Adding the user to the kvm group won't affect the current shell session immediately.
          sudo usermod -aG kvm "$USER" || true

          # Use sg to run a command with the kvm group in CI.
          sg kvm -c 'id && ls -l /dev/kvm'

      - name: Prepare workspace
        run: |
          mkdir -p vm/cache

      - name: Restore cached Ubuntu minimal image
        id: cache-ubuntu-img
        uses: actions/cache@v4
        with:
          path: vm/cache/${{ env.UBUNTU_MINIMAL_IMG_NAME }}
          key: ubuntu-minimal-${{ runner.os }}-${{ env.UBUNTU_MINIMAL_IMG_NAME }}

      - name: Download Ubuntu Minimal Cloud Image (only if cache miss)
        if: steps.cache-ubuntu-img.outputs.cache-hit != 'true'
        run: |
          curl -fsSL -o "vm/cache/${UBUNTU_MINIMAL_IMG_NAME}" "${UBUNTU_MINIMAL_IMG_URL}"
          ls -lh "vm/cache/${UBUNTU_MINIMAL_IMG_NAME}"

      - name: Prepare cloud-init seed + overlay disk
        run: |
          cd vm

          ssh-keygen -t ed25519 -N "" -f id_ed25519 <<<y >/dev/null 2>&1

          cat > user-data <<'EOF'
          #cloud-config
          users:
            - name: ubuntu
              sudo: ["ALL=(ALL) NOPASSWD:ALL"]
              groups: [sudo]
              shell: /bin/bash
              ssh_authorized_keys:
                - __SSH_PUBKEY__
          ssh_pwauth: false
          disable_root: true
          package_update: false
          packages: []
          runcmd:
            - [ sh, -c, "echo cloud-init-ready > /var/tmp/cloud-init-ready" ]
          EOF

          PUBKEY="$(cat id_ed25519.pub)"
          sed -i "s|__SSH_PUBKEY__|${PUBKEY}|g" user-data

          cat > meta-data <<'EOF'
          instance-id: gh-actions-qemu
          local-hostname: gh-actions-qemu
          EOF

          cloud-localds -v seed.iso user-data meta-data

          BASE_IMG="cache/${UBUNTU_MINIMAL_IMG_NAME}"
          test -f "${BASE_IMG}"
          qemu-img create -f qcow2 -F qcow2 -b "${BASE_IMG}" ubuntu-minimal.qcow2 20G

      - name: Boot VM with KVM (SSH forwarded to localhost:2222) [sg kvm + exec bash]
        run: |
          cd vm

          # Fail fast if port 2222 is already in use
          (ss -ltnp | grep ':2222' && echo "port 2222 already in use" && exit 1) || true

          : > qemu.log

          # Run QEMU under the kvm group using sg, and exec into a bash that runs the command sequence.
          sg kvm -c 'exec /usr/bin/bash -lc "
            set -euo pipefail

            nohup qemu-system-x86_64 \
              -machine accel=kvm \
              -cpu host \
              -smp 2 \
              -m 2048 \
              -nographic \
              -drive file=ubuntu-minimal.qcow2,format=qcow2,if=virtio \
              -drive file=seed.iso,media=cdrom \
              -netdev user,id=net0,hostfwd=tcp::2222-:22 \
              -device virtio-net-pci,netdev=net0 \
              >> qemu.log 2>&1 &

            echo \$! > qemu.pid
            test -s qemu.pid
          "'

          sleep 2
          PID="$(cat qemu.pid)"

          if ! ps -p "$PID" >/dev/null 2>&1; then
            echo "QEMU exited immediately. Dumping qemu.log:"
            echo "----------------------------------------"
            tail -n 200 qemu.log || true
            echo "----------------------------------------"
            exit 1
          fi

          ps -p "$PID" -o pid,cmd

      - name: Wait for SSH to become ready
        run: |
          cd vm

          for i in {1..120}; do
            if nc -z 127.0.0.1 2222; then
              echo "SSH port is open."
              break
            fi
            sleep 2
          done

          SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o ConnectTimeout=5"
          for i in {1..120}; do
            if ssh ${SSH_OPTS} -i id_ed25519 -p 2222 ubuntu@127.0.0.1 "test -f /var/tmp/cloud-init-ready"; then
              echo "cloud-init is ready."
              break
            fi
            sleep 2
          done

      - name: SSH and cat /etc/os-release
        run: |
          cd vm
          SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
          ssh ${SSH_OPTS} -i id_ed25519 -p 2222 ubuntu@127.0.0.1 "cat /etc/os-release"

      - name: Cleanup (stop QEMU)
        if: always()
        run: |
          if [ -f vm/qemu.pid ]; then
            PID="$(cat vm/qemu.pid || true)"
            if [ -n "${PID}" ] && ps -p "${PID}" >/dev/null 2>&1; then
              kill "${PID}" || true
              sleep 2
              ps -p "${PID}" >/dev/null 2>&1 && kill -9 "${PID}" || true
            fi
          fi

      - name: Upload qemu.log (always)
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: qemu-log
          path: vm/qemu.log
```
