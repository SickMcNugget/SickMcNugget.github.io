---
layout: post
title:  "Trying sched_ext"
date:   2025-04-15 15:27:46 +08:00
categories: tutorial linux kernel bpf sysadmin
published: true
---
# What's sched_ext?
With the advent of BPF making its way into the Linux kernel, new opportunities have presented themselves to developers and kernel maintainers alike. One new technology is *sched_ext*, which allows a custom CPU scheduler to be built on top of the BPF subsystem. This allows more effective scheduling to be achieved for specific programs when the developer knows what the requirements of their software are.

# Preface
I started with a Rocky 9.5 VM to do everything I talk about in this article. First, I grabbed the *minimal* iso from the best available mirror, https://rockylinux.mirror.digitalpacific.com.au/9/isos/x86_64/. After that, I created the VM from the ISO, and updated to the latest kernel.

```bash
dnf -y update
systemctl reboot
```
Upon reboot, I downloaded the latest stable Linux kernel source so that I could build it myself. Unfortunately, it seems that the Elrepo mainline kernel doesn't have SCHED_EXT enabled in the kernel configuration, so we'll have to do that ourselves. At the time of writing, v6.14.2 was stable. Just use whatever is stable for you.

```bash
# pahole (dwarves) is essential for BTF. Without it, the SCHED_CLASS_EXT option isn't even available in KConfig menu.
dnf -y install gcc make ncurses-devel openssl-devel flex bison dwarves elfutils-libelf-devel 
curl -O https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.14.2.tar.xz
```

We should have the build prerequisites available, but take that with a grain of salt.
```bash
tar -xf linux-6.14.2.tar.xz
cd linux-6.14.2
cp /boot/config-$(uname -r)* .config
```
Now that we have our current kernel config inside the linux source, we can modify it to suit our needs. Since I'm fairly new to this process, I left far too many drivers enabled, which drastically increased the compilation time and the amount of space taken up by the kernel after/during build. If you are more experienced, disable some device drivers that you know you wont need.

Let's enable the options we need to ensure the build works
We need to set the following:
- **General Setup->Extensible Scheduling Class->Enabled**
- **Cryptographic API->Certificates for signature checking->Additional X.509 keys for default system keyring->Change to ""**
- (Optional) **Device Drivers->Disable Any you don't need**
```bash
make nconfig
```

Now we're ready to build. Lookout for any errors and install packages as necessary
```bash
make -j$(nproc)
```
After we've built the kernel, grubby provides us an `/sbin/installkernel` script that can automatically install a newly built kernel for us, yay!
```bash
# Below uses the installkernel script
make modules_install install
```

Now we have the new kernel, so reboot the VM
```bash
systemctl reboot
```

Once we're back in the VM, lets clean some space to make our lives easier for next time.
```bash
# Make sure this says the expected kernel version
uname -r
# Mine outputs: "6.14.2"

cd linux-6.14.2
make clean
```

Now shutoff the VM so we can make a snapshot of the qcow2.
```bash
systemctl poweroff
```

On the host, let's make a minimally sized .qcow2 that we can reuse later if something goes wrong now. I'm going to assume your image is called rocky9.qcow2.
Again, please ensure that the VM is powered down when you do this.
```bash
sudo su -
cd /var/lib/libvirt/images/

# You need guestfs-tools
dnf -y install guestfs-tools
# OR
apt install -y guestfs-tools

# This just zeroes out unused bytes
virt-sparsify --in-place rocky9.qcow2
# Now let's create a compressed disk
# I had a problem where my tmpfs was too small, so I just created a tmp folder inside the /var/lib/libvirt/images folder that I used instead. It has to be an absolute path when you use a tmp folder, so just make sure that's set properly
TMPDIR="$(realpath ./tmp)" virt-sparsify --compress rocky9.qcow2 rocky9-compressed.qcow2
```
You now have a QCOW2 disk ready to go for sched_ext stuff.
