---
layout: post
title: "Part 5: Trampoline Hooking to Capture Entity Positions"
date: 2025-07-17 00:00:00 +0530
permalink: /part-5-trampoline-hooking/
---

<style>
.cpp-code {
  background: #1e1e1e;
  color: #dcdcdc;
  padding: 12px;
  border-radius: 8px;
  font-family: "Cascadia Code", Consolas, "Liberation Mono", Menlo, monospace;
  font-size: 14px;
  line-height: 1.45;
  overflow-x: auto;
  white-space: pre;
}

/* token colors */
.cpp-code .kw      { color: #569cd6; }  /* keywords (if, for, return, etc.) */
.cpp-code .type    { color: #4ec9b0; }  /* built-in / user types */
.cpp-code .fn      { color: #dcdcaa; }  /* functions / intrinsics */
.cpp-code .num     { color: #b5cea8; }  /* numbers */
.cpp-code .var     { color: #9cdcfe; }  /* variables */
.cpp-code .const   { color: #ce9178; }  /* string literals */
.cpp-code .comment { color: #6a9955; font-style: italic; } /* comments */
.cpp-code .preproc { color: #c586c0; }  /* #include, #define */
</style>

### What is Trampoline Hooking?

So far, we’ve played with detours and code caves. But now we’re stepping into trampoline hooking.  

The name is pretty self-explanatory: imagine program execution running along smoothly, then suddenly hitting our hook. Instead of just 
overwriting instructions permanently, we “bounce” execution away (into our own custom code allocated with VirtualAllocEx), do what we need, 
and then jump back into the original function flow (just after our hook), just like landing back on the trampoline.

### The Plan:  

Our target is a function instruction that accesses the Z-axis of entities. By trampoline hooking it, we can capture every entity that passes 
through this code. That way we build up a clean list of entity positions we can later use for world-to-screen ESP rendering. 

We can’t simply overwrite the instruction permanently, that would break the game’s logic. Instead, we’ll:

- Redirect execution at the Z-axis instruction into our allocated memory.
- Inside our hook, log which entity triggered this instruction and execute instructions overwritten by our hook.
- After we’ve done our work, trampoline execution back into the original flow of the game, so nothing breaks.

**Let's begin**

### The Hook

Let's start with how we are going to hook:

For this hook, we’re not going to rely on a code cave. Instead, we’ll dynamically allocate memory inside the game process using the Windows 
API function VirtualAllocEx. This function lets us carve out a chunk of memory in the target process and mark it as executable, so we can 
safely drop in our own custom code.

Let's start with getting:

- Process ID
- Module Base Address
- Handle to the Process

<div class="cpp-code"><span class="comment">// Get Process Id</span>
<span class="var">ProcId</span> = <span class="fn">GetProcId</span>(<span class="const">L"GhostOfTsushima.exe"</span>);
<span class="fn">CHECK</span>(<span class="var">ProcId</span>);

<span class="fn">std::cout</span> &lt;&lt; <span class="const">"PROC ID : "</span> &lt;&lt; <span class="var">ProcId</span> &lt;&lt; <span class="fn">std::endl</span>;

<span class="comment">// Get Base Address</span>
<span class="var">baseAddr</span> = <span class="fn">GetModuleBaseAddress</span>(<span class="var">ProcId</span>, <span class="const">L"GhostOfTsushima.exe"</span>);
<span class="fn">CHECK</span>(<span class="var">baseAddr</span>);

<span class="fn">std::cout</span> &lt;&lt; <span class="const">"Base Address : "</span> &lt;&lt; <span class="fn">std::hex</span> &lt;&lt; <span class="var">baseAddr</span> &lt;&lt; <span class="fn">std::endl</span>;

<span class="comment">// Get a Handle To The Process</span>
<span class="var">hProcess</span> = <span class="fn">OpenProcess</span>(<span class="preproc">PROCESS_ALL_ACCESS</span>, <span class="num">0</span>, <span class="var">ProcId</span>);
<span class="fn">CHECK</span>(<span class="var">hProcess</span>);
</div>

### Setting Up Memory for the Trampoline

Now, we allocate some executable memory inside the game with VirtualAllocEx.

<div class="cpp-code"><span class="type">LPVOID</span> <span class="fn">VirtualAllocEx</span>(
    <span class="type">HANDLE</span> <span class="var">hProcess</span>,        <span class="comment">// Handle to target process</span>
    <span class="type">LPVOID</span> <span class="var">lpAddress</span>,       <span class="comment">// Desired address (NULL lets the system decide)</span>
    <span class="type">SIZE_T</span> <span class="var">dwSize</span>,          <span class="comment">// Size of memory block</span>
    <span class="type">DWORD</span> <span class="var">flAllocationType</span>, <span class="comment">// MEM_COMMIT | MEM_RESERVE for reserving and allocating physical memory</span>
    <span class="type">DWORD</span> <span class="var">flProtect</span>         <span class="comment">// Memory protection (e.g., PAGE_EXECUTE_READWRITE)</span>
);
</div>

for our case i will use *VirtualAllocEx* with these arguments:

<div class="cpp-code">
<span class="type">uintptr_t</span> <span class="var">reserve</span> = (<span class="type">uintptr_t</span>)<span class="fn">VirtualAllocEx</span>(
    <span class="var">hProcess</span>,                <span class="comment">// The process handle we opened earlier</span>
    <span class="kw">NULL</span>,                    <span class="comment">// NULL = let Windows decide where to put it</span>
    <span class="num">128</span>,                     <span class="comment">// Reserve 128 bytes for our hook code</span>
    <span class="preproc">MEM_COMMIT</span> | <span class="preproc">MEM_RESERVE</span>,<span class="comment">// Allocate + commit memory in one go</span>
    <span class="preproc">PAGE_EXECUTE_READWRITE</span>   <span class="comment">// Make it readable, writable, and executable</span>
);
</div>

- hProcess → the handle we opened with OpenProcess.
- NULL → we’re not picky about the address; Windows picks a safe spot.
- 128 → just enough space for our trampoline code and a jump back.
- MEM_COMMIT & MEM_RESERVE → allocate and commit memory in one call.
- PAGE_EXECUTE_READWRITE → we need the memory to run code, so it must be executable.
- reserve → Pointer to our Allocated Memory

If you want to learn more beyond this breakdown: [MSDN VirtualAllocEx](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualallocex)

### Hooking the Z-Axis Instruction

First we have to find our z coordinate in cheat engine. We can do so by going up and down on a slope in game and increase and decrease just like 
what we did to find our view matrix but instead of looking up and down we make our player move up and down.  
After narrowing down our results we can lock the value in cheat engine to see which address makes it so that our player is locked in place (with 
respect to the Z-axis).

Now we can see which instructions access our Z-axis:
![ESP-Image1](/ViewProj-Blog/assets/images/part-5/ce_z_access.png)

I'm going to use the second instruction seen here just due to it being the most reliable one from testing.

Looking at the disassembly:
![ESP-Image1](/ViewProj-Blog/assets/images/part-5/ce_disassmbly_zaxis.png)

Let's start counting how many bytes we need. We require 12 bytes for a full, mov reg; jmp reg hook.  
Counting bytes we get:

![ESP-Image1](/ViewProj-Blog/assets/images/part-5/ce_disassmbly_zaxis - Copy.png)

**7+3+3 = 13;**  

So we have 12 bytes so we need to nop 1 byte to prevent spillover.

### Writing the Trampoline

Now that we have the essentials set up (Process ID, Module Base Address, Process Handle, and a pointer to our allocated memory from 
VirtualAllocEx stored in reserve), it’s time to actually build the trampoline.

The trampoline has two jobs:

1. Preserve the original instructions we overwrite when inserting our jump.
2. Redirect execution into our allocated memory, run our custom code, then “bounce” back just after the jump

**Step 1: Backing Up Overwritten Bytes**

When we inject a jump at the target instruction, we overwrite some of the game’s original instructions. If we don’t restore them later inside
our trampoline, the game will break because those instructions were never executed.

So, first we read and store those original bytes. In this case, we need to restore 13 bytes.

<div class="cpp-code"><span class="type">BYTE</span> <span class="var">stolenBytes</span>[<span class="num">13</span>];
<span class="fn">ReadProcessMemory</span>(
    <span class="var">hProcess</span>,
    (<span class="type">BYTE</span>* )<span class="var">baseAddr</span> + <span class="num">0x8743F5</span>,   <span class="comment">// target address = module base + RVA</span>
    <span class="var">stolenBytes</span>,
    <span class="fn">sizeof</span>(<span class="var">stolenBytes</span>),
    <span class="kw">nullptr</span>
);
</div>

Now we have a safe copy of the instructions we’re about to overwrite.

**Step 2: Writing the Jump**

We need to insert an absolute jump to our allocated trampoline. The standard 5-byte relative jump instruction (E9 rel32) is inadequate because VirtualAllocEx rarely allocates 
memory within the ±2GB range required for relative addressing. Therefore, to encode the full 64-bit absolute address, which won't fit in the initial 5 bytes, we must build the 
jump in two steps:

1. Move the trampoline address into a register. We use mov rax, <address> (10 bytes). This loads the pointer to our allocated memory into rax.
2. Jump via that register. Then we do jmp rax (2 bytes), which transfers execution to our trampoline.

Here’s the code that builds and writes that jump:

<div class="cpp-code"><span class="comment">// Build "mov rax, &lt;address&gt;"</span>
<span class="type">BYTE</span> <span class="var">movArray</span>[<span class="num">10</span>];
<span class="var">jmpArray</span>[<span class="num">0</span>] = <span class="num">0x48</span>; <span class="comment">// REX prefix for 64-bit</span>
<span class="var">jmpArray</span>[<span class="num">1</span>] = <span class="num">0xB8</span>; <span class="comment">// Opcode for "mov rax, imm64"</span>
<span class="fn">memcpy</span>(&amp;<span class="var">jmpArray</span>[<span class="num">2</span>], &amp;<span class="var">reserve</span>, <span class="fn">sizeof</span>(<span class="type">uintptr_t</span>)); <span class="comment">// Insert trampoline address</span>

<span class="comment">// Build "jmp rax"</span>
<span class="type">BYTE</span> <span class="var">jmpArray</span>[<span class="num">2</span>];
<span class="var">movArray</span>[<span class="num">0</span>] = <span class="num">0xFF</span>; <span class="comment">// Opcode group for near/indirect jumps</span>
<span class="var">movArray</span>[<span class="num">1</span>] = <span class="num">0xE0</span>; <span class="comment">// ModRM for "jmp rax"</span>

<span class="type">BYTE</span> <span class="var">nopArray</span> = { <span class="num">0x90</span> };

<span class="comment">// Write both parts into the target function</span>
<span class="fn">WriteProcessMemory</span>(
    <span class="var">hProcess</span>,
    (<span class="type">BYTE</span>* )<span class="var">baseAddr</span> + <span class="num">0x8743F5</span>, <span class="comment">// target instruction</span>
    <span class="var">movArray</span>,
    <span class="fn">sizeof</span>(<span class="var">movArray</span>),
    <span class="kw">nullptr</span>
);

<span class="fn">WriteProcessMemory</span>(
    <span class="var">hProcess</span>,
    (<span class="type">BYTE</span>* )<span class="var">baseAddr</span> + <span class="num">0x8743F5</span> + <span class="fn">sizeof</span>(<span class="var">movArray</span>), <span class="comment">// directly after mov rax</span>
    <span class="var">jmpArray</span>,
    <span class="fn">sizeof</span>(<span class="var">jmpArray</span>),
    <span class="kw">nullptr</span>
);

<span class="comment">// NOP the one spillover byte</span>
<span class="fn">WriteProcessMemory</span>(
    <span class="var">hProcess</span>,
    (<span class="type">BYTE</span>* )<span class="var">baseAddr</span> + <span class="num">0x8743F5</span> + <span class="fn">sizeof</span>(<span class="var">movArray</span>) + <span class="fn">sizeof</span>(<span class="var">jmpArray</span>), <span class="comment">// directly after jmp rax</span>
    <span class="var">nopArray</span>,
    <span class="fn">sizeof</span>(<span class="var">nopArray</span>),
    <span class="kw">nullptr</span>
); 
</div>

At this point, the original instruction at GhostOfTsushima.exe + 0x8743F5 no longer executes. Instead, execution now flows into our trampoline 
code (inside the memory block allocated with VirtualAllocEx).

### Capturing & Storing Entities

To actually track every entity that hits our hooked instruction, we need a place to temporarily stash its pointer before our C++ code picks it up.

We’ll keep all unique entities in an unordered_set:

<div class="cpp-code"><span class="type">std::unordered_set</span>&lt;<span class="type">uintptr_t</span>&gt; <span class="var">uniqueEntities</span>;
</div>

**Allocating space for the “current entity”**  
First, we reserve a single 8-byte slot inside the game process where our hook can write the most recent entity pointer:

<div class="cpp-code"><span class="var">entityAddr</span> = (<span class="type">uintptr_t</span>)<span class="fn">VirtualAllocEx</span>(
    <span class="var">hProcess</span>,
    <span class="kw">NULL</span>,
    <span class="fn">sizeof</span>(<span class="type">uintptr_t</span>),
    <span class="preproc">MEM_COMMIT</span> | <span class="preproc">MEM_RESERVE</span>,
    <span class="preproc">PAGE_EXECUTE_READWRITE</span>
);
</div>

This will let us store our entities in the temporarily stash.  

**Writing to the slot in assembly**
Inside the game, rcx holds the current entity pointer. So our injected shellcode looks like this:
```asm
mov r10, <entityAddr>  ; load our reserved address into r10
mov [r10], rcx         ; store rcx (entity pointer) into [entityAddr]
```
> ⚠️ Why not just do mov [entityAddr], rcx?  
Because x64 instructions can’t encode a full 64-bit absolute address directly. The only way is to load the address into a register 
(r10 here) and then store through that register.

**Shellcode & Stolen Bytes**

We build our shellcode then write our shellcode and the stolen bytes to our allocated memory:

<div class="cpp-code"><span class="comment">// build our payload</span>
<span class="type">BYTE</span> <span class="var">shellcode</span>[] = {
    <span class="num">0x49</span>, <span class="num">0xBA</span>,              <span class="comment">// mov r10, imm64</span>
    <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>,  <span class="comment">// placeholder for entityAddr</span>
    <span class="num">0x49</span>, <span class="num">0x89</span>, <span class="num">0x0A</span>         <span class="comment">// mov [r10], rcx</span>
};

<span class="comment">// add the allocated stash address to our payload</span>
<span class="fn">memcpy</span>(&amp;<span class="var">shellcode</span>[<span class="num">2</span>], &amp;<span class="var">entityAddr</span>, <span class="fn">sizeof</span>(<span class="type">uintptr_t</span>));

<span class="comment">// write our payload</span>
<span class="fn">WriteProcessMemory</span>(<span class="var">hProcess</span>, (<span class="type">BYTE</span>*)<span class="var">reserve</span>, <span class="var">shellcode</span>, <span class="fn">sizeof</span>(<span class="var">shellcode</span>), <span class="kw">nullptr</span>);

<span class="comment">// write the stolen bytes</span>
<span class="fn">WriteProcessMemory</span>(<span class="var">hProcess</span>, (<span class="type">BYTE</span>*)<span class="var">reserve</span> + <span class="fn">sizeof</span>(<span class="var">shellcode</span>), <span class="var">stolenBytes</span>, <span class="fn">sizeof</span>(<span class="var">stolenBytes</span>), <span class="kw">nullptr</span>);
</div>

At runtime, every time our hooked instruction is hit, the entity pointer in rcx gets stored at entityAddr.

**Jump Back**

We jump back to original instruction:

<div class="cpp-code"><span class="comment">// build our jmpBack</span>
<span class="type">BYTE</span> <span class="var">jmpBack</span>[] = {
    <span class="num">0x48</span>, <span class="num">0xB8</span>,              <span class="comment">// mov rax, imm64</span>
    <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>, <span class="num">0</span>,  <span class="comment">// placeholder address</span>
    <span class="num">0xFF</span>, <span class="num">0xE0</span> <span class="comment">// jmp rax</span>
};

<span class="comment">// calculate our return address (Our return address will be the original instruction address plus the 13 bytes we overwritten)</span>
<span class="var">jmpBackAddy</span> = <span class="var">baseAddress</span> + <span class="num">0x8743F5</span> + <span class="num">13</span>;

<span class="comment">// add our return address to our jmpBack</span>
<span class="fn">memcpy</span>(&amp;<span class="var">jmpBack</span>[<span class="num">2</span>], &amp;<span class="var">reserve</span>, <span class="fn">sizeof</span>(<span class="type">uintptr_t</span>));

<span class="comment">// write the jmpBack</span>
<span class="fn">WriteProcessMemory</span>(<span class="var">hProcess</span>, (<span class="type">BYTE</span>*)<span class="var">reserve</span> + <span class="fn">sizeof</span>(<span class="var">shellcode</span>) + <span class="fn">sizeof</span>(<span class="var">stolenBytes</span>), <span class="var">jmpBack</span>, <span class="fn">sizeof</span>(<span class="var">jmpBack</span>), <span class="kw">nullptr</span>);

</div>

Now after execution of our payload and the stolen bytes in our allocated memory it jumps back to our original instruction as to not break flow of execution.

**Reading entities back in C++**

Now, our external process polls the reserved memory region (entityAddr) in the target process to monitor for new entity pointers:

<div class="cpp-code"><span class="kw">while</span> (<span class="kw">true</span>)
{
    <span class="type">uintptr_t</span> <span class="var">curEntity</span> = <span class="num">0</span>;

    <span class="fn">ReadProcessMemory</span>(<span class="var">hProcess</span>, (<span class="type">BYTE</span>*)<span class="var">entityAddr</span>, &amp;<span class="var">curEntity</span>, <span class="fn">sizeof</span>(<span class="var">curEntity</span>), <span class="kw">nullptr</span>);

    <span class="kw">if</span> (<span class="var">curEntity</span> != <span class="num">0</span> &amp;&amp; <span class="var">uniqueEntities</span>.<span class="fn">find</span>(<span class="var">curEntity</span>) == <span class="var">uniqueEntities</span>.<span class="fn">end</span>())
    {
        <span class="var">uniqueEntities</span>.<span class="fn">insert</span>(<span class="var">curEntity</span>);
        <span class="fn">std::cout</span> &lt;&lt; <span class="const">"New unique entity found: "</span> &lt;&lt; <span class="fn">std::hex</span> &lt;&lt; <span class="var">curEntity</span> &lt;&lt; <span class="fn">std::endl</span>;
    }
}
</div>

*uniqueEntities.find(curEntity)* will return *uniqueEntities.end()* if no such data exits inside the unorderd map, then we simply insert it using 
*uniqueEntities.insert(curEntity)*

> You could also check if curEntity has valid health, position or some specific flag to ensure only valid entities are passed into our unordered_set.

This way, every time the Z-axis instruction executes for some entity, we capture it once and stash it in uniqueEntities. Later, we can loop 
through this set and use their positions for ESP drawing.


























