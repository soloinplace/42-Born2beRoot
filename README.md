*This activity has been created as part of the 42 curriculum by csodini.*

# Born2beRoot

> My first Linux server, built from the ground up in a virtual machine.

## Description

**Born2beRoot** is a system administration project introducing the fundamentals of virtualization, Linux server installation, storage management, access control, networking, security policies, and automated system monitoring.

The project consists of installing and configuring a minimal Linux server inside a virtual machine while following a strict security specification.

This implementation uses:

* **Debian GNU/Linux 13 (Trixie)**
* **Oracle VirtualBox**
* **LUKS / dm-crypt** for disk encryption
* **LVM** for logical volume management
* **AppArmor** for mandatory access control
* **OpenSSH** for remote administration
* **UFW** for firewall management
* **sudo** with restricted and logged administrative access
* **cron** for periodic monitoring

No graphical server or desktop environment is installed.

---

## Project Description - System Architecture

The virtual disk has a total size of **20 GB** and is organized as follows:

```text
/dev/sda                                  20G
│
├── /dev/sda1                           953M
│   └── ext4
│       └── /boot
│
└── /dev/sda2                          19.1G
    └── LUKS / dm-crypt
        └── /dev/mapper/sda2_crypt     19.1G
            └── LVM Physical Volume
                └── Volume Group: vg0
                    │
                    ├── LV root        11.2G
                    │   └── ext4
                    │       └── /
                    │
                    ├── LV home         4.7G
                    │   └── ext4
                    │       └── /home
                    │
                    └── LV swap         1.9G
                        └── swap area
```

The corresponding `lsblk` layout is:

```text
NAME           SIZE TYPE  MOUNTPOINTS
sda             20G disk
├─sda1         953M part  /boot
└─sda2        19.1G part
  └─sda2_crypt 19.1G crypt
    ├─vg0-root 11.2G lvm   /
    ├─vg0-home  4.7G lvm   /home
    └─vg0-swap  1.9G lvm   [SWAP]
```

The root, home, and swap logical volumes are managed by LVM and are located inside the encrypted LUKS container.

### Boot process

The VM uses a traditional BIOS/GRUB boot chain:

```text
VirtualBox BIOS
       ↓
      GRUB
       ↓
Linux kernel + initramfs
       ↓
Unlock LUKS encrypted volume
       ↓
Activate LVM
       ↓
Mount the root filesystem
       ↓
Debian userspace
```

The `/boot` filesystem remains outside the encrypted LUKS container and contains the files required early in the boot process, including the Linux kernel and initramfs.

---

## Why Debian?

I chose **Debian** because Born2beRoot is my first system administration project and Debian provides a minimal, stable, and well-documented environment without hiding the underlying Linux mechanisms.

Debian also allowed me to work directly with tools such as APT, AppArmor, UFW, PAM, systemd, OpenSSH, LUKS, and LVM.

### Debian vs Rocky Linux

| Debian                                             | Rocky Linux                                         |
| -------------------------------------------------- | --------------------------------------------------- |
| Community-driven distribution                      | Enterprise-oriented RHEL-compatible distribution    |
| Uses `.deb` packages                               | Uses `.rpm` packages                                |
| APT package management                             | DNF package management                              |
| AppArmor commonly used                             | SELinux commonly used                               |
| UFW can provide a simple firewall interface        | firewalld is commonly used                          |
| Large community and documentation base             | Strong compatibility with the RHEL ecosystem        |
| Particularly approachable for a first Linux server | Useful preparation for enterprise/RHEL environments |

**Debian advantages:** simplicity, stability, extensive documentation, and a relatively low barrier to entry.

**Debian disadvantages:** some packages prioritize stability over providing the newest upstream release.

**Rocky advantages:** close compatibility with Red Hat Enterprise Linux and strong relevance in enterprise infrastructure.

**Rocky disadvantages:** SELinux, firewalld, and the wider RHEL administration model can make an initial setup more complex for a beginner.

---

## Security Configuration

### Users and groups

The machine hostname is:

```text
csodini42
```

The main user is:

```text
csodini
```

It belongs to the required administrative groups:

```text
sudo
user42
```

Administrative work is performed through `sudo` instead of remaining permanently logged in as `root`.

---

### Password policy

Password aging is configured with:

```text
Maximum password age:      30 days
Minimum password age:       2 days
Expiration warning:         7 days
```

Password complexity is enforced through PAM and `pam_pwquality`.

The policy requires:

```text
Minimum length:             10 characters
Uppercase character:        required
Lowercase character:        required
Digit:                      required
Maximum repeated character: 3
Username in password:       forbidden
Difference from old passwd: 7 characters
```

