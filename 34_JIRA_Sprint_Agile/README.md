# 34 — JIRA, Sprint & Agile Tools for Embedded Engineers

> **Your Context:** In consulting companies (Thundersoft/Qualcomm, Eximietas/ARM ADC, Amazon), you likely use JIRA daily for bug tracking and sprint planning. But many engineers use it passively — just updating ticket status. This section teaches you to use it **proactively** to manage your work, demonstrate your value, and organize your learning.

---

## Why JIRA Matters to YOU Specifically

1. **Visibility** — Your work as a contractor/consultant is only as visible as your JIRA updates. Writing good tickets = getting credit.
2. **Job security** — Managers judge impact by JIRA throughput. Know how to game it productively.
3. **Freelancing** — When you run your own consulting projects, you'll use JIRA/Linear/GitHub Issues to manage client work.
4. **Learning tracker** — You can create a personal JIRA project to track your mastery of this curriculum.

---

## Section Structure

```
34_JIRA_Sprint_Agile/
├── 01_Agile_For_Embedded.md          ← What Agile means in embedded teams
├── 02_JIRA_Basics.md                 ← Project types, issue types, workflows
├── 03_Sprint_Planning.md             ← How sprints work, velocity, story points
├── 04_Writing_Good_Bug_Reports.md    ← The skill that makes you indispensable
├── 05_JIRA_For_Freelancers.md        ← Trello / Linear / GitHub Issues as alternatives
├── 06_Personal_Kanban.md             ← Managing YOUR learning as a kanban board
└── 07_Estimation_Skills.md           ← Story points and time estimation for embedded work
```

---

## Part 1: Agile Basics for Embedded Engineers

### What Is Agile? (Plain English)

Traditional waterfall (what many embedded projects used to do):
```
Requirements → Design → Code → Test → Ship
(6-12 months per cycle, big-bang release)
```

Agile (what modern embedded teams do):
```
Week 1-2 (Sprint 1): Add UART driver → test → demo
Week 3-4 (Sprint 2): Add I2C driver → test → demo
Week 5-6 (Sprint 3): Add SPI driver → test → demo
(2-week cycles, working software every sprint)
```

**For embedded:** Agile is adapted because hardware has longer lead times. You can't always have "shippable hardware" every 2 weeks. But you CAN:
- QEMU-first development (your specialty!) — QEMU works in sprint 1, hardware in sprint 4
- Split: firmware team runs 2-week sprints; HW team runs 4-week cycles

### Common Agile Terms

| Term | Meaning | Embedded Example |
|------|---------|------------------|
| **Sprint** | 2-week development cycle | "Sprint 23: UFS driver bring-up on SC7280" |
| **Backlog** | All work to be done (unordered) | Full list of features/bugs |
| **Story Point** | Relative effort estimate | 1=trivial, 3=half day, 5=1 day, 8=2 days, 13=week |
| **Velocity** | Average story points per sprint | "Our team does 40 points/sprint" |
| **Standup** | 15-min daily meeting | What did I do? What will I do? Any blockers? |
| **Retrospective** | End-of-sprint: what can we improve? | Process improvement meeting |
| **Epic** | Large feature spanning multiple sprints | "UFS 4.0 full driver support" |
| **Story** | User-visible feature | "As a user, I can mount a UFS device" |
| **Task** | Technical sub-work | "Implement UFSHCI register map" |
| **Bug** | Defect | "Kernel panic on UFS reset" |

---

## Part 2: JIRA Fundamentals

### Issue Types Hierarchy

```
Epic
└── Story (user-facing feature)
    ├── Task (technical work)
    ├── Sub-task (smaller pieces)
    └── Bug (defect)
```

### JIRA Workflow (typical embedded team)

```
To Do → In Progress → In Review → Done
         ↓               ↓
      Blocked         Needs Rework
```

### Creating Effective JIRA Tickets

**Bad ticket (common in embedded teams):**
```
Title: UFS not working
Description: UFS driver has issue
Priority: High
```

