# 27 — AI Dev Environment

> How to set up and use AI coding tools to **10× your embedded Linux productivity**.  
> Tools covered: VS Code + Copilot + Agent Mode + Cursor + Claude + Amazon Q + local Ollama + Continue.dev

---

## Why This Section Exists

Most embedded engineers use AI tools like a search engine — type a question, get an answer.

**That's leaving 90% of the value on the table.**

AI tools, used correctly, can:
- Write boilerplate driver code in 30 seconds
- Explain any kernel function from source
- Debug kernel panics by analyzing oops output
- Generate test cases for your driver
- Review your patch for common mistakes
- Write device tree bindings from a datasheet spec
- Answer "how does X work" questions at expert level

This section shows you exactly how.

---

## Tool Stack Overview

```
┌─────────────────────────────────────────────────────────┐
│                    YOUR EDITOR                          │
│   VS Code + Neovim (both configured in this section)   │
├──────────────────────────┬──────────────────────────────┤
│    AI Assistance Tools   │     Code Intelligence        │
│  ─────────────────────   │  ──────────────────────────  │
│  GitHub Copilot          │  clangd (C/C++ LSP)          │
│  Claude (Projects)       │  ccls (alternative LSP)      │
│  Amazon Q Developer      │  ctags + cscope              │
│  ChatGPT                 │  Elixir Bootlin (web)        │
│  Continue.dev (local)    │  Sourcegraph (web)           │
├──────────────────────────┴──────────────────────────────┤
│              LOCAL AI (Offline)                         │
│   Ollama + tinyllama/codellama/deepseek-coder           │
│   llama.cpp (run on Radxa 5B+ ARM64!)                   │
└─────────────────────────────────────────────────────────┘
```

---

## Setup: VS Code for Embedded Linux Development

### Extensions to Install

```bash
# Install VS Code (if not already)
sudo snap install code --classic

# Then install these extensions (command palette or CLI):
code --install-extension ms-vscode.cpptools          # C/C++ IntelliSense
code --install-extension llvm-vs-code-extensions.vscode-clangd  # clangd LSP
code --install-extension GitHub.copilot              # GitHub Copilot
code --install-extension GitHub.copilot-chat         # Copilot Chat
code --install-extension amazonwebservices.amazon-q-vscode  # Amazon Q
code --install-extension Continue.continue           # Continue.dev (local AI)
code --install-extension mhutchie.git-graph          # Git visualization
code --install-extension GitLab.gitlab-workflow      # GitLab/Gerrit
code --install-extension ms-vscode-remote.remote-ssh  # Remote SSH
code --install-extension ms-vscode.hexeditor         # Binary file editor
code --install-extension ms-vscode-remote.remote-containers  # Dev containers
```

### clangd Setup for Kernel Source

clangd needs a `compile_commands.json` to understand kernel source:

```bash
# In your kernel source directory:
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig

# Generate compile_commands.json
bear -- make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
# OR use kernel's own script:
python3 scripts/clang-tools/gen_compile_commands.py

# Now VS Code + clangd understands the kernel
# - Ctrl+click on functions works
# - Go to definition works across the entire kernel
# - Hover shows documentation
# - Auto-complete for kernel APIs
```

### VS Code Settings for Kernel Work

```json
// .vscode/settings.json (in your kernel directory)
{
    "clangd.arguments": [
        "--background-index",
        "--suggest-missing-includes",
        "--cross-file-rename",
        "--header-insertion=never",
        "--compile-commands-dir=${workspaceFolder}"
    ],
    "C_Cpp.intelliSenseEngine": "disabled",  // disable MS IntelliSense, use clangd
    "editor.tabSize": 8,                      // kernel uses 8-space tabs
    "editor.insertSpaces": false,             // real tabs
    "files.trimTrailingWhitespace": true,
    "editor.rulers": [80],                    // kernel 80-char limit line
    "files.associations": {
        "*.dts": "dts",
        "Kconfig": "kconfig",
        "Makefile": "makefile"
    },
    "terminal.integrated.env.linux": {
        "ARCH": "arm64",
        "CROSS_COMPILE": "aarch64-linux-gnu-"
    }
}
```

---

## GitHub Copilot: The Right Way

### What Copilot is Good At (Use Aggressively)

```c
/* 1. Boilerplate driver code */
// Type this comment and let Copilot fill the struct:
/* Platform driver for RAVI custom DMA controller */
// Copilot will suggest the full platform_driver struct

/* 2. Repetitive code patterns */
// Start typing the first register read, Copilot completes the pattern:
val = readl(base + REG_STATUS);
// Copilot predicts next few registers from context

/* 3. Error handling patterns */
// Type the function call:
ret = devm_clk_get(&pdev->dev, "core");
// Copilot suggests the correct error check pattern

/* 4. Completing known patterns */
// Type:
static const struct of_device_id
// Copilot suggests the full table based on your driver name/compatible

/* 5. Test case generation */
// Comment: "Write a unit test for ufshcd_prepare_uic_cmd"
// Copilot generates a reasonable test skeleton
```

### Copilot Chat Prompts for Embedded Work

