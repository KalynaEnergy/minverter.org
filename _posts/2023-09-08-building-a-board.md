---
layout: post
title: Board design 
tags: 
categories: 
---

It'll be hard to do anything without the capability of making useful schematics and board layouts. I'm using [Fusion 360](https://www.autodesk.ca/en/products/fusion-360/overview), a reasonbly nice commercial CAD system, the electronics part of which grew out of Eagle. It's been awhile, and they are constantly updating and changing things, so I figured I'd go through a [tutorial](https://www.youtube.com/playlist?list=PLmA_xUT-8UlL80Xm8Gxz98YNum3I9GInr) while working up a simple board. I picked building a half bridge driver board for the [Nexperia GAN190-650FBE](https://assets.nexperia.com/documents/data-sheet/GAN190-650FBE.pdf) GaN transistor, because these are new and interesting devices for which dev boards are not available. 

The bill of materials isn't too complicated. Here is what I'll need:
  * GAN190-650FBE (x2)
  * NCP51530AMNTWG (half-bridge driver, as recommended by [AN90041](https://assets.nexperia.com/documents/application-note/AN90041.pdf))
  * 4 pin header, for the driver power and HIN and LIN inputs
  * Power input, ground, and output
  * a LED so we can see if it's on
  * some resistors and diodes (to step down the driver output to the ~6.2V Nexperia's e-mode GaNs like)
And I think that's it. I'm trying to keep this simple.

Finding suitable footprints for components is a key part of dealing with boards. Building libraries of these, tested and known to work with your manufacturing processes, is a major task. Lots of things are available online from various sources, for example the driver chip has a footprint in [UltraLibrarian](https://app.ultralibrarian.com/details/089370B2-87B9-11EB-9033-0A34D6323D74/onsemi/NCP51530AMNTWG?ref=digikey). But the GAN190-650FBE is new enough that I haven't found something online. That's okay, it just means we have to create the component ourselves, based on the datasheet. [Here](https://www.youtube.com/watch?v=zqar0XWtFaY) is the tutorial I'll be following for that. Wish me luck.