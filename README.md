# Ubuntu Desktop AMD64 on QEMU with SPICE

A conversational, start-to-finish build and install guide

Host: clean Ubuntu x86_64 • Guest: Ubuntu Desktop 24.04.4 AMD64 • UEFI + KVM + SPICE + Clipboard + SSH port forwarding

Updated: May 2026

# 1. What you are building

You will use QEMU on an x86_64 Ubuntu host to install and boot an Ubuntu Desktop AMD64 virtual machine. The VM boots via UEFI firmware, uses virtio devices for disk and networking, uses KVM for hardware acceleration, exposes SSH to the host through a simple port forward, and uses SPICE for the graphical console.

This version uses SPICE instead of QEMU VNC. SPICE is better for a desktop VM because it supports smoother graphical interaction and, with `spice-vdagent` installed inside the guest, host-to-guest and guest-to-host clipboard sharing.

This is not ARM emulation. This is an AMD64 guest running on an x86_64 host, so QEMU can use KVM and run much faster than cross-architecture emulation.

The reliable default CPU setting is:

```bash
-cpu host
```

That exposes the real host CPU features to the guest. If the host is an AMD Zen 3 machine, the guest will naturally see AMD CPU features. If you only want the guest to look more like an AMD EPYC Milan CPU, there is an optional section later, but it should not be the default because some hosts and KVM setups reject forced EPYC models.

By the end, you will have:

* A VM disk image, `disk.qcow2`, containing Ubuntu Desktop AMD64.
* Two UEFI flash images: one for firmware code and one for persistent UEFI variables.
* A repeatable QEMU command line for installer boot over SPICE.
* A repeatable QEMU command line for normal boot over SPICE and SSH.
* Working copy/paste between the host and guest after `spice-vdagent` is installed.
* An optional headless command after SSH or serial console access is confirmed.
* Optional AMD Zen 3 style CPU branding.

## Directory layout we will use

Everything lives under your home directory. Example paths:

```text
~/qemu-ubuntu-amd64/                    # VM workspace
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
* A SPICE viewer on the host, provided by `virt-viewer`.

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

## 3.2 Install packages

Install the packages required to run QEMU with KVM acceleration, use OVMF UEFI firmware, create qcow2 disks, download the Ubuntu ISO, and connect to the VM over SPICE:

```bash
sudo apt install -y \
  qemu-system-x86 \
  qemu-utils \
  qemu-kvm \
  ovmf \
  wget \
  ca-certificates \
  virt-viewer \
  cpu-checker
```

What these are for:

* `qemu-system-x86`: provides `qemu-system-x86_64`, the x86_64 system emulator.
* `qemu-utils`: provides tools such as `qemu-img` for creating and inspecting VM disks.
* `qemu-kvm`: provides KVM integration for hardware-accelerated virtualization.
* `ovmf`: provides x86_64 UEFI firmware files.
* `wget` and `ca-certificates`: download the Ubuntu ISO over HTTPS.
* `virt-viewer`: provides `remote-viewer`, the SPICE viewer.
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

Check that SPICE-related options are present:

```bash
qemu-system-x86_64 -help | grep -n -- "-spice"
```

If either check does not print anything, reinstall the Ubuntu QEMU packages:

```bash
sudo apt install --reinstall -y qemu-system-x86 qemu-utils
```

# 5. Create the VM workspace

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
* `varstore.img`: the persistent UEFI variable store, also known as NVRAM. This remembers boot entries and other firmware settings.

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

# 7. Boot the installer with SPICE

Ubuntu Desktop uses a graphical installer. This guide exposes a local SPICE server and connects to it with `remote-viewer`.

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
  -device virtio-serial-pci \
  -chardev spicevmc,id=vdagent,name=vdagent \
  -device virtserialport,chardev=vdagent,name=com.redhat.spice.0 \
  -display none \
  -spice addr=127.0.0.1,port=5930,disable-ticketing=on
```

What to expect:

* The terminal may look idle. That is normal because the UI is in the SPICE viewer.
* SPICE is bound to localhost only, `127.0.0.1`. It is not exposed to the network.
* The SPICE server listens on TCP port `5930`.
* The VM also forwards host TCP port `8023` to guest TCP port `22` for SSH.
* Clipboard sharing will not fully work until `spice-vdagent` is installed inside the guest.

