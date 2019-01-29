# Saturday 26 January 2019

----

## System Design Primer

----

### Databases

#### Federation vs Sharding

Federation:

* different databases for different purposes. e.g. a database for users, a database for forums, a database for products, etc.
* advantages:
  * spreads out reads and writes
  * smaller dbs means more data can fit in memory, which means more cache hits
* disadvantages:
  * more complex app logic
  * joins are more complex over the network
  * more hardware, more complexity

Sharding

* split a databsae **table** across multiple nodes. e.g. if the `users` table is huge, split it into different databases according to last name
* advantages:
  * spreads out reads and writes
  * more cache hits, reduced index size
  * No master-slave coordination so writes can happen in parallel
  * if one goes down, you don't lose ALL data (although you should still have backups)
* disadvantages:
  * more complex app logic
  * need to make sure the shards aren't lopsided, like having power users all stored on one shard
  * joins are more complex
  * more hardware, more problems

#### Denormalization

A lot of systems are read heavy, so reads that require complex joins, especially across different nodes, are costly. Denormalization is basically  making redundant copies of data written in multiple tables so you can read data without doing an expensive join.

* disadvantages of denormalization:
  * duplication of data uses more disk space (no big deal)
  * redundant copies of data need to stay in sync, which makes database design more difficult
  * heavy write systems might get no benefit

----

### Asyncronism (Message Queues)

Sometimes a client doesn't want to wait for a server's response before moving on to do other things. So just pop the job into a message queue and let the server read from the queue when it's ready.

Kafka is a message queue, but it can also be thought of as a distributed event log that keeps different parts of a microservice architecture in sync (eventually). The event paradigm also allows us to "go back in time" and replay events.

----

### Map Reduce

Map --> Shuffle --> Reduce

Main idea is to take a big input, break it up into small pieces, find local solutions, then combine local solutions to find global solutions.

For example, let's say you want to find the number of occurences of each word in a document. This would be a pretty simple function in Python:

```python
from collections import defaultdict
def word_counts(document):
    words = document.split()
    count_dict = defaultdict(int) # { word: count}
    for word in words:
        count_dict[word] += 1
    return count_dict
```

The problem occurs when the document is 3 TB and the dictionary won't fit in memory.

Map phase:

* break the document into smaller pieces that will fit in memory and distribute those pieces among several machines
* run the function on each machine separately to get many local solutions. In this case, we might want a different output file for each individual word rather than a dictionary.

Shuffle phase:

