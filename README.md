# Ubuntu Server AMD64 “Zen 3” on QEMU

A conversational, start-to-finish build and install guide

Host: clean Ubuntu x86_64 • Guest: Ubuntu Server 24.04 AMD64 • UEFI + SSH port forwarding • CPU presented as AMD Zen 3–class

Updated: May 2026

# 1. What you are building

You will use QEMU on an x86_64 Ubuntu host to install and boot an **amd64** Ubuntu Server VM. The VM boots via **UEFI (OVMF)**, uses virtio devices for disk and networking, and exposes SSH to the host via a simple port forward.

By the end, you will have:

* A QEMU binary that supports `-netdev user` (user-mode networking via SLiRP).
* A VM disk image (qcow2) containing Ubuntu Server.
* Two UEFI firmware files: one for firmware code (read-only) and one for UEFI variables (persistent).
* A repeatable QEMU command line for installer boot (VNC) and normal boot (terminal + SSH).
* A guest CPU model configured to **look like Zen 3** (typically `EPYC-Milan`, vendor `AuthenticAMD`).

## Directory layout we will use

Everything lives under your home directory. Example paths:

```text
~/qemu/                          # (optional) QEMU source tree
~/qemu/build/                    # (optional) QEMU build output
~/qemu-ubuntu-zen3/              # VM workspace (you create this)
  OVMF_CODE.fd                   # UEFI firmware code (read-only)
  OVMF_VARS.fd                   # UEFI variable store (persistent)
  disk.qcow2                     # VM disk
  ubuntu-24.04-live-server-amd64.iso  # installer ISO
```

# 2. Before you start

You will need:

* An x86_64 machine running Ubuntu 24.04 (fresh install is fine).
* Hardware virtualization enabled in BIOS/UEFI (VT-x/AMD-V).
* A working Internet connection (to download packages and the ISO).

Tip: This is **not** cross-architecture emulation. Because host and guest are both amd64, you can use **KVM** and get near-native speed.

# 3. Prepare a clean Ubuntu host

## 3.1 Update the host

Open a terminal and run:

```bash
sudo apt update
sudo apt -y upgrade
sudo reboot
```

After reboot, run:

```bash
sudo apt update
```

## 3.2 Install packages (QEMU + firmware + VNC viewer)

Install the packages required to run an amd64 VM with UEFI firmware and a VNC installer UI:

```bash
sudo apt install -y \
  qemu-system-x86 qemu-utils qemu-kvm \
  ovmf \
  libslirp-dev \
  wget ca-certificates \
  tigervnc-viewer \
  cpu-checker
```

Confirm KVM is available:

```bash
kvm-ok
```

If you get permission issues using KVM later:

```bash
sudo usermod -aG kvm "$USER"
newgrp kvm
```

# 4. Build QEMU from source (optional, with user networking enabled)

Most people can skip this on Ubuntu 24.04 and use the packaged QEMU. This section exists to mirror the structure of the original guide and to help if you need a custom QEMU build with specific features.

## 4.1 Clone QEMU

```bash
cd ~
git clone https://git.qemu.org/git/qemu.git
cd qemu
```

## 4.2 Configure the build

```bash
rm -rf build
./configure --target-list=x86_64-softmmu --enable-slirp --enable-kvm
```

## 4.3 Build

```bash
ninja -C build
```

## 4.4 Verify the `user` networking backend is present

```bash
./build/qemu-system-x86_64 -help | grep -n "^-netdev user"
```

If you skip section 4, use the system QEMU instead:

```bash
qemu-system-x86_64 --version
```

# 5. Create the VM workspace (disk + UEFI firmware)

## 5.1 Create a directory for the VM

```bash
mkdir -p ~/qemu-ubuntu-zen3
cd ~/qemu-ubuntu-zen3
pwd
```

## 5.2 Create a qcow2 disk

```bash
qemu-img create -f qcow2 disk.qcow2 80G
```

## 5.3 Prepare UEFI firmware files (OVMF)

Copy OVMF firmware files into your VM folder (Ubuntu 24.04 typically ships the 4M variants):

```bash
cp /usr/share/OVMF/OVMF_CODE_4M.fd ./OVMF_CODE.fd
cp /usr/share/OVMF/OVMF_VARS_4M.fd ./OVMF_VARS.fd
```

Check the files:

```bash
ls -lh OVMF_CODE.fd OVMF_VARS.fd disk.qcow2
```

Do not recreate `OVMF_VARS.fd` after installation unless you intentionally want to reset firmware state.

