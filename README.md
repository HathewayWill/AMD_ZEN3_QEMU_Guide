# Ubuntu Desktop AMD64 on QEMU

A conversational, start-to-finish build and install guide

Host: clean Ubuntu x86_64 • Guest: Ubuntu Desktop 24.04.4 AMD64 • UEFI + KVM + VNC + SSH port forwarding

Updated: May 2026

# 1. What you are building

You will use QEMU on an x86_64 Ubuntu host to install and boot an Ubuntu Desktop AMD64 virtual machine. The VM boots via UEFI firmware, uses virtio devices for disk and networking, uses KVM for hardware acceleration, and exposes SSH to the host through a simple port forward.

This is not ARM emulation. This is an AMD64 guest running on an x86_64 host, so QEMU can use KVM and run much faster than cross-architecture emulation.

The reliable default CPU setting is:

```bash
-cpu host
```

That exposes the real host CPU features to the guest. If the host is an AMD Zen 3 machine, the guest will naturally see AMD CPU features. If you only want the guest to look more like an AMD EPYC Milan CPU, there is an optional section later, but it should not be the default because some hosts and KVM setups reject forced EPYC models.

By the end, you will have:

* A VM disk image (qcow2) containing Ubuntu Desktop AMD64.
* Two UEFI flash images: one for firmware code (read-only) and one for UEFI variables (persistent).
* A repeatable QEMU command line for installer boot over VNC.
* A repeatable QEMU command line for normal boot over VNC and SSH.
* An optional headless command after SSH or serial console access is confirmed.
* Optional AMD Zen 3 style CPU branding.

## Directory layout we will use

Everything lives under your home directory. Example paths:

```text
~/qemu-ubuntu-amd64/                    # VM workspace (you create this)
  efi.img                               # UEFI firmware flash, based on OVMF_CODE_4M.fd
  varstore.img                          # UEFI variable store, based on OVMF_VARS_4M.fd
  disk.qcow2                            # VM disk
  ubuntu-24.04.4-desktop-amd64.iso      # installer ISO
```

The ISO point release may change over time. The commands below use a shell variable so you only need to update the filename in one place.

# 2. Before you start

You will need:

* An x86_64 machine running Ubuntu 24.04 or newer.
* Hardware virtualization enabled in BIOS or UEFI.
* KVM available on the host.
* At least 20 GB free disk space, more if you keep snapshots or install large packages.
* A working Internet connection to download packages and the Ubuntu ISO.

Tip: If your real host CPU is Intel, do not expect KVM to safely pretend it is an AMD CPU in every case. For normal use, keep `-cpu host`. Use the optional `EPYC-Milan` section only after the VM works with the default CPU setting.

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

## 3.2 Install packages (QEMU + KVM + firmware + VNC viewer)

Install the packages required to run QEMU with KVM acceleration, use OVMF UEFI firmware, create qcow2 disks, and connect to the installer over VNC:

```bash
sudo apt install -y \
  qemu-system-x86 \
  qemu-utils \
  qemu-kvm \
  ovmf \
  wget \
  ca-certificates \
  tigervnc-viewer \
  cpu-checker
```

What these are for:

* `qemu-system-x86`: provides `qemu-system-x86_64`, the x86_64 system emulator.
* `qemu-utils`: provides tools such as `qemu-img` for creating and inspecting VM disks.
* `qemu-kvm`: provides KVM integration for hardware-accelerated virtualization.
* `ovmf`: provides x86_64 UEFI firmware files.
* `wget` and `ca-certificates`: download the Ubuntu ISO over HTTPS.
* `tigervnc-viewer`: lets you interact with the Ubuntu Desktop installer over VNC.
* `cpu-checker`: provides `kvm-ok`, a quick KVM availability check.

Check that KVM is available:

```bash
kvm-ok
```

A good result looks like this:

```text
INFO: /dev/kvm exists
KVM acceleration can be used
```

If QEMU later fails with a KVM permission error, add your user to the `kvm` group:

```bash
sudo usermod -aG kvm "$USER"
newgrp kvm
```

Confirm the host architecture:

```bash
uname -m
dpkg --print-architecture
```