**Good ticket (what impresses managers and clients):**
```
Title: [SC7280] UFS UFSHCI controller fails to initialize after suspend/resume

Description:
Environment:
  - Platform: SC7280 (Snapdragon 7c Gen 2)
  - Kernel: 5.15.78-qualcomm-chromeos
  - Board: Herobrine Chromebook (reference board)
  - Coreboot: r4.20-123-gabcdef

Steps to Reproduce:
  1. Boot SC7280 board
  2. Run: echo mem > /sys/power/state
  3. Resume board (press key)
  4. Run: lsblk

Expected Result:
  /dev/sda visible, UFS device accessible

Actual Result:
  dmesg shows: "ufshcd: controller initialization failed"
  /dev/sda not present after resume

Kernel Log:
  [  123.456] ufshcd 4804000.ufs: ufshcd_hba_enable failed -110
  [  123.457] ufshcd 4804000.ufs: ufshcd_init: Register UFS host controller failed

Root Cause Analysis:
  Likely: PMIC/clock not re-enabled before UFS controller init on resume path
  Suspect: ufshcd_suspend/resume in drivers/ufs/host/ufshcd.c

Workaround: None known. Reboot required.

Acceptance Criteria:
  - UFS accessible after suspend/resume cycle
  - 100 suspend/resume cycles without failure
  - No regression on fresh boot

Labels: ufs, suspend-resume, sc7280, regression
```

### JIRA JQL — Search Like a Pro

JQL (JIRA Query Language) lets you search and filter issues:

```sql
-- All high-priority bugs assigned to you
assignee = currentUser() AND priority = High AND type = Bug

-- All open items in current sprint
sprint in openSprints() AND status != Done

-- Your team's recent activity
project = BSP AND updated > -7d ORDER BY updated DESC

-- Find all UFS-related bugs
project = BSP AND labels = ufs AND status != Done

-- Bugs created this sprint not yet in progress
project = BSP AND sprint in openSprints() 
  AND status = "To Do" AND type = Bug

-- Your closed items this month (shows your output)
assignee = currentUser() AND status = Done 
  AND resolved > startOfMonth()
```

---

## Part 3: Sprint Planning

### How a Sprint Works (Step by Step)

```
Week 0 (Sprint Planning, 2 hours):
  1. Product Owner presents backlog items in priority order
  2. Team estimates each item in story points
  3. Team selects items that fit in the sprint (based on velocity)
  4. Each item broken into tasks, assigned to engineers

Week 1-2 (Sprint Execution):
  - Daily standup (15 min): What did I do? What will I do? Blockers?
  - Engineers work on tasks, update JIRA status
  - Blocked items escalated immediately

Week 2 Day 10 (Sprint Review, 1 hour):
  - Demo completed work to stakeholders
  - Only DONE items count (not "90% done")

Week 2 Day 10 (Retrospective, 30 min):
  - What went well? (keep doing)
  - What went poorly? (stop doing / improve)
  - What should we try? (experiment)
```

### Story Point Estimation for Embedded Work

```
1 point   = < 2 hours
  Examples: Add a printk statement, fix a typo in DT, update Kconfig

3 points  = half day
  Examples: Fix a probe failure (root cause known), add a DT entry,
            write a simple sysfs attribute

5 points  = 1 full day
  Examples: Write a new I2C driver from scratch (simple device),
            port a driver to new platform (well-documented)

8 points  = 2 days
  Examples: Debug a suspend/resume issue (root cause unknown),
            write a platform driver with DMA + IRQ

13 points = 1 week
  Examples: Full UFS error injection test suite, CoreBSP TrustZone
            component bring-up on new chipset

21+ points = Too large, needs splitting
```

### Your Daily Standup Format

```
Yesterday:
  - Fixed SC7280 UART clock initialization (ticket BSP-123)
  - Reviewed Ravi's QFPROM patch series, sent review feedback

Today:
  - Start UFS controller resume path fix (ticket BSP-145)
  - Build test on SC7280 + SC7180

Blockers:
  - Need hardware access to SC7280 EVB — requesting from HW lab (BSP-145)
```