The quality policy is also enforced for the root account where applicable.

Password-aging information is stored in the system account database and configured for existing users with `chage`, while `/etc/login.defs` provides the default aging policy for newly created accounts.

---

### sudo

`sudo` is configured through a dedicated file under:

```text
/etc/sudoers.d/
```

The configuration includes:

* a maximum of **3 authentication attempts**;
* a custom error message after an incorrect password;
* command logging;
* input and output logging under `/var/log/sudo/`;
* mandatory TTY usage;
* a restricted `secure_path`.

The configuration was edited and validated with `visudo`.

---

### SSH

OpenSSH is used for remote administration.

The SSH daemon listens on:

```text
TCP port 4242
```

Direct remote login as `root` is disabled:

```text
PermitRootLogin no
```

During development, VirtualBox NAT port forwarding was configured so that host port `4242` is redirected to guest port `4242`.

The VM can therefore be reached from the host with:

```bash
ssh -p 4242 csodini@127.0.0.1
```

Here, `127.0.0.1` is the IPv4 loopback address of the host machine, and VirtualBox forwards the connection to the SSH service inside the VM.

---

### Firewall

The server uses **UFW — Uncomplicated Firewall**.

The default policy is:

```text
Incoming connections: DENY
Outgoing connections: ALLOW
```

The only explicitly allowed incoming service is:

```text
4242/tcp
```

for SSH.

### UFW vs firewalld

Both tools provide higher-level interfaces for configuring Linux firewall rules.

**UFW** is intentionally simple and rule-oriented. It is well suited to a small server where only a limited number of ports need to be exposed.

**firewalld** provides a more dynamic model based around concepts such as zones and services. This can be useful on systems with multiple interfaces or changing network trust levels.

For this server, UFW is appropriate because the required policy is deliberately minimal: deny incoming traffic by default and allow SSH on TCP port 4242.

---

## Mandatory Access Control

### AppArmor

AppArmor is enabled and active at startup on this Debian system.

Traditional Unix permissions determine whether a **user** may access a resource. AppArmor adds another security layer by restricting what an individual **application** is allowed to access, even when traditional permissions would otherwise permit the operation.

AppArmor policies are primarily based on filesystem paths.

No custom AppArmor profile was required for this project; the requirement for the Debian implementation is that AppArmor is active at startup.

### AppArmor vs SELinux

Both are **Mandatory Access Control (MAC)** systems designed to restrict applications beyond traditional Unix user/group permissions.

| AppArmor                                            | SELinux                              |
| --------------------------------------------------- | ------------------------------------ |
| Primarily path-based rules                          | Primarily label/type-based rules     |
| Generally easier to read initially                  | More granular policy model           |
| Commonly associated with Debian/Ubuntu environments | Standard in the RHEL/Rocky ecosystem |
| Lower initial learning curve                        | Steeper initial learning curve       |

AppArmor was used because it is the mandatory access-control system required for the Debian implementation of this project.

---

## Virtualization

### VirtualBox

VirtualBox acts as the **hypervisor** for this project.

It exposes virtual hardware to the guest system, including:

* virtual CPUs;
* virtual RAM;
* a virtual network interface;
* a virtual optical drive;
* a virtual hard disk.

Debian therefore operates as if it were installed on a physical computer, while its hardware is provided by VirtualBox.

The virtual disk uses the VDI format.

Networking uses VirtualBox NAT with a port-forwarding rule allowing the host to reach the guest SSH service.

### VirtualBox vs UTM

**VirtualBox** is a mature general-purpose virtualization platform and integrates well with x86 systems. It was the most appropriate choice for this project because the VM was created and evaluated in an x86 VirtualBox environment.

**UTM** is particularly useful on Apple Silicon systems and can use Apple's virtualization framework as well as QEMU-based emulation.

The main distinction relevant to this project is portability: a VM built for one CPU architecture or hypervisor is not necessarily directly interchangeable with a VM built for another.

---

## Monitoring

A Bash script located at:

```text
/usr/local/bin/monitoring.sh
```

collects and displays information about the current machine state.

It reports:

* operating-system architecture and kernel version;
* number of physical processors;
* number of virtual processors;
* memory usage;
* disk usage;
* CPU utilization;
* last boot time;
* LVM status;
* established TCP connections;
* logged-in users;
* IPv4 address;
* MAC address;
* number of commands executed through `sudo`.

The script combines standard Unix tools such as:

```text
uname
lscpu
free
df
vmstat
uptime
lsblk
ss
who
ip
grep
awk
cut
sort
wc
```

