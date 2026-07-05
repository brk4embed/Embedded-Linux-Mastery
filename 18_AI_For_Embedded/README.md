# 18 — AI for Embedded Engineers

> 80+ prompt templates. Workflows that work. Real productivity gains from day one.

---

## The Core Premise

Most embedded engineers use AI like this:
```
"Hey ChatGPT, what does kmalloc do?"
```

Engineers who use AI as a true multiplier do this:
```
"I'm writing a DMA-coherent scatter-gather buffer manager for a custom PCIe 
NVMe controller. The hardware supports 64-entry SG lists, each entry can be 
up to 4MB, and the controller requires 4KB alignment for each entry. 

Here's my current struct: [code]

Issues I'm seeing: [specific symptoms]

Generate: (1) improved struct, (2) allocation helper with error handling, 
(3) map/unmap functions using dma_map_sg(), (4) unit test skeleton"
```

**Context + specifics → dramatically better outputs.**

---

## Prompt Library: Kernel Development

### Kernel API Discovery
```
I need to [SPECIFIC TASK] in a Linux kernel driver.
Platform: ARM64, kernel version 6.6
Relevant context: [subsystem, driver type, constraints]
What kernel API should I use? Show the header file and a minimal example.
```

### Driver Skeleton Generation
```
Write a Linux kernel platform driver for [PERIPHERAL NAME].
Requirements:
- DT binding: compatible = "[vendor],[name]"
- Resources: MMIO at [address], IRQ (level-triggered), clock "[name]"
- Features: [list what the driver does]
- Expose via: [sysfs/IIO/char dev/block dev]
- Use devm_ for all resources
- Include suspend/resume PM callbacks
- Module info: author = "Ravi Kumar Bokka", license = "GPL"
```

### Code Review
```
Review this kernel driver code for:
1. Memory leaks (kmalloc without kfree on all error paths)
2. NULL pointer dereferences
3. Missing IS_ERR() checks on ERR_PTR returns
4. Incorrect use of atomic vs sleeping locks
5. DMA coherency issues
6. Missing error codes (returning 0 on error)
7. Kernel coding style violations

[Paste code]
```

### Kernel Oops Analysis
```
Analyze this ARM64 kernel oops:
System: [board name], kernel [version], defconfig [name]
Occurred during: [what operation triggered it]

[Paste full oops including call trace and registers]

Provide:
1. What type of fault this is
2. Most likely root cause (3 hypotheses, ranked)
3. Step-by-step debug plan
4. What specific code to look at in the call trace
```

---

## Prompt Library: Boot and BSP

### Boot Failure Analysis
```
My embedded board (RK3588-based, Radxa Rock 5B+) shows this UART output
before hanging:

[paste last 30 lines of UART output]

Boot config:
- SPL/TPL: rockchip-tpl (rkbin version [X])
- U-Boot: [version]  
- Kernel: [version] + [defconfig]
- Storage: [eMMC / SD card]

What stage failed? What should I check?
List 5 possible causes ranked by likelihood.
```

### Device Tree Node Creation
```
I'm adding support for [PERIPHERAL] to my RK3588 device tree.
Datasheet reference: [registers at offset 0x..., IRQ line N, needs clock X]

Create a DT node following Linux kernel DT binding conventions:
1. The DT node itself (for the board .dts)
2. The DT binding YAML file (Documentation/devicetree/bindings/)
3. The corresponding driver's of_device_id .compatible string

Peripheral characteristics: [describe]
```

### U-Boot Debug
```
My U-Boot is failing to boot the kernel.
U-Boot output:
[paste output]

My bootcmd:
[paste bootcmd]

My bootargs:
[paste bootargs]

What is wrong? Provide the corrected bootcmd and bootargs.
```

---

## Prompt Library: Debugging

### ftrace Analysis
```
I captured this ftrace output while debugging [issue]:
[paste ftrace output]

I'm looking for: [what you're looking for — timing, call sequence, etc.]
The driver is [name], kernel version [X].

What does this trace reveal? What anomalies do you see?
```