## 7.2 Connect to the installer over SPICE

Open another host terminal and run:

```bash
remote-viewer spice://127.0.0.1:5930
```

You should see the Ubuntu installer.

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

# 8. Boot the installed system with SPICE and SSH

## 8.1 Launch QEMU without the ISO

Use this for the first installed boot because it gives you a visible desktop console if SSH is not ready yet:

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
  -device virtio-serial-pci \
  -chardev spicevmc,id=vdagent,name=vdagent \
  -device virtserialport,chardev=vdagent,name=com.redhat.spice.0 \
  -display none \
  -spice addr=127.0.0.1,port=5930,disable-ticketing=on
```

Connect to the console:

```bash
remote-viewer spice://127.0.0.1:5930
```

## 8.2 Install the SPICE guest agent

Inside the Ubuntu guest, install and enable the SPICE agent:

```bash
sudo apt update
sudo apt install -y spice-vdagent
sudo systemctl enable --now spice-vdagent
```

Reboot the guest:

```bash
sudo reboot
```

After the guest reboots, start QEMU again with the same SPICE command and reconnect:

```bash
remote-viewer spice://127.0.0.1:5930
```

Host-to-guest and guest-to-host text copy/paste should now work in the SPICE viewer.

## 8.3 SSH into the guest

From another host terminal:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

Example:

```bash
ssh -p 8023 <your_user_name>@localhost
```

Example first connection output:

```text
The authenticity of host '[localhost]:8023 ([127.0.0.1]:8023)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[localhost]:8023' (ED25519) to the list of known hosts.
<your-guest-username>@localhost's password:
```

If SSH fails with `Connection reset by peer`, the QEMU host forward probably exists, but the SSH server inside the guest is missing or not running. Open the guest through SPICE and run:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

Then try again from the host:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

# 9. Copying text and files between host and guest

## 9.1 Desktop copy/paste through SPICE

For normal desktop text copy/paste, use SPICE with `spice-vdagent` installed in the guest.

Host requirements:

```bash
sudo apt install -y virt-viewer
```

Guest requirements:

```bash
sudo apt install -y spice-vdagent
sudo systemctl enable --now spice-vdagent
```

Then connect through:

```bash
remote-viewer spice://127.0.0.1:5930
```

Copy text on the host and paste it into the guest, or copy text in the guest and paste it back onto the host.

## 9.2 Terminal paste through SSH

SSH is often the easiest way to paste commands or text into the VM:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

To paste a text blob into a file inside the guest:

```bash
cat > notes.txt
```

Paste the text, then press `Ctrl+D`.

## 9.3 Copy files with scp

From the host to the guest:

```bash
scp -P 8023 /path/on/host/file.txt <your-guest-username>@localhost:/home/<your-guest-username>/
```

Example:

```bash
scp -P 8023 ~/Downloads/test.txt <your_user_name>@localhost:/home/<your_user_name>/
```

For a folder:

```bash
scp -P 8023 -r ~/my-folder <your_user_name>@localhost:/home/<your_user_name>/
```

From the guest back to the host:

```bash
scp -P 8023 <your_user_name>@localhost:/home/<your_user_name>/file.txt ~/Downloads/
```

Remember: SSH uses lowercase `-p`, while `scp` uses uppercase `-P` for the port.

## 9.4 Sync files with rsync

For repeated copying, `rsync` is nicer:

```bash
rsync -av -e "ssh -p 8023" ~/my-folder/ <your_user_name>@localhost:/home/<your_user_name>/my-folder/
```

# 10. Optional headless boot

Do not use headless mode as your first post-install boot unless you already know SSH works. Ubuntu Desktop is not guaranteed to show a useful serial login prompt until you configure one.

## 10.1 Minimal headless command

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

If the VM boots but the terminal shows no login prompt, that usually means the guest was not configured for a serial console. Boot with SPICE first, then configure serial output inside the guest:

```bash
sudo systemctl enable serial-getty@ttyS0.service
sudo sed -i 's/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"/' /etc/default/grub
sudo update-grub
sudo reboot
```

After that, retry the headless command.

# 11. Networking choices: user vs TAP/bridge vs passt

This guide uses user-mode networking because it requires no root network setup and works almost anywhere. However, you have options.

## 11.1 Option A: user-mode networking

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

## 11.2 Option B: TAP/bridge networking

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

## 11.3 Option C: passt backend, if present

Some QEMU builds support a `passt` backend. Check with:

```bash
qemu-system-x86_64 -help | grep -n "^-netdev passt"
```

If present, `passt` can be another host-friendly networking option. User-mode networking is still the easiest default.

# 12. CPU choices: reliable default vs AMD-style branding

## 12.1 Recommended default

Use this for the VM to launch reliably with KVM:

```bash
-cpu host
```

This exposes the real host CPU to the guest. On an AMD Zen 3 host, the guest will naturally report AMD CPU features.

## 12.2 Optional AMD Zen 3 style model

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

## 12.3 Important CPU note

With KVM, QEMU generally cannot invent a completely different CPU as freely as it can under full software emulation. If the real host CPU is Intel, forcing an AMD vendor string or an EPYC model can fail.

If artificial CPU branding matters more than performance, you can test TCG:

```bash
-accel tcg,thread=multi
```

That will be much slower than KVM. For normal use, keep KVM and use `-cpu host`.

# 13. QEMU flags explained line-by-line

This section explains the main QEMU flags used in the installer and normal-boot commands. For each flag we cover what it does, what virtual hardware it represents, and whether you can remove it safely.

| Flag / example                                                   | What it does                                                                        | Maps to virtual hardware?                                                 | Can you remove it?                                                                                                   |
| ---------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `-machine q35`                                                   | Selects a modern x86_64 machine model with PCIe support.                            | Yes: board/chipset model.                                                 | Keep it.                                                                                                             |
| `-accel kvm`                                                     | Uses hardware virtualization for speed.                                             | No direct guest device; affects how QEMU executes guest code on the host. | Keep it for performance on x86_64 hosts.                                                                             |
| `-cpu host`                                                      | Exposes the real host CPU model and CPU features to the guest.                      | Yes: guest CPU model and features.                                        | Keep it as the reliable default.                                                                                     |
| `-smp 8`                                                         | Exposes 8 virtual CPUs to the guest.                                                | Yes: vCPU count and topology.                                             | Optional. Adjust to match your host.                                                                                 |
| `-m 8192`                                                        | Allocates 8192 MiB of RAM to the guest.                                             | Yes: guest RAM size.                                                      | Optional. Adjust to match your host.                                                                                 |
| `-drive if=pflash,format=raw,file=efi.img,readonly=on`           | Attaches UEFI firmware code and makes it read-only.                                 | Yes: flash device holding UEFI firmware.                                  | Keep if you want UEFI boot.                                                                                          |
| `-drive if=pflash,format=raw,file=varstore.img`                  | Attaches the persistent UEFI variable store.                                        | Yes: flash region for UEFI variables.                                     | Strongly recommended. Removing it causes non-persistent firmware state.                                              |
| `-drive if=virtio,format=qcow2,file=disk.qcow2`                  | Attaches the main VM disk using virtio.                                             | Yes: virtio block disk.                                                   | Keep for the installed OS.                                                                                           |
| `-device virtio-net-pci,netdev=net0`                             | Adds a virtio network card connected to backend `net0`.                             | Yes: virtio network interface.                                            | Optional only if you do not need networking.                                                                         |
| `-netdev user,id=net0,hostfwd=tcp::8023-:22`                     | Creates a user-mode NAT network backend and forwards host TCP 8023 to guest TCP 22. | Host-side network backend, not a guest device.                            | Optional if you switch to TAP, bridge, or passt. Keep `hostfwd` for easy SSH.                                        |
| `-device virtio-scsi-pci,id=scsi0`                               | Adds a virtio SCSI controller so the ISO can be attached as a SCSI CD-ROM.          | Yes: virtio SCSI controller.                                              | Installer-only in this guide.                                                                                        |
| `-drive if=none,id=cdrom,format=raw,readonly=on,file="$ISO"`     | Defines the ISO as a backend named `cdrom`.                                         | Host-side block backend definition.                                       | Installer-only. Remove after installation.                                                                           |
| `-device scsi-cd,drive=cdrom`                                    | Attaches the ISO backend as a SCSI CD-ROM device.                                   | Yes: CD-ROM device.                                                       | Installer-only. Remove after installation.                                                                           |
| `-boot order=d`                                                  | Requests CD-ROM boot first.                                                         | Firmware boot behavior.                                                   | Installer-only. Remove after installation.                                                                           |
| `-device virtio-vga`                                             | Provides a display adapter for the SPICE console.                                   | Yes: virtual display adapter.                                             | Keep for SPICE graphical console. Remove for pure SSH/headless use.                                                  |
| `-device qemu-xhci`                                              | Adds a USB 3 controller.                                                            | Yes: USB controller.                                                      | Useful for graphical input. Usually safe to remove for SSH-only use.                                                 |
| `-device usb-kbd`                                                | Adds a USB keyboard.                                                                | Yes: USB HID keyboard.                                                    | Useful for graphical input. Usually safe to remove for SSH-only use.                                                 |
| `-device usb-tablet`                                             | Adds an absolute pointing device for better mouse behavior.                         | Yes: USB HID tablet.                                                      | Useful for graphical input. Usually safe to remove for SSH-only use.                                                 |
| `-device virtio-serial-pci`                                      | Adds a virtio serial controller used by the SPICE guest agent channel.              | Yes: virtio serial controller.                                            | Keep if you want SPICE clipboard integration.                                                                        |
| `-chardev spicevmc,id=vdagent,name=vdagent`                      | Creates a SPICE virtual channel for the guest agent.                                | Host-side SPICE channel.                                                  | Keep if you want SPICE clipboard integration.                                                                        |
| `-device virtserialport,chardev=vdagent,name=com.redhat.spice.0` | Exposes the SPICE agent channel to the guest.                                       | Yes: virtio serial port device.                                           | Keep if you want SPICE clipboard integration.                                                                        |
| `-display none`                                                  | Prevents QEMU from opening a local GUI window.                                      | Host UI behavior.                                                         | Keep when using SPICE as the display viewer.                                                                         |
| `-spice addr=127.0.0.1,port=5930,disable-ticketing=on`           | Starts a local SPICE server on port 5930 with no ticket password.                   | Host UI behavior.                                                         | Keep for SPICE. Consider adding authentication if exposing beyond localhost, but this guide binds only to localhost. |
| `-nographic`                                                     | Disables graphics and routes serial console to your terminal.                       | Host console wiring.                                                      | Optional. Good after serial console is configured.                                                                   |

# 14. Minimal recommended post-install commands

## 14.1 Desktop/SPICE command

Use this when you want the Ubuntu graphical desktop, clipboard sharing, and SSH forwarding:

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
  -device virtio-serial-pci \
  -chardev spicevmc,id=vdagent,name=vdagent \
  -device virtserialport,chardev=vdagent,name=com.redhat.spice.0 \
  -display none \
  -spice addr=127.0.0.1,port=5930,disable-ticketing=on
```

