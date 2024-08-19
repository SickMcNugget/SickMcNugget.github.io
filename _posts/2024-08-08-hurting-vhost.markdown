---
layout: post
title:  "Hurting vhost-net for fun"
date:   2024-08-08 15:18:33 +08:00
categories: linux performance
published: true
---
Recently, I've spent some time with the great acronyms of linux, including X11, QEMU, vhost-net and VNC. How does all this software link together? I needed to determine where vhost-net bottlenecks when it runs sluggishly. To test this, I needed VMs, hence the use of QEMU. To send a lot of traffic, I chose graphical data, hence the use of both X11 and VNC. Since all benchmarks are false anyway, I felt that it would be more useful if I instead used rough measurements to track down the source of the bottleneck, and thus today's experiment was born! 

Assume everything I talk about today is in reference to Linux, since we don't talk about **M\*\*\*\*\*\*\*t** or **A\*\*\*e**.

# How it started
What is vhost-net, you may ask? Ever pinged a VM from another VM on the same host? You've used vhost-net (most likely!). It's a solution to reducing the number of context switches caused by VM network traffic in the form of a kernel module. The module contains a ring buffer that allows extremely fast communication with the virtio-net process inside the VM, since they both become aware of the memory space occupied by the ring buffer. [Red Hat][redhat-vhost] have a better explanation of the internals than I do.

To begin, we need to create some VMs that have the packages we need for this experiment. The preparation is composed of 3 stages: The **Host**, The **Server** and The **Client**. Let's begin with the host.

## Preparing our host to run virtual machines
I'm running the following versions of software (in case their internals change in the future):
<div class="language-bash highlighter-rouge highlight"><pre class="highlight">
<code><span style="color:green">qemu-system-x86-64</span> 1:7.2+dfsg-7+deb12u6
<span style="color: green">libvirt-daemon</span> 9.0.0-4
<span style="color: green">virt-manager</span> 1:4.1.0-2
<span style="color: green">btop</span> 1.2.13-1
<span style="color: green">linux-image-amd64</span> 6.1.99-1 <span style="color: grey">#6.1.0-23</span>
<span style="color: green">linux-headers-6.1.0-23-amd64</span> 6.1.99-1
<span style="color: green">x11vnc</span> 0.9.16-9
<span style="color: green">tightvncpasswd</span> 1:1.3.10-7<!--
<span style="color: green">tightvncserver</span> 1:1.3.10-7-->
<span style="color: green">tigervnc-viewer</span> 1.12.0+dfsg-8
</code></pre></div>

Install [btop][btop], [virt-manager][virt-manager-install], [libvirt][libvirt-install] and [QEMU][QEMU-install].
```bash
sudo apt install btop libvirt-daemon libvirt-dev virt-manager qemu-system
```

I started with a debian 12.5.0 iso (just grab a new version from [the official site][debian-install]). Use this disk to create 2 VMs, _client_ and _server_. Make sure to set the desktop environment to Xfce! It probably isn't required, but I personally had problems getting VNC working with GNOME. Once both the VMs are ready to be booted, start them up and prepare to install packages on them.

## Preparing our Virtual Machines for sharing graphics
On the **server**, we need a VNC server (x11vnc), python3 Tk bindings (python3-tk) and a password generator for VNC (tightvncpassword).
```bash
# Install packages
sudo apt update -y && sudo apt upgrade -y && sudo apt install -y python3-tk x11vnc tightvncpasswd
```
Once our packages are installed, we can also prepare our test script, create a VNC password and run the server!
```bash
echo << EOF > graphictest.py
from tkinter import Tk
from tkinter import ttk

COLORS = ["red", "green", "blue", "yellow", "magenta", "cyan", "white", "black"]
DELAY_MS = 1000
counter = 0

root = Tk()
root.title("Graphics test")
root.geometry("1280x800")

def change_color():
  global counter
  root.configure(bg=COLORS[counter])
  counter = (counter + 1) % len(COLORS)
  root.after(DELAY_MS, change_color)

change_color()
root.mainloop()
EOF 

# Create a VNC password (make it the same on both client and server)
echo <PASSWORD> | vncpasswd -f > $HOME/.vnc/passwd

# Start the VNC server
x11vnc -display 0 -rfbauth ~/.vnc/passwd -forever -nowf -cursor arrow -noxdamage -noscr -xkb &
```
The settings for x11vnc above make it as _inefficient_ as possible, since we want to generate as much network traffic as we can.

It's now time to set up the **client** to connect to the server. Start by installing a VNC client (tigervnc-viewer) and password generator.
```bash
# client
sudo apt update -y && sudo apt upgrade -y && sudo apt install -y tigervnc-viewer tightvncpasswd
```
Now we can generate the same password we made for the server, and then connect over VNC!

```bash
# Create a VNC password (make it the same on both client and server)
echo <PASSWORD> | vncpasswd -f > $HOME/.vnc/passwd

# Connect to the VNC server
vncviewer <IP>:0 -PreferredEncoding raw -passwd $HOME/.vnc/passwd
```