# 6. Download Ubuntu Server AMD64 24.04 install media

Download the Ubuntu Server 24.04 amd64 ISO into your VM directory. Example (URL in a code block):

```bash
cd ~/qemu-ubuntu-zen3
wget -O ubuntu-24.04-live-server-amd64.iso "https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso"
```

Optional checksum verification (example pattern):

```bash
sha256sum ubuntu-24.04-live-server-amd64.iso
```

# 7. Boot the installer (VNC)

Ubuntu Server uses a full-screen installer UI. The easiest way to interact with it under QEMU is to expose a local VNC server and connect with a VNC viewer.

## 7.1 Launch QEMU for installation

Run QEMU from your VM directory (where the images and ISO live).

If you built QEMU in section 4, use `~/qemu/build/qemu-system-x86_64`. Otherwise use `qemu-system-x86_64`.

```bash
qemu-system-x86_64 \
  -machine q35 \
  -accel kvm \
  -cpu EPYC-Milan,vendor=AuthenticAMD \
  -smp 8 \
  -m 8192 \
  -drive if=pflash,format=raw,file=OVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=OVMF_VARS.fd \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8022-:22 \
  -drive file=ubuntu-24.04-live-server-amd64.iso,media=cdrom \
  -device virtio-vga,xres=1920,yres=1080 \
  -display none \
  -vnc 127.0.0.1:1
```

If your QEMU build does not recognize `EPYC-Milan`, list available EPYC models and pick the closest:

```bash
qemu-system-x86_64 -cpu help | grep -i epyc
```

## Adjusting CPU cores and memory (the knobs you’ll change most)

You control “how big” the VM is with **two flags**:

* **CPU cores:** `-smp <N>`
* **RAM:** `-m <MiB>`

### Quick examples

**2 cores, 4 GiB RAM (lightweight testing):**

```bash
-smp 2 -m 4096
```

**4 cores, 8 GiB RAM (comfortable for installs + basic dev):**

```bash
-smp 4 -m 8192
```

**16 cores, 32 GiB RAM (heavier workloads):**

```bash
-smp 16 -m 32768
```

### Notes that save time

* `-m` is in **MiB**. So `8192` = ~8 GiB, `32768` = ~32 GiB.
* With **KVM**, adding more cores generally scales far better than TCG emulation.

What to expect:

* The terminal may print little and appear to “hang.” That is normal: the UI is on VNC.
* VNC is bound to localhost only (127.0.0.1). It is not exposed to the network.

## 7.2 Connect to the installer over VNC

```bash
vncviewer -RemoteResize=0 127.0.0.1:5901
```

## 7.3 Install Ubuntu Server

Inside the installer:

1. Choose language, keyboard, and timezone as desired.
2. Partitioning: the default guided install is fine for most cases.
3. Networking: user-mode networking provides outbound Internet access, keep DHCP defaults.
4. SSH setup: select **Install OpenSSH server** so you can SSH in after first boot.
5. Finish installation and allow the system to reboot.

When the installer finishes and reboots, stop QEMU with Ctrl+C in the terminal running QEMU. Next we boot without the ISO.

# 8. Boot the installed system (terminal + SSH)

## 8.1 Launch QEMU (no ISO)

```bash
cd ~/qemu-ubuntu-zen3

qemu-system-x86_64 \
  -machine q35 \
  -accel kvm \
  -cpu EPYC-Milan,vendor=AuthenticAMD \
  -smp 8 \
  -m 8192 \
  -drive if=pflash,format=raw,file=OVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=OVMF_VARS.fd \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8022-:22 \
  -nographic
```

## 8.2 SSH into the guest

From another host terminal:

```bash
ssh -p 8022 <your-username>@localhost
```

# 9. Networking choices: user vs tap/bridge vs passt

This guide uses user-mode networking because it requires no root setup and works anywhere. However, you have options:

## 9.1 Option A (recommended): user-mode networking

Keep these flags:

```text
-device virtio-net-pci,netdev=net0
-netdev user,id=net0,hostfwd=tcp::8022-:22
```

Pros:

* Zero host networking configuration.
* Easy SSH from host using hostfwd.

Cons:

* NAT-like behavior, inbound connections require explicit port forwards.

## 9.2 Option B: TAP/bridge networking (more real networking)

Example host setup (creates tap0 owned by your user):

```bash
sudo ip tuntap add dev tap0 mode tap user "$USER"
sudo ip link set tap0 up
```

Then run QEMU with:

```bash
-device virtio-net-pci,netdev=net0
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no
```

## 9.3 Option C: passt backend (if present)

Some QEMU builds support a `passt` backend. If your help output shows `-netdev passt`, you can use it as another host-friendly approach.

# 10. QEMU flags explained line-by-line (what it maps to, and what you can delete)

This section mirrors the intent of the original guide, but for amd64 + KVM + OVMF.

| Flag / example                                 | What it does                                        | Maps to virtual hardware?               | Can you remove it?                                         |
| ---------------------------------------------- | --------------------------------------------------- | --------------------------------------- | ---------------------------------------------------------- |
| `-machine q35`                                 | Uses the Q35 chipset model (modern PCIe).           | Yes: chipset/platform model.            | Recommended to keep for modern guests.                     |
| `-accel kvm`                                   | Uses hardware virtualization for speed.             | No guest hardware; host execution mode. | If removed, QEMU will fall back to TCG (slow).             |
| `-cpu EPYC-Milan,vendor=AuthenticAMD`          | Presents an AMD Zen 3–class CPU model to the guest. | Yes: guest CPU model/features.          | Optional. Use `-cpu host` if you do not need AMD identity. |
| `-smp N`                                       | Exposes N vCPUs to the guest.                       | Yes: vCPU topology.                     | Optional, but useful.                                      |
| `-m MiB`                                       | Allocates guest RAM.                                | Yes: RAM size.                          | Optional, but useful.                                      |
| `-drive if=pflash,...OVMF_CODE.fd,readonly=on` | UEFI firmware code, read-only.                      | Yes: firmware flash.                    | Keep for UEFI boot.                                        |
| `-drive if=pflash,...OVMF_VARS.fd`             | UEFI variable store (persistent).                   | Yes: NVRAM.                             | Strongly recommended to keep.                              |
| `-drive if=virtio,...disk.qcow2`               | Main VM disk via virtio.                            | Yes: virtio block device.               | Required for installed OS.                                 |
| `-device virtio-net-pci,netdev=net0`           | Virtio NIC.                                         | Yes: NIC.                               | Optional if you do not need networking.                    |
| `-netdev user,...hostfwd=tcp::8022-:22`        | User-mode NAT + SSH port forward.                   | Host-side backend.                      | Optional if you switch networking mode.                    |
| `-drive file=...iso,media=cdrom`               | Attaches installer ISO as CD-ROM.                   | Yes: CD device.                         | Installer-only. Remove after install.                      |
| `-display none -vnc 127.0.0.1:1`               | No local GUI, expose VNC for installer.             | Host UI.                                | Installer convenience.                                     |
| `-nographic`                                   | Route serial console to terminal.                   | Host console wiring.                    | Great for server usage.                                    |

## 10.1 Minimal recommended post-install command (safe defaults)

```bash
qemu-system-x86_64 \
  -machine q35 \
  -accel kvm \
  -cpu EPYC-Milan,vendor=AuthenticAMD \
  -smp 8 -m 8192 \
  -drive if=pflash,format=raw,file=OVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=OVMF_VARS.fd \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8022-:22 \
  -nographic
```

# 11. Troubleshooting quick hits

## `EPYC-Milan` is not recognized

List available EPYC models:

```bash
qemu-system-x86_64 -cpu help | grep -i epyc
```

Replace `EPYC-Milan` with the closest model listed.

## SSH does not respond on port 8022

Inside the guest:

```bash
sudo systemctl status ssh
sudo systemctl enable --now ssh
```

## VNC connects but the UI is the wrong size

Use a fixed framebuffer size and disable remote resize:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5901
```

# 12. Confirm system architecture (host vs guest)

Run these on the **host**:

```bash
uname -m
dpkg --print-architecture
```

Typical expected output on the host:

* `uname -m` -> `x86_64`
* `dpkg --print-architecture` -> `amd64`

Now run the same commands **inside the guest**:

```bash
uname -m
dpkg --print-architecture
lscpu | egrep 'Vendor ID|Model name'
```

Typical expected output in the guest:

* `uname -m` -> `x86_64`
* `dpkg --print-architecture` -> `amd64`
* `Vendor ID` -> `AuthenticAMD`

# References

Ubuntu 24.04 Server amd64 download page (URL in a code block):

```text
https://releases.ubuntu.com/24.04/
```

QEMU documentation (invocation and networking):

```text
https://www.qemu.org/docs/master/system/invocation.html
https://www.qemu.org/docs/master/system/net.html
```

---