Connect with:

```bash
remote-viewer spice://127.0.0.1:5930
```

## 14.2 SSH/headless command

Once SSH works and you no longer need the desktop console, you can simplify to the essentials:

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

This removes graphical devices, SPICE, and clipboard integration, while keeping a stable UEFI + disk + network + SSH workflow. Use the SPICE boot command instead if you still need the desktop console or graphical clipboard sharing.

# 15. Troubleshooting quick hits

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

## Error: SPICE option is not recognized

Check whether your QEMU build supports SPICE:

```bash
qemu-system-x86_64 -help | grep -n -- "-spice"
```

If nothing prints, reinstall QEMU from Ubuntu packages:

```bash
sudo apt install --reinstall -y qemu-system-x86
```

## `remote-viewer` command not found

Install `virt-viewer` on the host:

```bash
sudo apt install -y virt-viewer
```

Then reconnect:

```bash
remote-viewer spice://127.0.0.1:5930
```

## Clipboard does not work

Make sure the QEMU command includes all three SPICE agent channel lines:

```bash
-device virtio-serial-pci \
-chardev spicevmc,id=vdagent,name=vdagent \
-device virtserialport,chardev=vdagent,name=com.redhat.spice.0
```

Then inside the guest, install and start the agent:

```bash
sudo apt update
sudo apt install -y spice-vdagent
sudo systemctl enable --now spice-vdagent
sudo reboot
```

