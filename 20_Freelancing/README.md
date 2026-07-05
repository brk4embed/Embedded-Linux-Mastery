# 20 — Freelancing & Consulting

> Your embedded Linux expertise is worth $150–300/hr to the right clients. Here's how to find them and get paid.

---

## Section Structure

```
20_Freelancing/
├── 01_Market_Positioning.md       ← Your unique value + how to price
├── 02_Finding_Clients.md          ← Where embedded consulting work is
├── 03_Proposal_Writing.md         ← Winning technical proposals
├── 04_Project_Scoping.md          ← How to scope and price projects
├── 05_Contracts_And_Payment.md    ← Legal basics, invoicing
├── 06_Portfolio_Building.md       ← GitHub + LinkedIn strategy
└── 07_Sample_Proposals/
    ├── UFS_Driver_Port_Proposal.md
    ├── QEMU_Device_Model_Proposal.md
    └── BSP_Bring_Up_Proposal.md
```

---

## Why Embedded Consulting Is Lucrative

```
Supply/demand mismatch:
  - Semiconductor startups: need embedded expertise but can't afford full team
  - IoT companies: need Linux BSP but not full-time
  - Automotive: safety-critical, need experienced engineers
  - Defense/Aerospace: cleared engineers rare
  - Hardware startups: need firmware fast, before hardware is ready

What they'll pay:
  BSP bring-up: $5,000–20,000 per project
  QEMU device model: $8,000–25,000 per project
  Driver development: $3,000–15,000 per driver
  AI/NPU integration: $10,000–40,000 per project
  Retainer: $5,000–10,000/month for ongoing support

Your strongest areas:
  1. QEMU device modeling (DW-UFS 4.0 → rare expertise)
  2. Pre-silicon validation with AI
  3. Qualcomm platform BSP (QTEE, Coreboot)
  4. Linux kernel upstream (demonstrable quality)
```

---

## Finding Clients

### High-Value Channels

```
1. LinkedIn (highest ROI for embedded consultants)
   - Post: "I just built a complete QEMU device model for UFS 4.0.
             Here's how it works: [Mermaid diagram + 3 key insights]"
   - Post: "DDR training failure: how I debugged it with just a UART"
   - Post: "5 kernel KASAN bugs I caught before tapeout"
   → These attract CTOs and lead engineers at semiconductor companies

2. Open Source Visibility
   - Your LKML patches: https://lore.kernel.org/?q=Ravi+Kumar+Bokka
   - Your Coreboot patches: https://review.coreboot.org
   - Companies recruiting from upstream contributors

3. Conference Talks (best long-term ROI)
   - Submit to: Embedded Linux Conference (ELCE), Linux Plumbers, Open Source Summit
   - Topic: "AI-Driven Pre-Silicon Validation with QEMU"
   - One accepted talk → 10+ consulting inquiries

4. GitHub Portfolio
   - UFS 4.0 QEMU model (public, documented)
   - AI pre-silicon validation platform
   - Open-source BSP tools

5. Niche Forums
   - LKML (be helpful, visible)
   - Bootlin community
   - Rockchip/Qualcomm developer forums
```

---

## Sample Proposal: QEMU Device Model

```markdown
# Technical Proposal: Custom QEMU Device Model
**Client:** [Company Name]  
**Prepared by:** Ravi Kumar Bokka | brk4embed@gmail.com  
**Date:** [Date]

---

## Understanding of Requirements

[Company] requires a QEMU device model of the [PERIPHERAL NAME] to enable:
- Software team to begin driver development 6–12 months before silicon
- Pre-silicon validation of driver correctness
- CI/CD integration for automated regression testing

---

## Proposed Solution

### Phase 1: Requirements Analysis (Week 1)
- Review IP specification and register map
- Identify minimum viable feature set for driver bring-up
- Define test scenarios with your software team
- Deliverable: Architecture document + test plan

### Phase 2: MMIO Register Model (Weeks 2–3)
- Implement all registers with read/write semantics
- Implement W1C (write-1-to-clear) IRQ status registers
- Validate against register spec
- Deliverable: Functional register model, passes basic MMIO test

### Phase 3: DMA Engine Model (Weeks 4–5)
- Implement descriptor ring processing
- Guest-to-host DMA using qemu_dma_read/write API
- IRQ delivery on completion
- Deliverable: DMA transfers work end-to-end

### Phase 4: Protocol Emulation (Weeks 6–8)
- [Protocol-specific: UFS UPIU / PCIe TLP / etc.]
- Response generation matching spec behavior
- Error injection capability
- Deliverable: Complete protocol stack, Linux driver probes successfully

### Phase 5: Validation & Handoff (Weeks 9–10)
- Run Linux driver test suite in QEMU
- Fix model bugs found by testing
- Documentation: model architecture, known limitations, test results
- Deliverable: Validated model + complete documentation

---

## Credentials

**Relevant Experience:**
- Built DW-UFS 4.0 QEMU device model for ARM ADC (current role)
  - UFSHCI 4.0 register model, UPIU state machine, SCSI handling
- 11+ years embedded Linux: Qualcomm, ARM, Google ChromeOS, Amazon
- Upstream Linux kernel contributor (QFPROM driver: kernel.org)
- Upstream Coreboot contributor (SC7180/SC7280 ChromeOS)

---

## Investment

| Phase | Duration | Fixed Price |
|-------|---------|-------------|
| Phase 1 | 1 week | ₹30,000 |
| Phases 2–3 | 3 weeks | ₹90,000 |
| Phases 4–5 | 4 weeks | ₹1,20,000 |
| **Total** | **8 weeks** | **₹2,40,000** |

*International rates available on request. USD/EUR pricing for US/EU clients.*

**Payment terms:** 30% upfront, 40% at Phase 3 completion, 30% at delivery.

---

## What You Receive

- Complete QEMU device model (C source, mergeable upstream)
- DT binding and kernel device node for the model
- Test harness with documented test cases
- Architecture documentation (Mermaid diagrams)
- 30 days post-delivery support for bug fixes
```

---

## Market Rates (2025, India + Global)

| Service | India Rate | International Rate |
|---------|-----------|-------------------|
| QEMU device model | ₹2–5L per project | $5,000–15,000 |
| BSP bring-up (new SoC) | ₹3–8L per project | $8,000–25,000 |
| Driver development (1 driver) | ₹1–3L per driver | $3,000–10,000 |
| AI/NPU integration | ₹4–10L per project | $10,000–30,000 |
| Monthly retainer | ₹50K–1.5L/month | $3,000–8,000/month |
| Technical consulting (hourly) | ₹3,000–8,000/hr | $100–300/hr |

**How to position for maximum rate:**
1. QEMU + AI = rare combination → always command premium
2. Upstream kernel contributions → demonstrable quality → less risk for client
3. Specific domain (UFS, TrustZone, Coreboot) → can't be found on Fiverr
4. 11+ years = senior expertise → don't compete on price

---

*Related: [00_Career_Roadmap/](../00_Career_Roadmap/) | [25_20_Year_Career_Survival_Plan/](../25_20_Year_Career_Survival_Plan/) | [24_Industry_Project_Portfolio/](../24_Industry_Project_Portfolio/)*
