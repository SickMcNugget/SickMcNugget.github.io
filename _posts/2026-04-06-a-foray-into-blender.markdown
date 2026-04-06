---
layout: post
title: "A foray into Blender"
date: 2026-04-06 20:06:35 +08:00
categories: gamedev learning
published: true
---
# 3D Modelling is needed, huh?
Ever since I decided that I definitely want to undertake game development, I've made it my mission to understand at least a little bit about the entire pipeline of creating a game. Until I put something out there, my understanding of playtesting and marketing are going to suck, but one thing at a time. I've been trying to use ProBuilder in Unity as a quick way of making some levels that I can use as prototypes for testing game mechanics, but it quickly made me want to tear my eyes out when I tried to create a small pit. For some reason, the vertices are just a guideline and ProBuilder does whatever it feels like instead (it's not random, but there are two faces of a plane that don't translate correctly.

<figure>
  <img src="/assets/images/unity/probuilder_deformation.png">
  <figcaption style="text-align: right">A poor man's deformation.</figcaption>
</figure>

Anyway, the horror that was my morning made me decide that my efforts are better spent on learning a modicum of Blender. When I eventually put together a team to work on a larger game project, some understanding of the tools they use will help a lot in the future. To learn Blender, specifically so I can create a simple plane mesh, I'm following along with the [Blender studio fundamentals series](https://studio.blender.org/training/blender-fundamentals-45-lts/) (which is free, if it wasn't obvious).

<figure>
  <img src="/assets/images/unity/unity_plane.png">
  <figcaption style="text-align: right">The plane I want to create, minus the spheres.</figcaption>
</figure>

## Getting started
So far I've learned the simple things, such as transforming, rotating, scaling, making cubes and using the UI.
Some of the most important things so far have been:
- Pressing `G` (grab), `R` (rotate) or `S` (scale) while a mesh is selected allows you to perform those actions
  - If you press `X`, `Y`, or `Z` while in any of these transform modes, you can constrain to an axis.
  - If you enter a number while in any of these transform modes, you can precisely enter values for the transformation.
  - For example, `R X 45 Enter` will rotate the mesh 45 degrees about the X-axis. Super useful
- You can change the transform orientation to work with global coordinates, object local coordinates, or surface normal coordinates.
- `F3` lets you search all the keybinds
- `Shift-A` lets you perform many actions, such as making primitive shapes
- `Shift-Space` lets you select tools more quickly
- `N` opens up the sidebar, which has menus such as `Item`, `Tool` and `View`, which let you:
  - Modify dimensions of the selected mesh
  - Modify the settings of the currently selected tool
  - Change camera/scene based settings (such as the positioning of the 3D cursor)
- `Shift-D` duplicates the selected mesh

## Where do I need to go
This is super useful so far, but I still need to learn how to modify meshes so I can mould them as I require.
After that, I also need to be able to export them and import them into Unity, too.
