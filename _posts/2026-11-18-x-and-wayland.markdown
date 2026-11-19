---
layout: post
title: "X and Wayland"
date: 2026-11-18 11:48:55 +08:00
categories: 
published: false
---
I realised today that I don't understand X and wayland, so it's worth reading up on how they work. This is especially valuable since I often find myself trying to work out how they each function, creating a cycle of *sort of knowing* followed by **completely forgetting** what I *sort of knew*.

# X
I'm going to start with the x-basics page from <https://magcius.github.io/xplain/article/x-basics.html>, as it seems to be what I'm looking for.

Supposedly, "X" should be called "the X Window System" as that is the name for the networked display system itself. 

Xorg refers to the most popular X server implementation. 

X11 refers to the standardised network protocol that X servers and clients use to communicate with one another.

Applications themselves usually use client libraries such as Xlib and xcb to write the appropriate wire format onto a socket.

## Simple requests
X clients create *requests* asking the X server to do something. `CreateWindow` is one such request telling the X server to create a window. At this point, the X server will create that window and send an *event* back to the X client. The client will only receive events that they've subscribed to. For example, with the Xlib library, the `XSelectInput` function can select events to listen to.

## Windows
A *window* in X11 is a structure that allows an X client connecting to an X server to display something on the screen and take input, too. Windows are defined by x, y, width, height (bounding rectangle). Windows also have a *stacking order* (essentially a Z-coordinate from game engines).

Showing a window in X11 is called *mapping* a window officially. *Unmapping* a window is the official name for hiding it. Windows start unmapped until an X client sends a `MapWindow` request to an X server. Similarly, an `UnmapWindow` request will unmap a mapped window (or no-op if the window is already unmapped). Unmapping a window doesn't destroy it; it can be mapped again later. Supposedly this is how minimising windows is implemented in most window managers.

## Drawing Pixels
The X Window System doesn't store the pixels for each window at all. Instead, there is one buffer of pixels for the screen called *the front buffer*. Windows **borrow** pixels to draw to this buffer. If the server needs pixels from a window, it sends an `Expose` event to get them. The window sends drawing commands back to the server, which then processes the drawing commands to update the pixels occupied by that window.

## Occlusion
The X server stores a record of visible pixels for each window. This data structure is called the *clip list* (in Xorg's code). This list stores rectangles containing the areas where a window is visible. This data structure is known as a *region* in X11 programming.

For something such as drawing a circle, the X SHAPE Extension can tell the X server to use a bounding region instead of a bouding rectangle. You just construct a circle region and then set the window's bouding region to be that circle. This is how xeyes and oclock get their shapes.

This extension does not allow for semitransparency. Pixels are still either opaque or transparent, but window bounding regions can be more complex than before with X SHAPE. X SHAPE is specifically subtractive. This means it can only carve away from the original rectangular window shape to create something custom.

## Pixmaps
When a window is created and an image is loaded, a Pixmap is created, which is a memory backing for pixel storage on the X Server. The pixels for the Pixmap image are only uploaded once using the `PutImage` request.

Whenever an `Expose` event needs the pixels of the underlying image from the client, the client uses a `CopyArea` request which tells the X server to use the pixels it already has from its own Pixmap to fill the area.

## Input in X11
X11 defines a large number of input events. One such event is called `Motion`, and can be used to know when the mouse pointer is moved over a window (and the location it ended at). There is also an `Enter` and `Leave` event for when the mouse pointer enters the window and leaves it.

## The Window Tree
Windows can have children, so that there is a hierarchical structure. This is called the window tree. After creating a window, clients can reparent one window to another using the `ReparentWindow` request. By default, all windows are parented to the root window.
