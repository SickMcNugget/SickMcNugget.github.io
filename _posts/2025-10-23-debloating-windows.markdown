---
layout: post
title:  "Debloating Windows and Running it with libvirt (QEMU + KVM)"
date:   2025-10-23 15:14:39 +08:00
categories: tutorial windows
published: true
---

# What?
I recently had to install Windows again for the first time in a long time. Whilst I use Windows 10 at home, I haven't completed an install for a very long time. At work I recently needed to install it, so I created a VM and did so with a Windows 11 ISO that was laying around.

Oh boy, I hate Windows 10, and getting Windows to play nice takes a small amount of effort. I wanted shared folders, automatic display resizing, and less bloat. Here's how I did that.

# The beginning
First, go through all the normal install steps. Create a local administrator account, or an organisation account, I don't really mind. After that, let's setup our shared drive functionality (which also happens to provide us with automatic display resizing).

## Shared Folders via VirtioFS
There are two webpages we need to visit for this step:
- [WinFsp] (a filesystem proxy to be downloaded and installed from the **guest**)
- [virtio-win] (Essentially guest additions if you've used VirtualBox. Download on the **host**, and then mount the ISO received on the **guest**)

### WinFsp
Visit the [WinFsp] page in the guest, download the installer (green button as of making this tutorial), and then simply run the installer. I don't know if you need to reboot, but do it to be safe.

### virtio-win
Visit the [virtio-win] wiki page on the host, download the `virtio-win.iso` file, and then mount this ISO to the guest. You may need to do this while the guest is shut down, or you can keep the guest running and use a network share. Again, I don't mind how you do this.

> One of my colleagues had a problem with his Windows bricking itself when he would mount the ISO from virt-manager, and we couldn't work out why (in our very short debug session). I pray you do not suffer the same fate.

Boot the VM now that the ISO is mounted, and double click `virtio-win-guest-tools.exe` inside the disk drive mounted on the guest. Run this installer to completion.

Congrats! You should now be able to mount a shared folder. Let's do that now, assuming you're using virt-manager (libvirt).

Shut down your VM. Tick `Enable shared memory` in the `Memory` pane. `Add Hardware` to the VM, and select `Filesystem`.
The `Filesystem` should have:
1. Driver: virtiofs
2. Source path: Your local directory that you want to share with the VM.
3. Target path: The name of the drive in the guest that will share the directory.
4. (Optional): Tick `Export filesystem as readonly mount` if that matters to you.

When the VM boots, lookup `Services` and enter that Application. You need to start the `VirtIO-FS Service`. I also set it to start automatically, but if it causes problems I'll report that in here later.

After the service is enabled, the share drive should be visible underneat `This PC` in `File Explorer`.

## Debloating Windows 11
This isn't just debloating, but also some quality of life changes that I prefer. Things such as returning the context menu from Windows 10 and dark mode aren't required, but I think they're much nicer than what Windows 11 thinks is right. Either way, there's an existing tool for the job

Enter [Win11Debloat]!

Simply run the below command to use my preferred options:
```ps1
& ([scriptblock]::Create((irm "https://debloat.raphi.re/"))) -Silent -RemoveApps -RemoveCommApps -RemoveGamingApps -DisableDVR -ClearStartAllUsers -DisableStartRecommended -DisableStartPhoneLink -DisableTelemetry -DisableSuggestions -DisableEdgeAds -DisableDesktopSpotlight -DisableLockscreenTips -DisableSettings365Ads -DisableSettingsHome -DisableBing -DisableCopilot -DisableRecall -DisableClickToDo -DisableEdgeAI -DisablePaintAI -DisableNotepadAI -RevertContextMenu -DisableMouseAcceleration -DisableStickyKeys -DisableFastStartup -ShowHiddenFolders -ShowKnownFileExt -HideDupliDrive -EnableDarkMode -DisableTransparency -DisableAnimations -TaskbarAlignLeft -CombineTaskbarWhenFull -HideTaskview -HideChat -DisableWidgets -EnableEndTask -HideHome -HideGallery -ExplorerToThisPC -HideOnedrive -Hide3dObjects -HideMusic -HideIncludeInLibrary -HideGiveAccessTo -HideShare
```
Alternatively, visit the [Win11DebloatParameters] page to decide on your own options.

This script shouldn't require a reboot.

Toodles

[WinFsp]: https://winfsp.dev/rel/
[virtio-win]: https://github.com/virtio-win/virtio-win.github.io/blob/main/Knowledge-Base/Driver-installation.md
[Win11Debloat]: https://github.com/Raphire/Win11Debloat
[Win11DebloatParameters]: https://github.com/Raphire/Win11Debloat/wiki/How-To-Use#parameters