Typical expected output on the host:

```text
x86_64
amd64
```

# 4. Use the packaged QEMU build

For this AMD64 VM, the Ubuntu-packaged QEMU is the recommended default. You do not need to build QEMU from source unless you have a specific feature requirement.

Check the QEMU binary:

```bash
qemu-system-x86_64 --version
```

Check that user-mode networking is present:

```bash
qemu-system-x86_64 -help | grep -n "^-netdev user"
```

Example output:

```text
-netdev user,id=str[,ipv4=on|off]...[,...][,hostfwd=rule]...
```

If that check does not print anything, reinstall the Ubuntu QEMU packages:

```bash
sudo apt install --reinstall -y qemu-system-x86 qemu-utils
```

# 5. Create the VM workspace (disk + UEFI flash)

## 5.1 Create a directory for the VM

```bash
mkdir -p ~/qemu-ubuntu-amd64
cd ~/qemu-ubuntu-amd64
pwd
```

Example output:

```text
/home/youruser/qemu-ubuntu-amd64
```

## 5.2 Create a qcow2 disk

qcow2 grows on demand, so the file starts small and expands as the guest writes data.

```bash
qemu-img create -f qcow2 disk.qcow2 80G
```

Example output:

```text
Formatting 'disk.qcow2', fmt=qcow2 cluster_size=65536 extended_l2=off compression_type=zlib size=85899345920 lazy_refcounts=off refcount_bits=16
```

## 5.3 Prepare UEFI firmware flash images

We create two flash images:

* `efi.img`: contains the OVMF UEFI firmware code. We mark it read-only when launching QEMU.
* `varstore.img`: the persistent UEFI variable store (NVRAM). This remembers boot entries and other firmware settings.

Run these commands exactly:

```bash
cp /usr/share/OVMF/OVMF_CODE_4M.fd efi.img
cp /usr/share/OVMF/OVMF_VARS_4M.fd varstore.img
```

Check the files:

```bash
ls -lh efi.img varstore.img disk.qcow2
```

Example output:

```text
-rw-r--r-- 1 youruser youruser 3.5M May 22 12:00 efi.img
-rw-r--r-- 1 youruser youruser 528K May 22 12:00 varstore.img
-rw-r--r-- 1 youruser youruser 196K May 22 12:01 disk.qcow2
```

Do not recreate `varstore.img` after installation unless you intentionally want to reset firmware state.

# 6. Download Ubuntu Desktop AMD64 install media

Set the ISO filename:

```bash
cd ~/qemu-ubuntu-amd64
export ISO="ubuntu-24.04.4-desktop-amd64.iso"
```

Download the ISO:

```bash
wget -O "$ISO" "https://releases.ubuntu.com/noble/$ISO"
```

Confirm the file exists:

```bash
ls -lh "$ISO"
```

Optional checksum verification:

```bash
wget -O SHA256SUMS "https://releases.ubuntu.com/noble/SHA256SUMS"
sha256sum -c SHA256SUMS --ignore-missing
```

You should see an `OK` result for the ISO file.

# 7. Boot the installer (VNC)

Ubuntu Desktop uses a graphical installer. The easiest way to interact with it under QEMU is to expose a local VNC server and connect with a VNC viewer.

## 7.1 Launch QEMU for installation

Run QEMU from your VM directory, where the images and ISO live:

```bash
cd ~/qemu-ubuntu-amd64
export ISO="ubuntu-24.04.4-desktop-amd64.iso"

qemu-system-x86_64 \
  -machine q35 \
  -accel kvm \
  -cpu host \
  -smp 8 \
  -m 8192 \
  -drive if=pflash,format=raw,file=efi.img,readonly=on \
  -drive if=pflash,format=raw,file=varstore.img \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8023-:22 \
  -device virtio-scsi-pci,id=scsi0 \
  -drive if=none,id=cdrom,format=raw,readonly=on,file="$ISO" \
  -device scsi-cd,drive=cdrom \
  -boot order=d \
  -device virtio-vga \
  -device qemu-xhci \
  -device usb-kbd \
  -device usb-tablet \
  -display none \
  -vnc 127.0.0.1:2
```