After reboot, start QEMU again and reconnect with `remote-viewer`.

## Boot returns to installer

You are still booting with the ISO attached or using the installer command. Boot without these installer-only lines:

```text
-drive if=none,id=cdrom,format=raw,readonly=on,file="$ISO"
-device scsi-cd,drive=cdrom
-boot order=d
```

## SSH gives `Connection reset by peer`

This usually means the host-side QEMU port forward exists, but the guest is closing port `22` because SSH is missing or not running.

Open the guest through SPICE and run:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

Then from the host, try again:

```bash
ssh -p 8023 <your-guest-username>@localhost
```

## SSH does not respond on port 8023

First confirm QEMU is listening on the host:

```bash
ss -ltnp | grep ':8023'
```

If you see `qemu-system-x86_64`, the host forward exists.

Then boot with SPICE, log into the guest, and check SSH:

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

## SPICE port 5930 is already in use

Pick another SPICE port:

```bash
-spice addr=127.0.0.1,port=5931,disable-ticketing=on
```

Then connect with:

```bash
remote-viewer spice://127.0.0.1:5931
```

## Headless boot shows a blank terminal

This usually means the guest is not configured to use the serial console. Boot with SPICE, then configure serial output inside the guest:

```bash
sudo systemctl enable serial-getty@ttyS0.service
sudo sed -i 's/^GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"/' /etc/default/grub
sudo update-grub
sudo reboot
```