**Kernel Oops Analysis:**
```
I have this kernel oops:
[paste oops output]

The system is ARM64, kernel 6.6, RK3588 platform.
What are the likely causes? What should I check first?
```

**Device Tree Review:**
```
Review this device tree node for a UFS controller:
[paste DT node]

Check for:
1. Missing required properties
2. Incorrect property types
3. Cell size mismatches
4. Missing clocks or regulators
```

**Driver Code Review:**
```
Review this kernel driver probe function for:
1. Error handling issues
2. Memory leaks
3. Missing devm_ wrappers
4. Missing error codes
5. Coding style violations

[paste probe function]
```

**API Questions:**
```
I need to allocate DMA-coherent memory in a kernel driver.
The device is on the AXI bus and doesn't support IOMMU.
What's the correct API? Show example with proper error handling.
```

### Copilot Agent Mode (VS Code)

Agent Mode (Chat panel → select "Agent" model) lets Copilot read/write multiple files:

```
Prompt for driver writing:
"Create a platform driver for an I2C temperature sensor with:
- DT binding with 'ravi,temp-sensor' compatible
- Read temperature via I2C at address 0x48, register 0x00
- Expose via IIO subsystem (iio_dev, iio_channel)
- sysfs attribute: /sys/bus/iio/devices/iio:device0/in_temp_input
- Proper devm_ allocation throughout
- Module parameters: poll_interval (default 1000ms)

Create: temp_sensor.c, temp_sensor.h, DT binding YAML, Makefile, Kconfig"
```

Agent Mode will create all these files with reasonable content.

---

## Claude for Deep Technical Work

Claude excels at:
- Explaining complex kernel subsystems (better than GPT-4 for kernel)
- Long context analysis (upload full kernel files)
- Architecture discussions
- Debugging reasoning ("think through this panic")

### Claude Projects Setup

1. Go to claude.ai → Create a new Project
2. Add these files as project knowledge:
   - Your driver under development
   - Relevant kernel headers (`include/linux/blkdev.h`, etc.)
   - The subsystem's core file (`drivers/scsi/scsi.c`)
   - Device datasheet sections (relevant registers)

3. Now every conversation has this context:

```
Prompt: "Based on the UFSHCI spec in my project knowledge and the kernel
driver code, why does ufshcd_clear_doorbell_regs() need to be called
before ufshcd_send_command()? Trace the doorbell bit lifecycle."
```

### Claude for Boot Debugging

```
I'm debugging a boot failure. The UART output is:
[paste UART output]

Board: RK3588-based custom board
Boot stage: SPL has initialized DDR, loading U-Boot
The system hangs after "Trying to boot from eMMC"

What are the likely causes?
What rkdeveloptool commands should I run?
What oscilloscope probes should I check?
```

---

## Amazon Q Developer for AWS/Cloud Integration

If your project involves AWS IoT, Greengrass, or cloud integration:

```bash
# Install Amazon Q CLI
curl "https://desktop-release.q.us-east-1.amazonaws.com/latest/linux/x86_64/amazon-q" -o amazon-q
chmod +x amazon-q && sudo mv amazon-q /usr/local/bin/

# In VS Code: Amazon Q extension is already installed
# Sign in with AWS Builder ID (free)
```

**Best use cases for embedded engineers:**
- Greengrass component development
- AWS IoT device shadow integration
- Lambda functions for telemetry processing
- CloudWatch log analysis for device fleet

```
Amazon Q prompt: "Write an AWS IoT Greengrass v2 component that:
- Reads temperature from /sys/class/thermal/thermal_zone0/temp
- Publishes to MQTT topic 'devices/{device_id}/temperature' every 30s
- Handles reconnection automatically
- Runs as a systemd service"
```

---

## Local AI with Ollama (Offline, Privacy-First)

Crucial when working with:
- Proprietary/NDA code
- Client code you can't share
- Offline environments

### Setup

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull models
ollama pull tinyllama      # 669MB — fast, good for quick Q&A
ollama pull codellama      # 3.8GB — better code generation
ollama pull deepseek-coder # 776MB — strong code model, small size
ollama pull llama3.2       # 2GB — good general + code

# Run directly
ollama run codellama
# >>> Write a Linux kernel platform driver for I2C...

# REST API (for integration)
curl http://localhost:11434/api/generate -d '{
  "model": "codellama",
  "prompt": "Write a kernel module that prints hello world",
  "stream": false
}'
```

### Continue.dev — Local AI in VS Code

```json
// ~/.continue/config.json
{
  "models": [
    {
      "title": "CodeLlama Local",
      "provider": "ollama",
      "model": "codellama",
      "apiBase": "http://localhost:11434"
    },
    {
      "title": "Claude 3.5 Sonnet",
      "provider": "anthropic",
      "model": "claude-3-5-sonnet-20241022",
      "apiKey": "YOUR_API_KEY"
    }
  ],
  "tabAutocompleteModel": {
    "title": "Tab Autocomplete",
    "provider": "ollama", 
    "model": "deepseek-coder"
  },
  "contextProviders": [
    {"name": "diff", "params": {}},
    {"name": "open", "params": {}},
    {"name": "codebase", "params": {}}
  ]
}
```

**Using Continue.dev:**
- `Ctrl+I` — inline edit selected code
- `Ctrl+L` — open chat panel
- `@codebase` — search your entire codebase
- `@open` — reference current open files
- `@diff` — reference current git diff

---

## Running LLMs on Radxa 5B+ (ARM64)

Your Radxa 5B+ is a perfect local AI inference machine:

```bash
# On Radxa 5B+ (ARM64, 16GB RAM recommended)

