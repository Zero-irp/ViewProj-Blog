---
layout: post
title: "Part 7: Final ESP with Imgui"
date: 2025-07-17 00:00:00 +0530
permalink: /part-7-final-esp/
---

<style>
.cpp-code {
  background: #1e1e1e;
  color: #dcdcdc;
  font-family: 'Fira Code', 'Consolas', monospace;
  font-size: 0.95rem;
  padding: 1rem 1.25rem;
  border-radius: 10px;
  line-height: 1.5;
  overflow-x: auto;
  white-space: pre;
  box-shadow: 0 2px 8px rgba(0,0,0,0.3);
}
.cpp-code .keyword { color: #569cd6; }
.cpp-code .type { color: #4ec9b0; }
.cpp-code .comment { color: #6a9955; font-style: italic; }
.cpp-code .number { color: #b5cea8; }
.cpp-code .function { color: #dcdcaa; }
.cpp-code .string { color: #ce9178; }
.cpp-code .var { color: #9cdcfe; }
</style>

### Introduction:

Now that we’ve covered how to capture entities and transform their positions from 3D world space to 2D screen space, it’s time to put everything 
together and build our final ESP overlay using ImGui.

In Part 5, we captured all entity positions by hooking the game’s Z-axis function. Each entity that accessed this function was stored in an 
unordered set, giving us a complete list of active objects in the game world.

Then, in Part 6, we learned how to convert these 3D positions into screen coordinates using the World to Screen function. This transformation 
allows us to take an entity’s world position and figure out exactly where it should appear on the player’s monitor.

With both pieces in place, the entity list and the ability to map world positions to screen coordinates we can now iterate through all 
captured entities, run each one through our World to Screen function, and draw a box, snapline, or any other visual indicator on top of them 
using ImGui.

### Code Explanation:

For this one i will be using the DirectX library to make it simpler.  

Let's begin:

**Step 1: Reading Entities and Their Data**

<div class="cpp-code"><span class="function">ReadProcessMemory</span>(<span class="var">hProcess</span>, (<span class="type">BYTE</span>*)<span class="var">entityAddr</span>, &amp;<span class="var">curEntity</span>, <span class="function">sizeof</span>(<span class="var">curEntity</span>), <span class="keyword">nullptr</span>);

<span class="type">float</span> <span class="var">curHealth</span> = <span class="number">0</span>;
<span class="function">ReadProcessMemory</span>(<span class="var">hProcess</span>, (<span class="type">BYTE</span>*)<span class="var">curEntity</span> + <span class="number">0x1974</span>, &amp;<span class="var">curHealth</span>, <span class="function">sizeof</span>(<span class="var">curHealth</span>), <span class="keyword">nullptr</span>);

<span class="type">Vec3</span> <span class="var">curEntPos</span> = { <span class="number">0</span>, <span class="number">0</span>, <span class="number">0</span> };
<span class="function">ReadProcessMemory</span>(<span class="var">hProcess</span>, (<span class="type">BYTE</span>*)<span class="var">curEntity</span> + <span class="number">0x120</span>, &amp;<span class="var">curEntPos</span>, <span class="function">sizeof</span>(<span class="var">curEntPos</span>), <span class="keyword">nullptr</span>);
</div>

- Purpose: Pull information from the target process about each entity.
- curEntity: Pointer to the current entity accessing the Z-axis function.
- curHealth: Health value at offset 0x1974 to filter out dead entities.
- curEntPos: World position at offset 0x120

> entityAddr is the stash address of our trampoline hook where it stashs addresses which have accessed the z 
axis.

**Step 2: Filtering Unique and Relevant Entities**
<div class="cpp-code"><span class="keyword">if</span> (<span class="var">curEntity</span> != <span class="number">0</span> && <span class="var">curHealth</span> != <span class="number">0</span> && <span class="var">LocalPlayerAddr</span> != <span class="var">curEntity</span> && <span class="var">uniqueEntities</span>.<span class="function">find</span>(<span class="var">curEntity</span>) == <span class="var">uniqueEntities</span>.<span class="function">end</span>())
{
    <span class="var">uniqueEntities</span>.<span class="function">insert</span>(<span class="var">curEntity</span>);
    <span class="function">std::cout</span> &lt;&lt; <span class="string">"New unique entity found: "</span> &lt;&lt; <span class="function">std::hex</span> &lt;&lt; <span class="var">curEntity</span> &lt;&lt; <span class="function">std::endl</span>;
}
</div>

- Purpose: Only store entities that are alive, not the local player, and haven’t already been inserted.
- This ensures your unordered set uniqueEntities only contains valid targets for the ESP.

**Step 3: Toggle ESP**
<div class="cpp-code"><span class="keyword">if</span> (<span class="function">GetAsyncKeyState</span>(<span class="var">VK_F3</span>) &amp; <span class="number">1</span>)
{
    <span class="var">ESP</span> = !<span class="var">ESP</span>;
}
</div>

- Purpose: Simple toggle using the F3 key to enable or disable the ESP overlay at runtime.

**Step 4: Loop Through Entities and Get Screen Positions**
<div class="cpp-code"><span class="keyword">for</span> (<span class="keyword">const</span> <span class="keyword">auto</span>& <span class="var">entity</span> : <span class="var">uniqueEntities</span>)
{
    <span class="function">ReadProcessMemory</span>(<span class="var">hProcess</span>, (<span class="type">BYTE</span>*)<span class="var">entity</span> + <span class="number">0x120</span>, &amp;<span class="var">curEntPos</span>, <span class="function">sizeof</span>(<span class="var">curEntPos</span>), <span class="keyword">nullptr</span>);

    <span class="type">XMMATRIX</span> <span class="var">viewProj</span>;
    <span class="function">ReadProcessMemory</span>(<span class="var">hProcess</span>, (<span class="type">BYTE</span>*)<span class="var">baseAddr</span> + <span class="number">0x1EC6BC0</span>, &amp;<span class="var">viewProj</span>, <span class="function">sizeof</span>(<span class="var">viewProj</span>), <span class="keyword">nullptr</span>);

    <span class="type">Vec2</span> <span class="var">screenFeet</span> = { <span class="number">0</span> , <span class="number">0</span> };
    <span class="function">WorldToScreen</span>(<span class="var">curEntPos</span>, <span class="var">screenFeet</span>, <span class="var">viewProj</span>, <span class="number">1920</span>, <span class="number">1080</span>);

    <span class="type">Vec2</span> <span class="var">screenHead</span> = { <span class="number">0</span> , <span class="number">0</span> };
    <span class="var">curEntPos</span>.z = <span class="number">173.f</span>
    <span class="function">WorldToScreen</span>(<span class="var">curEntPos</span>, <span class="var">screenHead</span>, <span class="var">viewProj</span>, <span class="number">1920</span>, <span class="number">1080</span>);
</div>

- Purpose: Iterate over all captured entities and get their current world positions.
- View-projection matrix: Needed to convert the world position to screen coordinates.
- WorldToScreen: Converts (x, y, z) → (screenX, screenY) for rendering.

**Step 5: Drawing Snaplines Using ImGui**
<div class="cpp-code"><span class="type">ImVec2</span> <span class="var">screenSize</span> = <span class="function">ImGui</span>::<span class="function">GetIO</span>().DisplaySize;
<span class="type">ImVec2</span> <span class="var">from</span> = <span class="function">ImVec2</span>(<span class="var">screenSize</span>.x / <span class="number">2.0f</span>, <span class="var">screenSize</span>.y); <span class="comment">// bottom center</span>
<span class="type">ImVec2</span> <span class="var">to</span> = <span class="function">ImVec2</span>(<span class="var">screenPos</span>.x, <span class="var">screenPos</span>.y);

<span class="type">ImU32</span> <span class="var">color</span> = <span class="function">IM_COL32</span>(<span class="number">255</span>, <span class="number">0</span>, <span class="number">0</span>, <span class="number">255</span>); <span class="comment">// red</span>
<span class="keyword">if</span> (<span class="var">screenPos</span>.x &lt; <span class="number">1920</span> &amp;&amp; <span class="var">screenPos</span>.y &lt; <span class="number">1080</span> &amp;&amp; <span class="var">screenPos</span>.x &gt; <span class="number">0</span> &amp;&amp; <span class="var">screenPos</span>.y &gt; <span class="number">0</span>)
{
    <span class="function">ImGui</span>::<span class="function">GetBackgroundDrawList</span>()-><span class="function">AddLine</span>(<span class="var">from</span>, <span class="var">to</span>, <span class="var">color</span>, <span class="number">1.5f</span>);
}
</div>

<div class="cpp-code"><span class="comment">// Height of the box</span>
<span class="keyword">float</span> <span class="var">h</span> = <span class="var">screenFeet</span>.y - <span class="var">screenHead</span>.y;

<span class="comment">// Width proportional to height</span>
<span class="keyword">float</span> <span class="var">w</span> = <span class="var">h</span> / <span class="number">2.0f</span>;

<span class="comment">// Top-left and bottom-right corners</span>
<span class="type">ImVec2</span> <span class="var">topLeft</span>(<span class="var">screenHead</span>.x - <span class="var">w</span> / <span class="number">2.0f</span>, <span class="var">screenHead</span>.y);
<span class="type">ImVec2</span> <span class="var">bottomRight</span>(<span class="var">screenHead</span>.x + <span class="var">w</span> / <span class="number">2.0f</span>, <span class="var">screenFeet</span>.y);

ImGui::<span class="function">GetBackgroundDrawList</span>()-><span class="function">AddRect</span>(
	<span class="var">topLeft</span>,
	<span class="var">bottomRight</span>,
	<span class="type">IM_COL32</span>(<span class="number">255</span>, <span class="number">0</span>, <span class="number">0</span>, <span class="number">255f</span>),
	<span class="number">0.0f</span>,
	<span class="number">0</span>,
	<span class="number">2.0f</span>
);
</div>

- Purpose: Draw a line from the bottom center of the screen to each entity’s screen position.
- Screen boundary check: Ensures you don’t draw off-screen.
- ImGui draw list: AddLine draws directly on the overlay background.

### TL;DR of the Code Flow

1. Read entity pointers from memory.
2. Filter only relevant entities and store them in an unordered set.
3. Toggle ESP with a keypress.
4. Loop through entities, read their world positions.
5. Convert positions to 2D screen coordinates.
6. Draw boxes or snaplines with ImGui at the screen positions.

Result:  

![IDA-screenshot](/ViewProj-Blog/assets/images/part-7/got_esp.gif)

### Closing Thoughts:

All this reversing just because the inverse of the camera world matrix multiplied with the projection matrix didn’t yield the correct VP matrix. From 3D graphics to tracing matrices, 
to reversing SIMD instructions, to detour hooks, to trampolines, to ESP, we’ve done a lot. There’s honestly way more to how an “ESP” really works than what I’ve covered here, but I’m 
just going to end it here, I’m sick of writing all this.