Then retry the headless command.

# 16. Confirm system architecture and CPU model

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

# 17. Optional custom QEMU build

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
  libslirp-dev \
  libspice-server-dev
```

Clone QEMU:

```bash
cd ~
git clone https://gitlab.com/qemu-project/qemu.git
cd qemu
```

Configure only the x86_64 system emulator with KVM, SLIRP/user networking, and SPICE support:

```bash
rm -rf build
./configure --target-list=x86_64-softmmu --enable-slirp --enable-kvm --enable-spice
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

Verify SPICE is available:

```bash
~/qemu/build/qemu-system-x86_64 -help | grep -n -- "-spice"
```

Use the custom binary by replacing `qemu-system-x86_64` with:

```bash
~/qemu/build/qemu-system-x86_64
```

Important: if you configure QEMU only for `aarch64-softmmu`, you build `qemu-system-aarch64`, not `qemu-system-x86_64`. For this guide, you need `x86_64-softmmu`.

# 18. Recommended default path

Use this order when setting up the VM:

1. Install host packages.
2. Confirm KVM works.
3. Create the VM workspace.
4. Create the qcow2 disk.
5. Copy the OVMF UEFI images to `efi.img` and `varstore.img`.
6. Download the Ubuntu Desktop AMD64 ISO.
7. Boot the installer command with SPICE.
8. Connect with `remote-viewer`.
9. Install Ubuntu and enable OpenSSH if available.
10. Stop QEMU after the installer reboots.
11. Boot without the ISO using the SPICE boot command.
12. Install `spice-vdagent` inside the guest.
13. Reboot the guest.
14. Reconnect through SPICE and confirm clipboard copy/paste works.
15. Install and enable OpenSSH inside the guest if SSH does not work.
16. SSH into the guest from the host using port `8023`.
17. Only use the headless command after SSH or serial console output is confirmed.
18. Keep `-cpu host` as the default. Only test `EPYC-Milan` after the VM already works.

# References

Ubuntu Desktop AMD64 ISO directory: [https://releases.ubuntu.com/noble/](https://releases.ubuntu.com/noble/)

Ubuntu Desktop AMD64 ISO used here: [https://releases.ubuntu.com/noble/ubuntu-24.04.4-desktop-amd64.iso](https://releases.ubuntu.com/noble/ubuntu-24.04.4-desktop-amd64.iso)

QEMU invocation documentation: [https://www.qemu.org/docs/master/system/invocation.html](https://www.qemu.org/docs/master/system/invocation.html)

QEMU networking documentation: [https://www.qemu.org/docs/master/system/net.html](https://www.qemu.org/docs/master/system/net.html)

SPICE project: [https://www.spice-space.org/](https://www.spice-space.org/)