What to expect:

* The terminal may look idle. That is normal because the UI is on VNC.
* VNC is bound to localhost only (`127.0.0.1`). It is not exposed to the network.
* QEMU VNC display `:2` maps to TCP port `5902`.

## Adjusting CPU cores and memory

You control how large the VM is with two flags:

* CPU cores: `-smp <N>`
* RAM: `-m <MiB>`

Quick examples:

```bash
-smp 2 -m 4096     # 2 cores, 4 GiB RAM
-smp 4 -m 8192     # 4 cores, 8 GiB RAM
-smp 8 -m 32768    # 8 cores, 32 GiB RAM
```

Notes that save time:

* `-m` is in MiB. So `8192` is about 8 GiB, and `32768` is about 32 GiB.
* With KVM on the same architecture, extra vCPUs usually scale much better than ARM64-on-x86 TCG emulation.
* Do not allocate more RAM or vCPUs than the host can comfortably spare.

## 7.2 Connect to the installer over VNC

Open another terminal and run:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5902
```

The `-RemoteResize=0` option prevents the viewer from constantly resizing the guest display.

## 7.3 Install Ubuntu Desktop

Inside the installer:

1. Choose language and keyboard settings.
2. Keep DHCP networking defaults.
3. Use the default guided storage layout unless you need custom partitions.
4. Create your user account.
5. Select the OpenSSH option if the installer offers it.
6. Finish the installation.

If the installer does not offer OpenSSH, install it after the first boot:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

When the installer finishes and reboots, stop QEMU with `Ctrl+C` in the terminal running QEMU. Next, boot without the ISO.

# 8. Boot the installed system (VNC + SSH)

## 8.1 Launch QEMU without the ISO

Use this for the first installed boot because it gives you a visible console if SSH is not ready yet:

```bash
cd ~/qemu-ubuntu-amd64

qemu-system-x86_64 \
  -machine q35 \
  -accel kvm \
  -cpu host \
  -smp 8 \
  -m 8192 \
  -drive if=pflash,format=raw,file=efi.img,readonly=on \
  -drive if=pflash,format=raw,file=varstore.img \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8023-:22 \
  -device virtio-vga \
  -device qemu-xhci \
  -device usb-kbd \
  -device usb-tablet \
  -display none \
  -vnc 127.0.0.1:2
```

Connect to the console:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5902
```

## 8.2 SSH into the guest

From another host terminal:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

Example first connection output:

```text
The authenticity of host '[localhost]:8023 ([127.0.0.1]:8023)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[localhost]:8023' (ED25519) to the list of known hosts.
<your-guest-username>@localhost's password:
```

# 9. Optional headless boot

Do not use headless mode as your first post-install boot unless you already know SSH works. Ubuntu Desktop is not guaranteed to show a useful serial login prompt until you configure one.

## 9.1 Minimal headless command

```bash
cd ~/qemu-ubuntu-amd64

qemu-system-x86_64 \
  -machine q35 \
  -accel kvm \
  -cpu host \
  -smp 8 \
  -m 8192 \
  -drive if=pflash,format=raw,file=efi.img,readonly=on \
  -drive if=pflash,format=raw,file=varstore.img \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8023-:22 \
  -nographic
```

SSH in from another terminal:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

If the VM boots but the terminal shows no login prompt, that usually means the guest was not configured for a serial console. Boot with VNC first, then configure serial output inside the guest:

```bash
sudo systemctl enable serial-getty@ttyS0.service
sudo sed -i 's/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"/' /etc/default/grub
sudo update-grub
sudo reboot
```

After that, retry the headless command.

# 10. Networking choices: user vs TAP/bridge vs passt

This guide uses user-mode networking because it requires no root network setup and works almost anywhere. However, you have options.

## 10.1 Option A (recommended): user-mode networking

Keep these flags:

```bash
-device virtio-net-pci,netdev=net0
-netdev user,id=net0,hostfwd=tcp::8023-:22
```

Pros:

* No host networking configuration.
* Easy outbound Internet access from the guest.
* Easy SSH from the host using `hostfwd`.

Cons:

* NAT-like behavior.
* Inbound connections require explicit forwarded ports.
* Advanced networking features are limited compared with bridged networking.

## 10.2 Option B: TAP/bridge networking

Use this when you want the guest to appear more like a normal machine on your LAN. It requires host setup and often root privileges.

Example host setup for a TAP device owned by your user:

```bash
sudo ip tuntap add dev tap0 mode tap user "$USER"
sudo ip link set tap0 up
```

Then replace the user networking flags with:

```bash
-device virtio-net-pci,netdev=net0 \
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no
```

If you want guest Internet through host NAT, you must also enable forwarding and add NAT rules on the host. A full bridge setup depends on your host network configuration.

## 10.3 Option C: passt backend, if present

Some QEMU builds support a `passt` backend. Check with:

```bash
qemu-system-x86_64 -help | grep -n "^-netdev passt"
```

If present, `passt` can be another host-friendly networking option. User-mode networking is still the easiest default.

# 11. CPU choices: reliable default vs AMD-style branding

## 11.1 Recommended default

Use this for the VM to launch reliably with KVM:

```bash
-cpu host
```

This exposes the real host CPU to the guest. On an AMD Zen 3 host, the guest will naturally report AMD CPU features.

## 11.2 Optional AMD Zen 3 style model

If your QEMU supports the model and your KVM setup accepts it, you can try:

```bash
-cpu EPYC-Milan
```

or:

```bash
-cpu EPYC-Milan,vendor=AuthenticAMD
```

Check available EPYC models:

```bash
qemu-system-x86_64 -cpu help | grep -i epyc
```

If QEMU fails to launch with an EPYC model, go back to:

```bash
-cpu host
```

## 11.3 Important CPU note

With KVM, QEMU generally cannot invent a completely different CPU as freely as it can under full software emulation. If the real host CPU is Intel, forcing an AMD vendor string or an EPYC model can fail.

If artificial CPU branding matters more than performance, you can test TCG:

```bash
-accel tcg,thread=multi
```

That will be much slower than KVM. For normal use, keep KVM and use `-cpu host`.

# 12. QEMU flags explained line-by-line

This section explains every QEMU flag used in the installer and normal-boot commands. For each flag we cover what it does, what virtual hardware it represents, and whether you can remove it safely.

