---
layout: post
title: "uBlock On Chrome"
date: 2025-11-10 09:38:41 +08:00
categories: chrome workaround linux tutorial
published: true
---

I've always used uBlock for ad-blocking, and Chrome are hell-bent on preventing it from working. As of November 10th, 2025 (when I'm creating this post. They've changed the steps again)

# How to get it working
First, in Chrome, click the puzzle piece in the top-left (or just go to manage extensions if you know how). In the top right of the manage-extensions page, you can enable a "Developer mode" button. Do that now.

Afterwards, download the [latest release of uBlock Origin](https://github.com/gorhill/uBlock/releases/tag/1.67.0), and unpack the archive somewhere that the folder won't annoy you.

In the manage-extensions page of Chrome, you'll want to "load unpacked" in the top left of the page and select the folder you just created. Maybe this will work for you, but as of today I've needed an additional step.

## Changing Chrome's launch options
I'm running KDE Plasma as my desktop environment at the moment, so I'll be editing Chrome's launch options through there. I'm also using the flatpak version of Chrome since it's easy to install and get going with.

To edit the launch options of chrome, open up the start menu, and search for Chrome. Right click it and press "Edit Application".

My current Chrome launch options are the following:
```
run --branch=stable --arch=x86_64 --command=/app/bin/chrome --file-forwarding com.google.Chrome @@u %U @@
```
To make uBlock work, we need to add a new flag at the end of the old command:
```
--disable-features=ExtensionManifestV2Unsupported,ExtensionManifestV2Disabled
```
So you should now have these command-line arguments:
```
run --branch=stable --arch=x86_64 --command=/app/bin/chrome --file-forwarding com.google.Chrome @@u %U @@ --disable-features=ExtensionManifestV2Unsupported,ExtensionManifestV2Disabled
```

Make sure you hit save, and voila! uBlock should work again.
