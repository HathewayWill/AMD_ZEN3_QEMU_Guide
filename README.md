# Ubuntu Server AMD64 on QEMU with an AMD Zen 3 Style CPU

A practical start-to-finish guide for installing and running Ubuntu Server AMD64 in QEMU on an Ubuntu x86_64 host.

Updated: May 2026

## What this guide builds

This guide creates an Ubuntu Server AMD64 virtual machine using QEMU, KVM, UEFI firmware, virtio storage, virtio networking, VNC for installation, and SSH port forwarding for day-to-day access.

The reliable default CPU setting is:

```bash
-cpu host
```

That is the best choice when using KVM because it exposes the real host CPU features to the guest and avoids common launch failures.

If your host is an AMD Zen 3 machine, the guest will naturally see AMD CPU features when using `-cpu host`. If you only want the guest to look like an AMD EPYC Milan CPU, this guide includes an optional section for `EPYC-Milan`, but it should not be the default because some hosts and KVM setups reject it.

By the end, you will have:

* A bootable Ubuntu Server AMD64 VM.
* A qcow2 disk image.
* OVMF UEFI firmware files.
* A VNC installer launch command.
* A normal boot command.
* SSH access from the host using a forwarded port.
* Optional AMD Zen 3 style CPU branding.

## Host and guest assumptions

Host:

* Ubuntu 24.04 or newer.
* x86_64 CPU.
* Hardware virtualization enabled in BIOS or UEFI.
* KVM available.

Guest:

* Ubuntu Server 24.04 LTS AMD64.
* UEFI boot.
* virtio disk and network devices.

This is not ARM emulation. This is an AMD64 guest on an x86_64 host, so KVM can be used for much better performance.

## Directory layout

This guide uses one VM workspace under your home directory:

```text
~/qemu-ubuntu-zen3/
  OVMF_CODE.fd
  OVMF_VARS.fd
  disk.qcow2
  ubuntu-24.04.4-live-server-amd64.iso
  run-installer.sh
  run-vnc.sh
  run-headless.sh
```

The ISO version may change over time. The commands below use a shell variable so you only need to update the filename in one place.

## 1. Prepare the Ubuntu host

Update the host first:

```bash
sudo apt update
sudo apt -y upgrade
sudo reboot
```

After reboot:

```bash
sudo apt update
```

Install QEMU, KVM support, OVMF firmware, tools, and a VNC viewer:

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

Check that KVM is available:

```bash
kvm-ok
```

A good result looks like this:

```text
INFO: /dev/kvm exists
KVM acceleration can be used
```

If you get a permission error later, add your user to the `kvm` group:

```bash
sudo usermod -aG kvm "$USER"
newgrp kvm
```

Also confirm your host architecture:

```bash
uname -m
dpkg --print-architecture
```

Expected output:

```text
x86_64
amd64
```

## 2. Create the VM workspace

Create and enter the workspace:

```bash
mkdir -p ~/qemu-ubuntu-zen3
cd ~/qemu-ubuntu-zen3
```

Create an 80 GB qcow2 disk:

```bash
qemu-img create -f qcow2 disk.qcow2 80G
```

Copy the OVMF UEFI firmware files:

```bash
cp /usr/share/OVMF/OVMF_CODE_4M.fd ./OVMF_CODE.fd
cp /usr/share/OVMF/OVMF_VARS_4M.fd ./OVMF_VARS.fd
```

Check the files:

```bash
ls -lh OVMF_CODE.fd OVMF_VARS.fd disk.qcow2
```

Important: do not replace `OVMF_VARS.fd` after installing Ubuntu unless you intentionally want to reset the VM firmware state.

## 3. Download the Ubuntu Server AMD64 ISO

Set the ISO filename:

```bash
cd ~/qemu-ubuntu-zen3
export ISO="ubuntu-24.04.4-live-server-amd64.iso"
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

## 4. Reliable installer launch with VNC

The installer uses a full-screen text UI. The easiest reliable setup is VNC.

Create an installer launcher:

```bash
cat > run-installer.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

ISO="${ISO:-ubuntu-24.04.4-live-server-amd64.iso}"
QEMU_BIN="${QEMU_BIN:-qemu-system-x86_64}"
SSH_PORT="${SSH_PORT:-8023}"
VNC_DISPLAY="${VNC_DISPLAY:-2}"

