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
Let's open these matrices in reclass and use these relations to confirm it!

Lets form an Hypothesis that this is the layout of the 4x4 Matrix:
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/reclass-view-cam-hypothesis.png)

Using its relations we can see:
- Dot((-0.104, -0.06, 0.993), (0.5, -0.866, 0.00)) = *-3.9999999999998E-5* which is basically just *0* and is just floating-point precision error.
- Cross((-0.104, -0.06, 0.993), (0.5, -0.866, 0.00)) = *(0.860, 0.496, 0.121)* which basically gave us the second row (starting row's from 0)

>This is useful because, in reverse engineering, you may not always find a clean and complete 4×4 matrix. Sometimes you only find a few 
direction vectors (like Right and Up) scattered across memory or passed into functions. But thanks to vector math, you can reconstruct the missing third 
vector using cross products, or verify their relationship using dot products (which should be zero for perpendicular vectors). These help validate our findings
(maybe layout is unconventional or partially obfuscated)

In very rare circumstances, even though it might look like a matrix in might be a random cluster of floats that dont inherently mean anything,
Once we see that the values are orthogonal, have unit lengths, and contain a clear translation component (usually the larger values > 1.0f), 
we can reasonably assume we’re dealing with a real transformation matrix.

>To be extra sure, you can move the camera left, right, up, and down, and observe how the matrix reacts and see how these vectors react and accuratly 
verify them.

**Lets now inverse it to get the view matrix:**
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/cam-world-inverse.png)

We can see it exactly matches the second 4x4 matrix we see in memory (Camera World Matrix (Inverse) = View Matrix).

>We can also take dot(Right, Pos) and compare it against the translation component of the second matrix as another way to confirm this relationship.
  
Thus pretty much confirming the first matrix to be *Camera World Matrix* and the second matrix to be the *View Matrix*.

### **Looking at other matrices**

Here we see some other matrices:
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/reclass-view-projection-matrix.png)

The first Matrix looks like a projection matrix and the second one looks like a multiplication of the projection matrix with the view 
matrix (VP Matrix), the third one looks like the View-projection matrix without large translational values.

**Why?**  
Because they fit the general layout's expected of these matrices.  
To be sure lets plug in the view projection matrix into a world to screen function to see if we can confirm this.   

To verify i will manually input the entity coordinates into my world to screen program.  
Here is the entity whose position we’ll convert:  
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/view-proj-test-in-game.png)

Since this image shows that the entity is at the top-left corner of our screen at a resolution of 1920x1080 we can expect the screen coordinates to
be close to (0, 0) since the Top-left corner is the origin of the screen (0, 0), not bottom left as you might expect (unlike OpenGl).  
![ESP-Image1](/ViewProj-Blog/assets/images/part-3/view-proj-screen-pos-results.png)

Here we can see the output screen coordinates matches closely to what we expected.
 
>We'll cover how to get the entity's 3D coordinates and how World-to-Screen works in detail later, for now, this confirms the matrix is correct.

## **Trying to Rebuild the View-Projection Matrix**

Now that we have confrimed that this is infact the view projection matrix lets try to reconstruct it manually  

1. Take the suspected Projection Matrix
2. Multiply it by the inverse of the Camera World Matrix

![ESP-Image1](/ViewProj-Blog/assets/images/part-3/discrepency-vp-with-vp-multiply.png)  
Here for some reason it doesn't quite match's up although quite similar.    

*Next Step: Time to start tracing back the functions that write to the final View-Projection Matrix in memory.*

































