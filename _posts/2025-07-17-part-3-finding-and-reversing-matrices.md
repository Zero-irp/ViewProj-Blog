---
layout: post
title: "Part 3: Finding & Reversing Matrices"
date: 2025-07-17 00:00:00 +0530
permalink: /part-3-finding-and-reversing-matrices/
---

### **How We Search for Matrices in Memory**

A common appraoch is to switch Cheat Engine's value type in *float* and then:

1. Look Directly up and scan for the value *1.0f*.
2. Look Directly Down and scan for the value *-1.0f*.
3. Rescan to narrow the matrices down to matrices that are actively changing with camera orientation.

Once you narrow it down, you can inspect nearby memory regions and layout patterns (like a 4x4 float matrix) to identify which matrix it is

This technique works because these matrices are often stored in memory in a row-major or column-major 4x4 float array, and the directional 
vectors (Right, Up, Forward) reflect real-time camera movement

>Basically Something in the matrix must encode the camera's up/down rotation

### **Searching for Matrices in Ghost of Tsushima**

Lets start searching for interesting matrices in memory in ghost of tsushima!

**Looking up in game**
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/look-up-got.png)

**Seraching for 1.0f in cheat engine**
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/ce-first-scan.png)

**Now looking Down**
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/look-down-got.png)

**Searching for -1.0f in cheat engine**
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/ce-second-scan.png)

Now you're going to get a lot of results. Keep refining your search (e.g., by changing camera angles and scanning again) until the 
number of results stabilizes and stops changing, that's usually a good sign you've isolated the relevant addresses.  

Now When reverse engineering a game, you’ll often find that the View, Projection, and View-Projection matrices live at fixed or static memory addresses.  
**Why?**  
Mainly becuase these matrices are core componetns of the game's rendering pipeline and are global to the camera or render context.  

Now we can add all static memory address's and browse its memory layout's to see if we find some interesting matrices.  

**Like Here:**
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/cam-world-matrix-and-view-matrix.png)

Here the first 4x4 matrix look like the *Camera World Matrix* and the second 4x4 matrix looks like the *View Matrix*.

### **Confirmation of the View and Camera World Matrices:**

In [Part 2.1: View Matrix](/ViewProj-Blog/part-2.1-view-matrix/) we talked about the orthogonailty and relations of the fundamental Vectors.  
Let's use those relations to confrim it!

