cd "$(dirname "$0")"

for file in OVMF_CODE.fd OVMF_VARS.fd disk.qcow2 "$ISO"; do
  if [ ! -e "$file" ]; then
    echo "Missing required file: $file" >&2
    exit 1
  fi
done

exec "$QEMU_BIN" \
  -machine q35 \
  -accel kvm \
  -cpu host \
  -smp 8 \
  -m 8192 \
  -drive if=pflash,format=raw,file=OVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=OVMF_VARS.fd \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::${SSH_PORT}-:22 \
  -device virtio-scsi-pci,id=scsi0 \
  -drive if=none,id=cdrom,format=raw,readonly=on,file="$ISO" \
  -device scsi-cd,drive=cdrom \
  -boot order=d \
  -device virtio-vga \
  -device qemu-xhci \
  -device usb-kbd \
  -device usb-tablet \
  -display none \
  -vnc 127.0.0.1:${VNC_DISPLAY}
EOF

chmod +x run-installer.sh
```

Launch the installer:

```bash
./run-installer.sh
```

The terminal may look idle. That is expected. The installer display is available through VNC.

Connect to VNC from another terminal:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5902
```

QEMU VNC display `:2` means TCP port `5902`.

## 5. Install Ubuntu Server

Inside the installer:

1. Choose language and keyboard settings.
2. Keep DHCP networking defaults.
3. Use the default guided storage layout unless you need custom partitions.
4. Create your user account.
5. Select `Install OpenSSH server`.
6. Finish the installation.

When the installer reboots, stop QEMU with `Ctrl+C` in the terminal running `run-installer.sh`.

After installation, boot without the ISO.

## 6. Normal boot with VNC and SSH

This is the recommended first post-install boot because it gives you a visible console if SSH is not ready yet.

Create a VNC boot launcher:

```bash
cat > run-vnc.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

QEMU_BIN="${QEMU_BIN:-qemu-system-x86_64}"
SSH_PORT="${SSH_PORT:-8023}"
VNC_DISPLAY="${VNC_DISPLAY:-2}"

cd "$(dirname "$0")"

for file in OVMF_CODE.fd OVMF_VARS.fd disk.qcow2; do
  if [ ! -e "$file" ]; then
    echo "Missing required file: $file" >&2
    exit 1
  fi
done

exec "$QEMU_BIN" \
  -machine q35 \
  -accel kvm \
  -cpu host \
  -smp 8 \
  -m 8192 \
  -drive if=pflash,format=raw,file=OVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=OVMF_VARS.fd \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::${SSH_PORT}-:22 \
  -device virtio-vga \
  -device qemu-xhci \
  -device usb-kbd \
  -device usb-tablet \
  -display none \
  -vnc 127.0.0.1:${VNC_DISPLAY}
EOF

chmod +x run-vnc.sh
```

Boot the installed VM:

```bash
./run-vnc.sh
```

Connect to the console:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5902
```

From another host terminal, SSH into the VM:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

Replace `<your-guest-username>` with the user you created during installation.

## 7. Optional headless boot

Do not use headless mode as your first post-install boot unless you already know SSH is working.

Create a headless launcher:

```bash
cat > run-headless.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

QEMU_BIN="${QEMU_BIN:-qemu-system-x86_64}"
SSH_PORT="${SSH_PORT:-8023}"

cd "$(dirname "$0")"

for file in OVMF_CODE.fd OVMF_VARS.fd disk.qcow2; do
  if [ ! -e "$file" ]; then
    echo "Missing required file: $file" >&2
    exit 1
  fi
done

exec "$QEMU_BIN" \
  -machine q35 \
  -accel kvm \
  -cpu host \
  -smp 8 \
  -m 8192 \
  -drive if=pflash,format=raw,file=OVMF_CODE.fd,readonly=on \
  -drive if=pflash,format=raw,file=OVMF_VARS.fd \
  -drive if=virtio,format=qcow2,file=disk.qcow2 \
  -device virtio-net-pci,netdev=net0 \
  -netdev user,id=net0,hostfwd=tcp::${SSH_PORT}-:22 \
  -nographic
EOF