# Install llama.cpp
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make -j4  # ARM64 optimized build automatically

# Download a quantized model
# From huggingface:
wget https://huggingface.co/TheBloke/Llama-3-8B-GGUF/resolve/main/Llama-3-8B.Q4_K_M.gguf

# Run
./llama-cli -m Llama-3-8B.Q4_K_M.gguf -p "Explain Linux kernel platform drivers" -n 200

# Serve as API (then VS Code can use it)
./llama-server -m Llama-3-8B.Q4_K_M.gguf --port 8080

# Use RKNN NPU for inference (2× faster on RK3588)
# See: 23_Radxa_5B_Plus_Labs/ for NPU inference setup
```

**Performance on Radxa 5B+:**
| Model | Size | Tokens/sec | Quality |
|-------|------|-----------|---------|
| TinyLlama Q4 | 669MB | ~25 t/s | Basic Q&A |
| Llama-3.2-3B Q4 | 1.9GB | ~12 t/s | Good code |
| Llama-3-8B Q4 | 4.7GB | ~5 t/s | Excellent |
| Llama-3-8B Q4 + NPU | 4.7GB | ~10 t/s | Excellent |

---

## The AI-Assisted Driver Development Workflow

```
Step 1: Understand the hardware
  → Ask Claude: "Explain [PERIPHERAL] and how Linux drivers interface it.
                 Reference: include/linux/[subsystem].h"

Step 2: Generate skeleton
  → Copilot Agent Mode: "Create platform driver skeleton for [peripheral]
                          with these requirements: [list]"

Step 3: Fill in hardware-specific code
  → You write: register definitions from datasheet
  → Copilot completes: read/write patterns, error handling

Step 4: Device tree binding
  → Ask Claude: "Generate DT binding YAML for [driver] following
                 Documentation/devicetree/bindings/[subsystem]/ conventions"

Step 5: Test
  → Ask Copilot: "Generate test cases for probe(), read(), write() in my driver"

Step 6: Review
  → Ask Claude: "Review this driver for: memory leaks, error path issues,
                 locking correctness, kernel coding style violations"

Step 7: Commit message
  → Ask Claude: "Write a kernel commit message for this driver following
                 the kernel contribution guidelines. The change is: [describe]"
```

---

## Prompt Templates Library

Save these in your prompt library (Cursor has built-in prompt management):

```markdown
## [KERNEL_OOPS] Analyze kernel oops
Analyze this ARM64 kernel oops. System: [board], kernel [version].
Identify: fault type, faulting instruction, root cause hypothesis, debug steps.
[PASTE OOPS]

## [DRIVER_REVIEW] Review driver code
Review for: NULL pointer dereference risks, memory leaks, error path coverage,
devm_ usage, locking correctness, DMA coherency issues.
[PASTE CODE]

## [DT_BINDING] Create DT binding
Create a DT binding YAML for [driver] following Linux kernel conventions.
Required properties: [list]. Optional: [list]. Example node: [show one].

## [PROBE_SKELETON] Platform driver probe
Write probe() for platform driver [name]. Requirements:
- devm_ for all allocations
- clock: [name], rate: [freq]
- IRQ: level-triggered
- Sysfs attribute: [attr_name]
- Error path: complete cleanup

## [COMMIT_MSG] Kernel commit message
Write a kernel commit message for this [driver/fix/feature].
Change: [description]. Impact: [who is affected]. Testing: [how tested].
Follow: 50-char subject, blank line, 72-char-wrapped body.
```

---

## Interview Questions

**Beginner:**
- What is clangd and why is it better than the standard C IntelliSense for kernel work?
- How does Copilot tab completion differ from Copilot Chat?
- What is Continue.dev and when would you use it over Copilot?

**Intermediate:**
- How do you set up VS Code to understand the Linux kernel source code (navigate to definitions, etc.)?
- What are the privacy implications of using cloud AI tools with embedded code? How do you mitigate them?
- Compare GitHub Copilot, Claude, and Amazon Q for embedded Linux work. When would you use each?

**Advanced:**
- How would you create a Claude Project that understands your company's BSP code? What would you include in the project knowledge?
- Design an AI-assisted workflow for bringing up a new peripheral driver from datasheet to upstream patch.
- How would you run a local LLM on Radxa 5B+ using the NPU for acceleration? What are the trade-offs vs CPU inference?

---

*Related: [18_AI_For_Embedded/](../18_AI_For_Embedded/) | [19_AI_Agents/](../19_AI_Agents/) | [29_QEMU_Embedded_AI_Labs/](../29_QEMU_Embedded_AI_Labs/)*