* Move all the local solutions for a given word onto the same machine. For example, every local solution "cat", so `{"cat":5}` on machine 1, `{"cat":7}` on manchine 2 and so on, should all be moved to the same machine, say machine A (which could be a machine we've already used).

Reduce phase:

* Now that all the local solutions are on the same machine, they can be loaded into local memory and combine into one solution, e.g. `{"cat": 1482693}`

----
----

## Linux Academy

----

### Video Player

The first hurdle for Linux Academy! Check out [this support page](https://support.linuxacademy.com/hc/en-us/articles/218203783-Video-Playback-on-Linux). You have to enable DRM content in firefox, [install ffmpeg-lib](https://tecadmin.net/install-ffmpeg-on-linux/), and possibly a third step ( I didn't need to do the third step).

----

### System Architecture

#### Psuedo File Systems

* a pseudo file system exists in memory (ephemeral)
  * `/proc` and `/sys` are two important pseudo file systems
* `/sys` has hardware and kernel info (system info)
* `/proc` is concerned with processes
* **Everything on Linux is seen as a file**

#### Kernel Modules

* core framework of the OS
* allows rest of system to operate with hardware, memory, networking, and itself
* can dynamically load and unload drivers without restarting
* See `lsmod`, `modprobe`, `modinfo` commands for information about interacting with kernel modules
* Note that `lsmod` only shows info about kernel modules that are **currently loaded into memory**

#### Hardware

* the `udev` service detects new hardware like a new hard disk, sends it through the `D-Bus` service to the `/dev` device pseudo file system. The `lsblk` command goes through D-Bus to get information from `/dev` to display
* `lspci`, `lsusb`, `lscpu`, `lsblk` commands all interact with `/dev` to get readable info about devices (PCI, USB, CPU, and block devices like hard drives)

#### Booting

1. BIOS on motherboard checks all input/output devices
2. Boot program GRUB (Grand Unified Bootloader) looks for the section of the hard drive needed to boot
3. Bootloader loads linux kernel
4. Kernel loads "initial RAM disk" which contains device drivers and gets ready to mount filesystems
5. Kernel starts initialization system that mounts the computer's filesystems and daemons. RAM disk is removed.

* use `dmesg` to view boot logs from "kernel ring buffer" in RAM. Use this to look for devices that kernel sees but don't show up in `/dev`
* on most modern systems, the init system is `systemd`, and on it, `journalctl` which logs every event that happens on the computer. To see kernel messages specifically, use `journalctl -k`
* `init` starts services serially, one after the other. `systemd` parallelized intialization much better
* kernal looks for `/sbin/init` program on boot. Init looks for configuration settings in `/etc/inittab`
* runlevels start and stop services. runlevels apply to entire system
  * runlevel 0: halt
  * runlevel 1: single user mode
  * runlevel 2: multiple users (no network, no remote file systems)
  * runlevel 3: multiple users (with networking). **Most common for servers**
  * runlevel 4: unused. Can be made custom
  * runlevel 5: same as 3, but with a graphical desktop
  * runlevel 6: reboot
* `/etc/inittab`:
  * \<id>:\<runlevel>:\<action>:\<process>
* init scripts usually located at `/etc/init.d`. They look like `rc0.d`, `rc1.d`, ..., for "run command" scripts. The number is the runlevel
* `rc.local` can be modified by a system administrator to start extra services or tasks that don't have init scripts

Upstart: init replacement for Ubuntu

* decrease bootup time to start services in parallel
* `startu`p event runs `mountall` and `/etc/init/rc-sysinit.conf` simultaneously
* `telinit` executes `/etc/init/rc.conf`, and lot of scripts then run in parallel
* init doesn't update when you, say, plug in a new monitor. Upstart, on the other hand, monitors services for events
* upstart attempts to respawn failed jobs up to 10 times at 5 second intervals
* **big idea:** upstart was the first dynamic, parallelized, event-driven init framework

**systemd**: The modern init

* innovation: not run with BASH scripts because they are inefficient (need to access library files)
* replaced shell scripts with `C` code for efficiency
* still compatible with older scripts
* usual location of systemd files: `/usr/lib/systemd`. Do not edit these files because they could be updated with package updates
* location for sys admins to edit unit files is `/etc/systemd/system`, which take precendent over `/usr`
* `systemctl list-unit-files` or `systemctl cat <something>.unit` to view unit files
* boot:
  * kernel looks for `sbin/init` like usual
  * developers created a symbolic link from there to the systemd binary

* BIOS > master boot loader > boot loader > kernel > device initialization with RAM disk > root file system mount

#### Change Runlevels, Boot Targets, Shutdown, Reboot

runlevels (before systemd)

* `runlevel` shows what runlevel you were on and which runlevel you are on now
* `sudo telinit <runlevel>`
* `cat /etc/inittab` will have your system's default runlevel info
* press any key during bootup to open GRUB, then type `a` to give different kernel arguments, including runlevel

change environment with systemd:

* most common use of systemd Target is to bring the system to a new state, i.e. graphical, multi-user,etc. (like runlevels)
* common tagets:
  * multi-user.target (like runlevel 3)
  * graphical.target (like runlevel 5)
  * rescue.target for a rescue shell where admin can debug (like runlevel 1)
  * basic.target (just a basic environment)
  * sysinit.target (system initialization)
* view with `systemctl cat <something>.unit`
* `systemctl list-unit-files -t target` to view all target files
* view default target environment with `systemctl get-default`
* change default target to multi-user with `systemctl set-default multi-user.target`
* change current target environment to multi-user with `systemctl isolate multi-user.target`

Reboot and Shutdown:

* reboot commands:
  * `reboot`
  * `telinit 6` (set runlevel to 6)
  * `shutdown -r now`
  * `systemctl isolate reboot.target`
* shutdown commands:
  * `poweroff`
  * `telinit 0` (set to runlevel 0)
  * `shutdown -h 1 minute`
  * `systemctl isolate poweroff.target`
* `acpid` Advanced Configuration and Power interface--registers  system events such as pressing power button or closing laptop lid. See `/etc/acpid` events and actions
* `wall` broadcasts a message to all logged in users

#### Linux Installation and Package Mangement

Main Filesystem Locations:

* `/` root directory
* `/var` variable directory
  * **log files**, dynamic content like websites
* `/home` user home directory
* `/boot` on a different partition where linux kernel and supporting files are stored
* `/opt` optional software, like third party software
* swap space
  * swap is temporary storage that acts like RAM
  * when RAM is full, kernel moves less used data to swap
  * located on swap partition (faster) or swap file (slower)
  * swap should be no less than 50% of available RAM
* partitions:
  * `/dev/sda` first hard drive ("a")
  * `/dev/sda1` the first partition of the first hard drive
* mount points
  * mounts filesystems onto partitions, e.g. `/boot` mounted on `/dev/sda3` and `/` root on `/dev/sda1`
* commands:
  * `mount` lists all partitions and mount points on the system (mostly pseudo filesystems)
  * `sudo fdisk -l /dev/sdb` will list (`-l`) info about the `sdb` disk and its partitions
  * `swapon --summary` will show information about swap space. On my machine, there is a swapfile located at `/swapfile`

Intro to LVM (Logical Volume Manager)

* allows "groups" of disks or partitions to be assembled into a single filesystem
* GRUB can't read LVM metadata, so LVM can be used for any mount point **except** `/boot`
* can resize volumes
* can create snapshots
* layout:
  * **physical volumes**, like `/dev/sda`, `/dev/sdb`
  * **volume groups** can group together physical volumes
  * **logical volumes** can partition a volume group (data on a logical volume can exist on multiple physical volumes)
  * **file systems** like `/`, `/var`
* commands:
  * `pvs` physical volume scan
  * `vgs` volume group scan
  * `lvs` logical volume scan

----
----

## Miscellaneous

Long talked about:

* CPU is always executing code or handling an interrupt
* Unix was monolithic in that there is one memory address space, whereas Linux kernel is monolithic + modular so memory addresses can be released and reused
* Windows 7 and vista were "micro kernels", with segmented memory address space and then translating between micro kernels when more memory is needed. This didn't work out. Linux is best of both worlds