chmod +x run-headless.sh
```

Run it:

```bash
./run-headless.sh
```

SSH in from another terminal:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

If the VM boots but the terminal shows no login prompt, that usually means the guest was not configured for a serial console. Use `run-vnc.sh`, log in through VNC or SSH, and then configure serial output if you want a true terminal console.

To enable a serial console inside the guest:

```bash
sudo systemctl enable serial-getty@ttyS0.service
sudo sed -i 's/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"/' /etc/default/grub
sudo update-grub
sudo reboot
```

After that, `run-headless.sh` should show boot output and a serial login prompt.

## 8. CPU options

### Recommended default

Use this for the VM to launch reliably with KVM:

```bash
-cpu host
```

This exposes the real host CPU to the guest.

### Optional AMD Zen 3 style model

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

### Important CPU note

With KVM, QEMU generally cannot invent a completely different CPU in the same way full software emulation can. If the real host CPU is Intel, forcing an AMD vendor string or an EPYC model can fail. If you need artificial CPU branding more than performance, you can use TCG, but it will be much slower:

```bash
-accel tcg,thread=multi
```

For normal use, keep KVM and use `-cpu host`.

## 9. Changing cores, RAM, SSH port, and VNC display

The launchers support simple environment variables.

Use 4 vCPUs and 8 GiB RAM by editing the scripts:

```bash
-smp 4
-m 8192
```

Use 8 vCPUs and 32 GiB RAM:

```bash
-smp 8
-m 32768
```

Change the SSH host port for one launch:

```bash
SSH_PORT=8024 ./run-vnc.sh
```

Then connect with:

```bash
ssh -p 8024 <your-guest-username>@localhost
```

Change the VNC display for one launch:

```bash
VNC_DISPLAY=3 ./run-vnc.sh
```

Then connect with:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5903
```

## 10. Optional custom QEMU build

Most users should use the Ubuntu package:

```bash
qemu-system-x86_64
```

Only build QEMU from source if you need a custom version.

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

Use the custom binary with the launcher scripts:

```bash
QEMU_BIN=~/qemu/build/qemu-system-x86_64 ./run-installer.sh
```

Important: if you configured QEMU only for `aarch64-softmmu`, you built `qemu-system-aarch64`, not `qemu-system-x86_64`. For this guide, you need `x86_64-softmmu`.

## 11. Networking options

### Recommended: user-mode networking

The guide uses this by default:

```bash
-device virtio-net-pci,netdev=net0
-netdev user,id=net0,hostfwd=tcp::8023-:22
```

Pros:

* No root networking setup.
* Easy outbound Internet access from the guest.
* Easy SSH from the host.

Cons:

* Inbound access requires explicit forwarded ports.
* The guest is behind QEMU NAT.

### TAP networking

Use TAP or bridge networking only if you need the VM to appear like a normal machine on your LAN.

Example TAP setup:

```bash
sudo ip tuntap add dev tap0 mode tap user "$USER"
sudo ip link set tap0 up
```

Then replace the networking flags with:

```bash
-device virtio-net-pci,netdev=net0
-netdev tap,id=net0,ifname=tap0,script=no,downscript=no
```

Bridge setup is more involved and depends on your host network configuration.

## 12. Confirm the guest architecture

Inside the guest, run:

```bash
uname -m
dpkg --print-architecture
lscpu | egrep 'Vendor ID|Model name|Architecture'
```

Expected architecture:

```text
x86_64
amd64
```

If using `-cpu host` on an AMD host, you should see an AMD vendor and model from the host.

If using `EPYC-Milan`, and the VM launches successfully, the CPU model should show an EPYC Milan style CPU.

## 13. Troubleshooting

### QEMU does not launch at all

Run these checks from the VM directory:

```bash
cd ~/qemu-ubuntu-zen3
ls -lh OVMF_CODE.fd OVMF_VARS.fd disk.qcow2 *.iso
kvm-ok
qemu-system-x86_64 --version
```

Then try the safest CPU setting:

```bash
-cpu host
```

Avoid `EPYC-Milan` until the VM launches reliably.

### `EPYC-Milan` is not recognized

List available models:

```bash
qemu-system-x86_64 -cpu help | grep -i epyc
```

Use `-cpu host` if the model is missing or rejected.

### KVM permission denied

Add your user to the `kvm` group:

```bash
sudo usermod -aG kvm "$USER"
newgrp kvm
```

Then retry the launch.

### Port 8023 is already in use

Pick another host SSH port:

```bash
SSH_PORT=8024 ./run-vnc.sh
```

SSH with:

```bash
ssh -p 8024 <your-guest-username>@localhost
```

### VNC port is already in use

Pick another VNC display:

```bash
VNC_DISPLAY=3 ./run-vnc.sh
```

Connect with:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5903
```

### VNC connects but the screen size is wrong

Use:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5902
```

The `-RemoteResize=0` option prevents the viewer from constantly resizing the guest display.

### SSH does not work

Boot with VNC:

```bash
./run-vnc.sh
```

Log in through the VM console, then run:

```bash
sudo systemctl status ssh
sudo systemctl enable --now ssh
```

From the host, try:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

### Headless boot shows a blank terminal

This usually means the guest was not configured to use the serial console. Boot with VNC first:

```bash
./run-vnc.sh
```

Then configure serial output inside the guest:

```bash
sudo systemctl enable serial-getty@ttyS0.service
sudo sed -i 's/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"/' /etc/default/grub
sudo update-grub
sudo reboot
```

Then retry:

```bash
./run-headless.sh
```

### The VM keeps booting back into the installer

Make sure you are using `run-vnc.sh` after installation, not `run-installer.sh`.

The installer launcher attaches the ISO and requests CD-ROM boot. The normal boot launcher does not attach the ISO.

## 14. QEMU flag reference

| Flag                                 | Purpose                                               | Keep it?                   |
| ------------------------------------ | ----------------------------------------------------- | -------------------------- |
| `-machine q35`                       | Uses a modern x86_64 machine model with PCIe support. | Yes                        |
| `-accel kvm`                         | Uses hardware virtualization for speed.               | Yes                        |
| `-cpu host`                          | Exposes the real host CPU to the guest.               | Yes                        |
| `-smp 8`                             | Gives the guest 8 virtual CPUs.                       | Adjust as needed           |
| `-m 8192`                            | Gives the guest 8192 MiB of RAM.                      | Adjust as needed           |
| `-drive if=pflash,...OVMF_CODE.fd`   | UEFI firmware code.                                   | Yes                        |
| `-drive if=pflash,...OVMF_VARS.fd`   | Persistent UEFI variables.                            | Yes                        |
| `-drive if=virtio,...disk.qcow2`     | Main VM disk.                                         | Yes                        |
| `-device virtio-net-pci,netdev=net0` | Virtio network card.                                  | Yes for networking         |
| `-netdev user,...hostfwd=...`        | NAT networking and SSH forwarding.                    | Yes for simple networking  |
| `-device virtio-scsi-pci,id=scsi0`   | SCSI controller for installer CD-ROM.                 | Installer only             |
| `-device scsi-cd,drive=cdrom`        | Attaches the ISO as a CD-ROM.                         | Installer only             |
| `-boot order=d`                      | Boots the installer ISO first.                        | Installer only             |
| `-device virtio-vga`                 | Display adapter for VNC.                              | Yes for VNC                |
| `-display none`                      | Disables a local graphical window.                    | Yes for VNC-only display   |
| `-vnc 127.0.0.1:2`                   | Starts a localhost-only VNC server.                   | Yes for VNC                |
| `-nographic`                         | Uses the terminal instead of a graphical display.     | Optional for headless boot |

## 15. Minimal command examples

### Installer boot

```bash
cd ~/qemu-ubuntu-zen3
./run-installer.sh
```

Connect:

```bash
vncviewer -RemoteResize=0 127.0.0.1:5902
```

### Normal boot with VNC

```bash
cd ~/qemu-ubuntu-zen3
./run-vnc.sh
```

SSH:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

### Normal headless boot

```bash
cd ~/qemu-ubuntu-zen3
./run-headless.sh
```

SSH:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

## 16. Recommended default path

Use this order when setting up the VM:

1. Install packages.
2. Create the VM workspace.
3. Download the Ubuntu Server AMD64 ISO.
4. Run `./run-installer.sh`.
5. Connect with VNC.
6. Install Ubuntu and enable OpenSSH.
7. Stop QEMU after the installer reboots.
8. Run `./run-vnc.sh` for the first installed boot.
9. SSH into the guest.
10. Only use `./run-headless.sh` after SSH or serial console output is confirmed.

Keep `-cpu host` as the default. Only test `EPYC-Milan` after the VM is already working.