The output is written directly to available local TTY and PTY terminal devices.

Local Linux virtual consoles use devices such as:

```text
/dev/tty1
/dev/tty2
...
```

while SSH sessions use pseudo-terminals such as:

```text
/dev/pts/0
/dev/pts/1
...
```

The Bash script uses pathname expansion to find the currently available terminal devices and writes the monitoring message to writable ones.

This approach was used because the installed `wall` implementation did not correctly broadcast to the SSH pseudo-terminal in this environment, even though direct writes to the PTY worked correctly.

---

## Scheduling

The monitoring script is executed by the root crontab:

```cron
@reboot /usr/local/bin/monitoring.sh
*/10 * * * * /usr/local/bin/monitoring.sh
```

The first rule executes the script when the `cron` service starts after boot.

The second rule executes it every ten minutes.

The five cron time fields represent:

```text
minute hour day-of-month month day-of-week
```

Therefore:

```cron
*/10 * * * *
```

means every ten minutes, regardless of hour, day, month, or weekday.

The cron service can be stopped without modifying the monitoring script:

```bash
sudo systemctl stop cron
```

and restarted with:

```bash
sudo systemctl start cron
```

---

## Instructions

### Starting the server

Start the virtual machine from VirtualBox.

During the boot sequence, enter the LUKS passphrase when prompted so that the encrypted storage can be unlocked.

After Debian has booted, log in with a non-root user.

---

### Connecting through SSH

With the VirtualBox NAT port-forwarding rule mapping host port `4242` to guest port `4242`, connect from the host with:

```bash
ssh -p 4242 csodini@127.0.0.1
```

---

### Useful verification commands

#### System identity

```bash
hostname
cat /etc/os-release
uname -a
```

#### Storage

```bash
lsblk
lsblk -f
findmnt
swapon --show
```

#### Users and groups

```bash
id csodini
groups
sudo -l
```

#### Services

```bash
systemctl status ssh
systemctl status apparmor
systemctl status cron
```

#### Firewall

```bash
sudo ufw status verbose
```

#### SSH configuration

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin)'
```

Expected important values:

```text
port 4242
permitrootlogin no
```

#### Scheduled monitoring

```bash
sudo crontab -l
```

#### Password aging

```bash
sudo chage -l csodini
sudo chage -l root
```

#### sudo configuration

```bash
sudo visudo -c
```

---

## Main Concepts Learned

This project provided practical experience with the complete path from a blank virtual disk to an administrable Linux server:

```text
virtualization
    ↓
firmware and boot process
    ↓
GRUB
    ↓
Linux kernel and initramfs
    ↓
partitioning
    ↓
disk encryption with LUKS/dm-crypt
    ↓
LVM
    ↓
filesystems and mount points
    ↓
users, groups and permissions
    ↓
PAM and password policies
    ↓
sudo
    ↓
SSH
    ↓
TCP/IP networking
    ↓
NAT and port forwarding
    ↓
firewall
    ↓
AppArmor
    ↓
systemd services
    ↓
shell scripting
    ↓
cron automation
```

A major goal of the project was not only to configure these components, but also to understand how the different layers interact.

---

## Resources

The following resources were useful for understanding and verifying the concepts used in this project:

* Debian documentation
* Debian manpages
* VirtualBox documentation
* OpenSSH documentation
* systemd documentation
* Linux manual pages
* Born2beRoot subject
* 42 peer-learning discussions

Useful manual pages included:

```text
man sshd_config
man sudoers
man ufw
man crontab
man pam_pwquality
man login.defs
man chage
man lvm
man cryptsetup
man systemctl
man lsblk
man ss
```

---

## Use of AI

AI was used as a **learning, troubleshooting, and verification tool** throughout the project.

Its main purposes were to:

* explain Linux, networking, security, and Bash concepts;
* clarify commands, options, pipelines, regular expressions, `awk`, and shell syntax;
* help understand the boot, storage, authentication, and virtualization layers;
* review and troubleshoot the configuration of PAM, sudo, SSH, UFW, LVM, and system monitoring;
* investigate unexpected behavior, such as the SSH pseudo-terminal issue with `wall`;
* help structure and review this README.

All configuration changes were performed manually and verified directly inside the virtual machine. AI suggestions were tested and questioned rather than used as a substitute for understanding the system.

---

## Repository Contents

Only the files required for submission are stored in the Git repository:

```text
.
├── README.md
└── signature.txt
```

The virtual machine itself is **not** committed to Git.

`signature.txt` contains the SHA-1 signature of the final virtual disk used for evaluation.