| Flag / example | What it does | Maps to virtual hardware? | Can you remove it? |
| --- | --- | --- | --- |
| `-machine q35` | Selects a modern x86_64 machine model with PCIe support. | Yes: board/chipset model. | Keep it. |
| `-accel kvm` | Uses hardware virtualization for speed. | No direct guest device; affects how QEMU executes guest code on the host. | Keep it for performance on x86_64 hosts. |
| `-cpu host` | Exposes the real host CPU model and CPU features to the guest. | Yes: guest CPU model and features. | Keep it as the reliable default. |
| `-smp 8` | Exposes 8 virtual CPUs to the guest. | Yes: vCPU count and topology. | Optional. Adjust to match your host. |
| `-m 8192` | Allocates 8192 MiB of RAM to the guest. | Yes: guest RAM size. | Optional. Adjust to match your host. |
| `-drive if=pflash,format=raw,file=efi.img,readonly=on` | Attaches UEFI firmware code and makes it read-only. | Yes: flash device holding UEFI firmware. | Keep if you want UEFI boot. |
| `-drive if=pflash,format=raw,file=varstore.img` | Attaches the persistent UEFI variable store. | Yes: flash region for UEFI variables. | Strongly recommended. Removing it causes non-persistent firmware state. |
| `-drive if=virtio,format=qcow2,file=disk.qcow2` | Attaches the main VM disk using virtio. | Yes: virtio block disk. | Keep for the installed OS. |
| `-device virtio-net-pci,netdev=net0` | Adds a virtio network card connected to backend `net0`. | Yes: virtio network interface. | Optional only if you do not need networking. |
| `-netdev user,id=net0,hostfwd=tcp::8023-:22` | Creates a user-mode NAT network backend and forwards host TCP 8023 to guest TCP 22. | Host-side network backend, not a guest device. | Optional if you switch to TAP, bridge, or passt. Keep `hostfwd` for easy SSH. |
| `-device virtio-scsi-pci,id=scsi0` | Adds a virtio SCSI controller so the ISO can be attached as a SCSI CD-ROM. | Yes: virtio SCSI controller. | Installer-only in this guide. |
| `-drive if=none,id=cdrom,format=raw,readonly=on,file="$ISO"` | Defines the ISO as a backend named `cdrom`. | Host-side block backend definition. | Installer-only. Remove after installation. |
| `-device scsi-cd,drive=cdrom` | Attaches the ISO backend as a SCSI CD-ROM device. | Yes: CD-ROM device. | Installer-only. Remove after installation. |
| `-boot order=d` | Requests CD-ROM boot first. | Firmware boot behavior. | Installer-only. Remove after installation. |
| `-device virtio-vga` | Provides a display adapter for the VNC console. | Yes: virtual display adapter. | Keep for VNC. Remove for pure SSH/headless use. |
| `-device qemu-xhci` | Adds a USB 3 controller. | Yes: USB controller. | Useful for VNC input. Usually safe to remove for SSH-only use. |
| `-device usb-kbd` | Adds a USB keyboard. | Yes: USB HID keyboard. | Useful for VNC input. Usually safe to remove for SSH-only use. |
| `-device usb-tablet` | Adds an absolute pointing device for better mouse behavior. | Yes: USB HID tablet. | Useful for VNC input. Usually safe to remove for SSH-only use. |
| `-display none` | Prevents QEMU from opening a local GUI window. | Host UI behavior. | Optional. Keep if you only want VNC. |
| `-vnc 127.0.0.1:2` | Starts a VNC server on localhost display `:2`, port `5902`. | Host UI behavior. | Keep for VNC. Remove if using local GUI or headless mode. |
| `-nographic` | Disables graphics and routes serial console to your terminal. | Host console wiring. | Optional. Good after serial console is configured. |

## 12.1 Minimal recommended post-install command (safe defaults)

Once Ubuntu is installed and you mostly live in SSH, you can simplify to the essentials:

```bash
cd ~/qemu-ubuntu-amd64

qemu-system-x86_64 \
  -machine q35 \
  -accel kvm \
  -cpu host \
  -smp 8 -m 8192 \
  -drive if=pflash,format=raw,file=efi.img,readonly=on \
  -drive if=pflash,format=raw,file=varstore.img \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::8023-:22 \
  -nographic
```

This removes installer-only devices, the CD-ROM, and VNC graphics/input, while keeping a stable UEFI + disk + network + SSH workflow. Use the VNC boot command instead if you still need the desktop console.

# 13. Troubleshooting quick hits

## Error: `Could not access KVM kernel module`

Check that virtualization is enabled in BIOS or UEFI, then run:

```bash
kvm-ok
```

If `/dev/kvm` exists but you get a permission error, add your user to the `kvm` group:

```bash
sudo usermod -aG kvm "$USER"
newgrp kvm
```

## Error: `network backend 'user' is not compiled into this binary`

This is unusual with Ubuntu's packaged QEMU. Reinstall QEMU first:

```bash
sudo apt install --reinstall -y qemu-system-x86 qemu-utils
```

Then check again:

```bash
qemu-system-x86_64 -help | grep -n "^-netdev user"
```

## Error: `EPYC-Milan` is not recognized

List available CPU models:

```bash
qemu-system-x86_64 -cpu help | grep -i epyc
```

Use the reliable default if the model is missing or rejected:

```bash
-cpu host
```

## Boot returns to installer

You are still booting with the ISO attached or using the installer command. Boot without these installer-only lines:

```text
-drive if=none,id=cdrom,format=raw,readonly=on,file="$ISO"
-device scsi-cd,drive=cdrom
-boot order=d
```

## SSH does not respond on port 8023

Boot with VNC, log into the guest, and check SSH:

```bash
sudo systemctl status ssh
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

From the host, try again:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

## Port 8023 is already in use

Pick another host SSH port by changing the `hostfwd` rule:

```bash
-netdev user,id=net0,hostfwd=tcp::8024-:22
```

Then connect with:

```bash
ssh -p 8024 <your-guest-username>@localhost
```

## VNC port 5902 is already in use

Pick another VNC display:

```bash
-vnc 127.0.0.1:3
```

Then connect with:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5903
```

