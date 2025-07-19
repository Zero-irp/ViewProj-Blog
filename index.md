---
layout: default
title: "Index"
---

# Reversing The View-Projection Matrix Blog

Welcome to my blog series on reverse engineering the matrix pipeline of a AAA game.

## Parts

- [Part 1: Introduction]({{ site.baseurl }}/part-1-intro)
- <div style="color:blue">Part 2: Understanding 3D Transformation Matrices</div>
  - [Part 2.1: View Matrix]({{ site.baseurl }}/part-2.1-view-matrix)
  - [Part 2.2: Projection Matrix]({{ site.baseurl }}/part-2.2-projection-matrix)
  - [Part 2.3: View Projection Matrix]({{ site.baseurl }}/part-2.3-View-Projection-matrix)
- [Part 3: Finding & Reversing Matrices]({{ site.baseurl }}/part-3-finding-and-reversing-matrices)
- <div style="color:blue">Part 4: Dissecting the Game’s Matrix Construction Pipeline And SIMD Math</div>
  - [Part 4.1: Tracing the Matrix Construction Path]({{ site.baseurl }}/part-4.1-tracing-matrix-construction)
  - [Part 4.2: Reversing SIMD Instructions for Matrix Math]({{ site.baseurl }}/part-4.2-reversing-simd-instrustions)
  - [Part 4.3: Redundancy, Junk Code, and Repetition]({{ site.baseurl }}/part-4.3-redundancy-junk-repetition)
- [Part 5: Trampoline Hooking to Capture Entity Positions]({{ site.baseurl }}/part-5-trampoline-hooking)
- [Part 6: World To Screen Explanation And Code]({{ site.baseurl }}/part-6-w2s)
- [Part 7: Final ESP with Imgui]({{ site.baseurl }}/part-7-final-esp)

