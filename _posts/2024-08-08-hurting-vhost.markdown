---
layout: post
title:  "Hurting vhost-net for fun"
date:   2024-08-08 15:18:33 +08:00
categories: linux performance
published: false
---
Recently, I've spent some time with jargon/acronyms like X11, QEMU, vhost-net, VNC and many, many others. All of these terms happened to intersect when it become worth testing the vhost process inside of QEMU using VNC to send graphics to X11-based window managers (whew!). Funnily enough, this happened to be a fairly painless process, since I decided that rough measurements were more useful than exact ones, a lesson I have learnt many times over.

# How it started
What is vhost-net, you may ask? Ever pinged a VM from another VM on the same host? You've used vhost-net (most likely!). It's a solution to reducing the number of context switches caused by VM network traffic. It's a kernel module containing a ring buffer that allows extremely fast communication with the virtio-net process inside the VM, since they both become aware of the memory space occupied by the ring buffer. [Red Hat][redhat-vhost] have a better explanation of the internals then I could ever give.

What is the effect of vhost performing poorly? Well, that's the question I wanted to answer. To begin, we need to create some VMs that have the packages we need to test vhost, graphically. I started with a debian 12.5.0 iso (just grab a new version from [the official site][debian-install]). Use this disk to create 2 VMs, _client_ and _server_. Make sure to set the desktop environment to Xfce! It probably isn't required, but I personally had problems getting VNC working with GNOME originally.

## Preparing our VMs
**Server**
```bash
# Install packages
sudo apt update -y && sudo apt upgrade -y && sudo apt install -y python3-tk x11vnc tightvncpasswd
# Create a VNC password (make it the same on both client and server)
echo <PASSWORD> | vncpasswd -f > $HOME/.vnc/passwd

# Start the VNC server
x11vnc -display 0 -rfbauth ~/.vnc/passwd -forever -nowf -cursor arrow -noxdamage -noscr -xkb &
```

```bash
# client
sudo apt update -y && sudo apt upgrade -y && sudo apt install -y tigervnc-viewer tightvncpasswd
# Create a VNC password (make it the same on both client and server)
echo <PASSWORD> | vncpasswd -f > $HOME/.vnc/passwd

# Connect to the VNC server
vncviewer <IP>:0 -PreferredEncoding raw -passwd $HOME/.vnc/passwd
```

# How it's going
Now that we've setup our VMs, we need a way to see what's going on between them from the host side. Enter `btop`, a flashier alternative to top. Give it a quick install using `sudo apt install btop`, and start it with `btop`. Now that we are ready to see network traffic from the host side, we can check which interfaces our VMs are connected to. If you've done this before, you know that they create and connect to `vnet`s. Go into virt-manager, choose one of the VMs you booted and go to it's "virtual hardware details", clicking on the NIC in the side panel. If you swap to the XML panel, you can see the target device it is connected to. In my case, the client is connected to `vnet4`.

Inside btop, you can press <b> or <n> to swap between different network interfaces. Do so until you reach the vnet interface attached to one of the VMs you created (it doesn't matter, since the upload of one VM should be equivalent to the download of another VM). 

## Cgroupifying our experiment
You're far enough now that you can do a baseline test to see how the VNC stream looks when vhost_net is performing as expected. Simply enter the server VM and run (WARNING, FLASHING COLOURS) `python3 graphictest.py`. You should see something like this on both VMs: 

<video loop="loop" width=640 style="display: block; margin: auto; margin-top: 0; padding: 0 0 15px 0">
  <source src="/assets/images/blog-vhost-happy-cropped.webm" type="video/webm">
</video>

Notice how both VMs remain in sync with one another, that will change later on.

At a resolution of 1280*800, each colour change produces a sudden burst of 153-154 Mibps of traffic, with each burst transferring a total of 4 MiB. Enough talking though, we want a video!

<video loop="loop" width=640 style="display: block; margin: auto; margin-top: 0; padding: 0 0 15px 0">
  <source src="/assets/images/vnc-traffic.webm" type="video/webm">
</video>

We'd like to break things, and to this, my weapon of choice is cgroups, so that we can limit the amount of CPU time available to the vhost process running on the host. Make sure to keep the VMs up when we start changing the cpu.max file, as you'll immediately see a change in performance when we do. Let's start by creating the cgroup we need. Since cgroups are hierarchical, we need to create two cgroups to ensure we are only using the controllers we need (technically this isn't necessary, but I prefer to be explicit about the controllers I am making use of).

```bash
# Two levels of cgroups need to be created so that the correct controllers are available for limiting.
cd /sys/fs/cgroup
mkdir vhost-toplevel
cd vhost-toplevel

# Add the desired controllers for the next cgroup
echo "+cpu +cpuset" >> cgroup.subtree_control
mkdir the-vhost-killer
cd the-vhost-killer
```

Now that we've created our cgroup, we need to prepare it and attach vhost.
The pgrep command below will give two vhosts and two QEMUs, choose one.

```bash
# Ask that only CPU 1 is used by processes in this cgroup
echo "1" > cpuset.cpus
# Find the vhost/qemu process id for limiting
pgrep vhost -a && pgrep qemu -a | awk '{print $1, $2, $3 $4;}'
# add the process id to the cgroup
echo "PID" >> cgroup.procs
```
If you're unsure whether this worked, you can use the same PID to check what cgroups the process belongs to:
```bash
cat /proc/<PID>/cgroup
```

That's it! We're ready to mess with vhost and see the results. To do so, we will be changning the `cpu.max` value to limit the amount of CPU time given to the vhost process. By default, the value is "max 100000", which means "I want 100000µs out of 100000µs to do my work." To live up to the name of the article, we can hurt vhost by setting it to use 1/1000th of the CPU time to perform the same task.

```bash
# Limit the CPU accordingly
echo "1000 1000000" > cpu.max   # 1/1000
echo "max  100000" > cpu.max     # Reset
```

And voila, here's what we see now:

<video loop="loop" width=640 style="display: block; margin: auto; margin-top: 0; padding: 0 0 15px 0">
  <source src="/assets/images/blog-vhost-sad-cropped.webm" type="video/webm">
</video>

Not only is vhost struggling struggling to keep up with sending the data, after its buffer fills up, we actually see skipped frames in the display output! Did I mention that this connection is running over TCP?

To remove the cgroup, either move the vhost PID to a new cgroup, or turn off the VM and then run:
```bash
cd /sys/fs/cgroup/
rmdir -p vhost-toplevel/the-vhost-killer
```

After installing these packages, I made a simple Tk script (using python bindings) to change the colour of the its window at a specified delay.
```py
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
```
Make sure this script is on the server end, since we installed python3-tk there.

After both of the VMs are setup, we're now ready to prepare ourselves

[redhat-vhost]: https://www.redhat.com/en/blog/introduction-virtio-networking-and-vhost-net
[debian-install]: https://www.debian.org/distrib/netinst