# How it's going
Now that both VMs are ready, we need a way to see what's going on between them from the host side. We'll use the `btop` program we installed earlier for this. Familiarise yourself with the keybinds and layout of btop, and when you are ready, prepare to use the network statistics of the tool. We need to check which interfaces our VMs are connected to so we can monitor them. By default, my VMs create `vnet` interfaces. The vnet interface can be matched to the VM using the *NIC* page on virt-manager, and viewing the XML:
```xml
<interface type="network">
  <mac address="52:54:00:f2:36:0f"/>
  <source network="default" portid="b3f056df-d5fd-42c7-8f58-7be9c2ccf851" bridge="virbr0"/>
  <target dev="vnet2"/> <!-- HERE -->
  <model type="virtio"/>
  <alias name="net0"/>
  <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
</interface>
```

Now that we know the interface, navigate to it using the btop keybinds.

## Running the experiment
You're far enough now that you can do a baseline test to see how the VNC stream looks when vhost_net is performing as expected. Simply enter the server VM and run (note: flashing colours) `python3 graphictest.py`. You should see something like this on both VMs: 

<video autoplay controls loop="loop" width=640 style="display: block; margin: auto; margin-top: 0; padding: 0 0 15px 0">
  <source src="/assets/images/blog-vhost-happy-cropped.webm" type="video/webm">
</video>
Notice how the VMs remain in sync when they update, we'll break this later on. At a resolution of 1280*800, each colour change produces a sudden burst of 153-154 Mibps of traffic, with each burst transferring a total of 4 MiB. Your network graphic should look something like this:

<video autoplay controls loop="loop" width=640 style="display: block; margin: auto; margin-top: 0; padding: 0 0 15px 0">
  <source src="/assets/images/vnc-traffic.webm" type="video/webm">
</video>

## One cgroup to rule them all
To punish vhost_net, I've chosen cgroups as my weapon. Specifically, [cgroupsv2][cgroups], as my kernel supports it by default. Cgroups allows us to limit the amount of CPU time available to processes, and we can attach this functionality to the process while it's running. When we make changes to the `cpu.max` file later on, keep your eyes on both the VMs as you'll see immediate changes in performance when we do. Lets create the cgroup we need and attach the vhost process to it.

```bash
# go to the root directory of the cgroup2 filesystem
cd $(mount | grep cgroup | awk '{print $3}')
# create the cgroup we will use to limit vhost
mkdir the-vhost-killer
cd the-vhost-killer

# Ensure only 1 cpu is used by vhost
echo "1" > cpuset.cpus
# find the vhost process and/or its associated QEMU process for limiting
pgrep vhost -a && pgrep qemu -a | awk '{print $1, $2, $3 $4;}'
# add the process id found above to the cgroup
echo "PID" >> cgroup.procs
```
If you're unsure whether this worked, you can use the same PID to check what cgroups the process belongs to:
```bash
cat /proc/<PID>/cgroup

```

If you want to ensure that your cgroup only has access to the relevant controllers, you can optionally create two levels of cgroups. This isn't strictly necessary, however.
```bash
# Two levels of cgroups need to be created so that only specific controllers are available for limiting.
cd /sys/fs/cgroup
mkdir vhost-toplevel
cd vhost-toplevel

# Add the desired controllers for the next cgroup
echo "+cpu +cpuset" >> cgroup.subtree_control
mkdir the-vhost-killer
cd the-vhost-killer
```

That's it! We're ready to mess with vhost and see the results. To do so, we'll change the `cpu.max` value to limit the amount of CPU time given to the vhost process. By default, the value is 'max 100000,' which means 'I want up to 100000µs every 100000µs to do my work.' Since this is a comfortable amount of time for vhost_net to get its work done, we are going to give it a thousandth of the time instead, to hurt it.
```bash
# Limit the CPU accordingly
echo "1000 1000000" > cpu.max   # 1/1000
echo "max  100000" > cpu.max     # Reset
```

And voila, here's what we see now:

<video autoplay controls loop="loop" width=640 style="display: block; margin: auto; margin-top: 0; padding: 0 0 15px 0">
  <source src="/assets/images/blog-vhost-sad-cropped.webm" type="video/webm">
</video>

Not only is vhost struggling struggling to keep up with sending the data, after a short delay we actually see skipped frames in the display output! Did I mention that this connection is running over TCP? Somewhere, either a time-based drop is occurring, or a buffer is filling up.

To remove the cgroup, either move the vhost PID to a new cgroup, or turn off the VM and then run:
```bash
cd $(mount | grep cgroup | awk '{print $3}')
rmdir -p vhost-toplevel/the-vhost-killer
```

In the next part of this article, I'll look deeper into the kernel to find the source of the bottleneck using eBPF and related tools!

[redhat-vhost]: https://www.redhat.com/en/blog/introduction-virtio-networking-and-vhost-net
[debian-install]: https://www.debian.org/distrib/netinst
[virt-manager-install]: https://virt-manager.org/
[libvirt-install]: https://libvirt.org/compiling.html
[QEMU-install]: https://www.qemu.org/download/#linux
[btop]: https://github.com/aristocratos/btop
[cgroups]: https://docs.kernel.org/admin-guide/cgroup-v2.html