## VNC connects but the screen size is wrong

Use:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5902
```

## Headless boot shows a blank terminal

This usually means the guest is not configured to use the serial console. Boot with VNC, then configure serial output inside the guest:

```bash
sudo systemctl enable serial-getty@ttyS0.service
sudo sed -i 's/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"/' /etc/default/grub
sudo update-grub
sudo reboot
```

Then retry the headless command.

# 14. Confirm system architecture and CPU model

This is a quick sanity check that answers three questions:

* Is the host really x86_64 or amd64?
* Is the guest really x86_64 or amd64?
* What CPU vendor and model does the guest see?

Run these on the host:

```bash
uname -m
dpkg --print-architecture
```

Typical expected output on the host:

```text
x86_64
amd64
```

Run these inside the guest, either in the VM console or over SSH:

```bash
uname -m
dpkg --print-architecture
lscpu | egrep 'Vendor ID|Model name|Architecture'
```

Typical expected architecture in the guest:

```text
x86_64
amd64
```

If using `-cpu host` on an AMD host, the guest should report AMD as the CPU vendor. If using `EPYC-Milan` and the VM launches successfully, the guest should report an EPYC Milan style CPU model.

# 15. Optional custom QEMU build

Most users should use Ubuntu's packaged QEMU:

```bash
qemu-system-x86_64
```

Only build QEMU from source if you need a custom version or feature.

Install build dependencies:

```bash
sudo apt install -y \
  git \
  build-essential \
  ninja-build \
  python3 \
  python3-venv \
  pkg-config \
  libglib2.0-dev \
  libpixman-1-dev \
  libslirp-dev
```

Clone QEMU:

```bash
cd ~
git clone https://git.qemu.org/git/qemu.git
cd qemu
```

Configure only the x86_64 system emulator:

```bash
rm -rf build
./configure --target-list=x86_64-softmmu --enable-slirp --enable-kvm
```

Build:

```bash
ninja -C build
```

Verify the binary exists:

```bash
~/qemu/build/qemu-system-x86_64 --version
```

Verify user-mode networking is available:

```bash
~/qemu/build/qemu-system-x86_64 -help | grep -n "^-netdev user"
```

Use the custom binary by replacing `qemu-system-x86_64` with:

```bash
~/qemu/build/qemu-system-x86_64
```

Important: if you configure QEMU only for `aarch64-softmmu`, you build `qemu-system-aarch64`, not `qemu-system-x86_64`. For this guide, you need `x86_64-softmmu`.

# 16. Recommended default path

Use this order when setting up the VM:

1. Install host packages.
2. Confirm KVM works.
3. Create the VM workspace.
4. Create the qcow2 disk.
5. Copy the OVMF UEFI images to `efi.img` and `varstore.img`.
6. Download the Ubuntu Desktop AMD64 ISO.
7. Boot the installer command with VNC.
8. Connect with VNC.
9. Install Ubuntu and enable OpenSSH if available.
10. Stop QEMU after the installer reboots.
11. Boot without the ISO using the VNC boot command.
12. SSH into the guest.
13. Only use the headless command after SSH or serial console output is confirmed.
14. Keep `-cpu host` as the default. Only test `EPYC-Milan` after the VM already works.

# References

Ubuntu Desktop AMD64 ISO directory: [https://releases.ubuntu.com/noble/](https://releases.ubuntu.com/noble/)

Ubuntu Desktop AMD64 ISO used here: [https://releases.ubuntu.com/noble/ubuntu-24.04.4-desktop-amd64.iso](https://releases.ubuntu.com/noble/ubuntu-24.04.4-desktop-amd64.iso)

QEMU user networking documentation: [https://www.qemu.org/docs/master/system/invocation.html#hxtool-4](https://www.qemu.org/docs/master/system/invocation.html#hxtool-4) and [https://www.qemu.org/docs/master/system/net.html](https://www.qemu.org/docs/master/system/net.html)