---

## Part 4: Writing World-Class Bug Reports

### The Anatomy of a Perfect Embedded Bug Report

```markdown
## Title
[Platform] [Component] [Brief description of symptom]
e.g.: [SC7280] [UFS] Kernel panic in ufshcd_abort() during error recovery

## Environment
| Field | Value |
|-------|-------|
| Platform | Qualcomm SC7280 |
| Kernel version | 5.15.78-cros |
| Coreboot | 4.20-r123 |
| Reproduciblity | 100% on EVB, 50% on Chromebook |

## Steps to Reproduce
1. Boot board with UFS as root device
2. Trigger error: echo 0 > /sys/bus/platform/.../reset
3. Wait 30 seconds

## Expected vs Actual
Expected: UFS recovers gracefully via error recovery
Actual: Kernel panic in ufshcd_abort()

## Kernel Log
```
[  45.123] ufshcd 4804000.ufs: Abort on outstanding tag 0
[  45.124] BUG: kernel NULL pointer dereference
[  45.125] PC is at ufshcd_abort+0x1f4/0x3a0
[  45.126] Call trace:
[  45.126]  ufshcd_abort+0x1f4/0x3a0
[  45.127]  scsi_abort_command+0x94/0xe0
```

## Root Cause (if known)
lhreq is NULL when abort is called during teardown phase.
Patch: check for NULL before dereference in ufshcd_abort()

## Fix / Workaround
Patch attached. Workaround: disable SCSI error recovery with scsi_timeout=0
```

---

## Part 5: Alternatives to JIRA (for Freelancing)

When you run your own consulting projects, you don't need JIRA's complexity:

| Tool | Cost | Best For |
|------|------|---------|
| **GitHub Issues** | Free | Open source projects, small client work |
| **Linear** | Free/paid | Fast, modern, developer-friendly |
| **Trello** | Free | Visual kanban, simple projects |
| **Notion** | Free/paid | Combined docs + tasks |
| **GitLab Issues** | Free | If using GitLab for code |

### Personal Learning Kanban with GitHub Projects

```
Create a GitHub Project board for this curriculum:

Backlog → In Progress → Done

Add cards:
[ ] Complete 06_Linux_Kernel section      (Not started)
[→] Practice vimdiff on kernel tree       (In Progress)
[✓] Push Embedded-Linux-Mastery to GitHub  (Done)
```

---

## Part 6: Use JIRA to Track YOUR Learning

Create a personal JIRA project (free tier) or GitHub Project to track this curriculum:

```
Project: EMBEDDED-MASTERY

Epics:
- EPICS-01: Debugging Tools Mastery
- EPICS-02: Kernel Development Skills
- EPICS-03: Open Source Contributions
- EPICS-04: AI Productivity Tools
- EPICS-05: Freelancing Pipeline

Sprint 1 (Week 1-2):
- [5pts] Read + practice 33_Code_Diff_Merge_Tools
- [5pts] Set up vimdiff + meld on dev machine
- [3pts] Practice quilt on coreboot tree
- [5pts] Write first blog post: "quilt for Coreboot patch management"

Sprint 2 (Week 3-4):
- [8pts] Complete 10_Code_Browsing ctags deep dive
- [5pts] Set up ctags/cscope on Linux kernel tree
- [3pts] Practice git bisect on a real regression
```

**This approach turns your learning into measurable, trackable work — exactly how your manager sees your professional work.**

---

## Quick Reference

```bash
# JIRA CLI (install: npm install -g jira-cli or use official jira CLI)
jira issue list --project BSP --assignee me
jira issue view BSP-123
jira issue transition BSP-123 "In Progress"
jira sprint list --project BSP

# GitHub Issues from CLI (install: gh)
gh issue list --assignee @me
gh issue create --title "Fix UFS suspend/resume" --label "bug,ufs"
gh issue view 42
gh issue close 42

# Create a project board
gh project create --title "Embedded Mastery Curriculum"
```
