---
layout: default
title: "Index"
---

# Reversing The Construction Of The View-Projection Matrix (Game Engine Reversing)

Welcome to my write-up series on reverse engineering the Rendering pipeline of an AAA game.

## Parts

- [Part 1: Introduction]({{ site.baseurl }}/part-1-intro)
- <div style="color:#6495ED">Part 2: Understanding 3D Transformation Matrices</div>
  - [Part 2.1: View Matrix]({{ site.baseurl }}/part-2.1-view-matrix)
  - [Part 2.2: Projection Matrix]({{ site.baseurl }}/part-2.2-projection-matrix)
  - [Part 2.3: View Projection Matrix]({{ site.baseurl }}/part-2.3-View-Projection-matrix)
- [Part 3: Finding & Reversing Matrices]({{ site.baseurl }}/part-3-finding-and-reversing-matrices)
- <div style="color:#6495ED">Part 4: Dissecting the Game’s Matrix Construction Pipeline And SIMD Math</div>
  - [Part 4.1: Tracing the Matrix Construction Path]({{ site.baseurl }}/part-4.1-tracing-matrix-construction)
  - [Part 4.2: Reversing SIMD Instructions for Matrix Math]({{ site.baseurl }}/part-4.2-reversing-simd-instrustions)
  - [Part 4.3: Completing the View-Projection Matrix]({{ site.baseurl }}/part-4.3-completing-view-proj-matrix)
  - [Part 4.4: Optimizing Redundant SIMD Instructions (For Fun)]({{ site.baseurl }}/part-4.4-optimizing-redundant-simd-instructions)
  - [Part 4.5: Detour Hooking to Optimize SIMD Operations (For Fun)]({{ site.baseurl }}/part-4.5-detour-hooking-simd-operations)
- [Part 5: Trampoline Hooking to Capture Entity Positions]({{ site.baseurl }}/part-5-trampoline-hooking)
- [Part 6: World To Screen Explanation And Code]({{ site.baseurl }}/part-6-w2s)
- [Part 7: Final ESP with Imgui]({{ site.baseurl }}/part-7-final-esp)