### GDB Session Help
```
I'm debugging a kernel crash with KGDB. 
My GDB session:
[paste GDB output / register values / backtrace]

I've identified the crash point:
[paste relevant code section]

What is the root cause? How do I trace the null pointer or corrupted pointer?
```

### KASAN Report Analysis
```
I got a KASAN report:
[paste full KASAN report]

Interpret this report:
1. What type of memory bug is this?
2. What code path caused it?
3. What is the fix?
4. How to prevent this class of bug?
```

---

## Prompt Library: Code Review & Quality

### Commit Message Generation
```
Write a Linux kernel commit message for this change.
Rules: 50-char subject, blank line, 72-char body, Signed-off-by.

The change is:
- Driver: [name]
- Subsystem: [name]
- Change type: [bugfix / feature / cleanup / refactor]
- What changed: [describe]
- Why: [reason]
- Impact: [who is affected]
- Testing: [how tested]

Code diff:
[paste diff]
```

### Patch Review Checklist
```
Review this kernel patch for submission to LKML.
Check for:
1. checkpatch.pl violations (common ones: trailing whitespace, 
   80+ char lines, missing blank lines after declarations)
2. Sparse type annotation issues
3. Missing MODULE_DEVICE_TABLE
4. Incorrect error handling patterns
5. Missing devm_ usage opportunities
6. Locking issues
7. Documentation quality (Kconfig help text, DT binding)

[Paste patch]
```

---

## Prompt Library: AI + Embedded Systems

### NPU Model Optimization
```
I'm deploying [MODEL_NAME] on Radxa 5B+ RK3588 NPU (6 TOPS, RKNN API).
Current performance: [X] ms/inference at [batch size]
Target: [Y] ms/inference

Current RKNN export command: [command]
Current model architecture: [describe layers]

How can I optimize:
1. Quantization strategy (INT8 vs INT4 per layer)
2. Input/output tensor layout for NPU efficiency
3. Pre/post processing to move off NPU
```

### AI Agent Design
```
Design a LangGraph agent that:
- Task: [what the agent should do]
- Tools: [what tools it should have access to]
- Input: [what it receives]
- Output: [what it should produce]

The agent is for embedded Linux engineering. 
Show: state schema, node definitions, edge conditions, example invocation.
```

---

## Prompt Library: Interview Prep

### STAR Story Generation
```
Help me structure a STAR interview story from this experience:

Role: [my role]
Company/Client: [context]
Technical challenge: [what was difficult]
My specific actions: [what I did — be specific]
Technical decisions I made: [key choices]
Outcome: [what improved, what shipped, impact]

Generate: a 2-minute verbal STAR story + a 1-paragraph written version.
```

### Concept Explanation Practice
```
I need to explain [KERNEL CONCEPT] in an interview.
Question: "[typical interview question about this concept]"

Generate: 
1. A 30-second answer (if interviewer just wants quick answer)
2. A 2-minute detailed answer (if they ask to elaborate)
3. A follow-up question the interviewer might ask and how to answer it
```

---

## AI-Assisted Daily Workflow

```
Morning (20 min):
  1. Open VS Code + Continue.dev (local Ollama)
  2. Ask: "Explain the next section of [topic I'm studying] 
           at expert level. I already know [what I know]."
  3. Save useful explanation to a notes file

Work (continuous):
  Use Copilot for:
  - Tab completion on driver boilerplate
  - Inline edit for error handling patterns
  - Variable naming suggestions
  
  Use Claude for:
  - Complex architecture questions
  - Code review (paste function, ask for issues)
  - Debugging reasoning ("think through this oops")

Evening (30 min):
  1. Pick 5 interview questions from 21_Interview_Preparation/
  2. Write out answer without AI
  3. Ask Claude: "Is my answer to this question correct and complete? 
                  What's missing?"
  4. Refine answer based on feedback
```

---

*Related: [27_AI_Dev_Environment/](../27_AI_Dev_Environment/) | [19_AI_Agents/](../19_AI_Agents/) | [29_QEMU_Embedded_AI_Labs/](../29_QEMU_Embedded_AI_Labs/)*
