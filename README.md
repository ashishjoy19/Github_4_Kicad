# Git workflow for electronics development
### A practical tutorial for KiCad hardware projects

> Written for the **nexus-node** project (AVR128DB32 RS485 stepper controller)  
> but applicable to any KiCad or open hardware project.

---

## Table of contents

1. [Why git for hardware?](#1-why-git-for-hardware)
2. [Core concepts](#2-core-concepts)
3. [Repository structure](#3-repository-structure)
4. [The .gitignore file for KiCad](#4-the-gitignore-file-for-kicad)
5. [Branch strategy](#5-branch-strategy)
6. [Daily workflow — step by step](#6-daily-workflow--step-by-step)
7. [Version tagging and releases](#7-version-tagging-and-releases)
8. [Handling merge conflicts](#8-handling-merge-conflicts)
9. [Working with GitHub and GitLab together](#9-working-with-github-and-gitlab-together)
10. [Quick reference card](#10-quick-reference-card)

---

## 1. Why git for hardware?

Software developers have used git for decades. Hardware developers are often slower to adopt it, but the benefits are just as real — arguably more so, because a bad PCB revision costs real money to manufacture.

Git gives you:

- **Full history** of every schematic and layout change. You can always go back and see exactly what changed between v2.1 and v2.2, and why.
- **Safe experimentation.** Try a new layout approach on a branch. If it doesn't work, delete the branch. Nothing is lost.
- **Traceability.** When a board comes back from the fab and something is wrong, you can check out the exact commit that generated those gerbers and reproduce the problem.
- **Collaboration.** A teammate can work on firmware while you work on the PCB layout without either of you overwriting the other's work.
- **Release snapshots.** Every manufactured version of your board is tagged permanently. In two years you can reproduce v2.1 exactly, even if v3.0 is completely different.

---

## 2. Core concepts

Before the workflow makes sense, these five concepts need to be solid.

### Commit

A commit is a snapshot of your files at a point in time, with a message explaining what changed. Think of it as a save point in a game — you can always return to any save point.

```
commit a3f9c2d
Author: Your Name <you@email.com>
Date:   Mon May 14 2025

    hw: move RS485 connector to bottom edge
```

Every commit has a unique ID (the hash like `a3f9c2d`). You never need to type the full hash — the first 7 characters are enough.

### Branch

A branch is a parallel line of development. The default branch is `main`. When you create a new branch, you get an independent copy of the project where you can make changes without affecting `main`.

```
main:     A --- B --- C
                       \
hw/v2.2:               D --- E --- F
```

Commits D, E, F happen on the `hw/v2.2` branch. `main` doesn't know about them yet.

### Merge

Merging takes the work from one branch and combines it into another. When `hw/v2.2` is ready, you merge it into `develop`, then eventually `develop` into `main`.

```
main:     A --- B --- C ----------- G   ← G is the merge commit
                       \           /
hw/v2.2:               D --- E --- F
```

### Tag

A tag is a permanent label attached to a specific commit. Unlike a branch (which moves forward as you add commits), a tag never moves. It is forever pointing at that one commit. This is how version numbers work in git.

```bash
git tag -a hw/v2.1 -m "First stable release"
```

### Remote

A remote is a copy of the repository hosted somewhere else — GitHub, GitLab, etc. `origin` is the default name for your remote. You `push` to send your local commits to the remote, and `pull` to bring remote commits down to your machine.

---

## 3. Repository structure

Structure your repo so hardware and firmware are clearly separated. They have different release cycles — a firmware bug fix should not require a new hardware version number.

```
nexus-node/
│
├── hardware/
│   ├── kicad/
│   │   ├── nexus-node.kicad_pro
│   │   ├── nexus-node.kicad_sch
│   │   ├── nexus-node.kicad_pcb
│   │   └── nexus-node.kicad_prl       ← do NOT commit (local prefs)
│   │
│   ├── fabrication/
│   │   ├── v2.1/
│   │   │   ├── gerbers/
│   │   │   ├── nexus-node-bom-v2.1.csv
│   │   │   └── nexus-node-cpl-v2.1.csv
│   │   └── v2.2/
│   │       └── gerbers/
│   │
│   └── 3d/
│       └── nexus-node-v2.2.step
│
├── firmware/
│   ├── src/
│   │   ├── nexus-node.ino
│   │   ├── rs485.h
│   │   └── rs485.cpp
│   └── lib/
│       └── (vendored libraries)
│
├── docs/
│   ├── pinout.md
│   ├── bom.md
│   ├── bom.xlsx
│   ├── schematic-v2.2.pdf           ← exported PDF for each release
│   └── images/
│       ├── pcb-render-v2.2.png
│       └── board-photo-v2.1.jpg
│
├── README.md
├── CHANGELOG.md
├── LICENSE
└── .gitignore
```

**Key rule:** Never overwrite old fabrication outputs. When you generate gerbers for v2.2, put them in `fabrication/v2.2/` — do not replace the v2.1 folder. This means you can always reproduce any manufactured board exactly.

---

## 4. The .gitignore file for KiCad

KiCad generates many files you do not want in version control. Create this `.gitignore` at the root of your repo:

```gitignore
# KiCad backup and cache files
*.bak
*.kicad_prl
fp-info-cache
_autosave-*
*.lck
*-backups/

# KiCad local settings (machine-specific, don't share)
*.kicad_prl

# OS junk
.DS_Store
Thumbs.db
desktop.ini

# Python cache
__pycache__/
*.pyc

# Editor temp files
*.swp
*~
```

**Note on gerbers:** You can choose whether to commit gerbers or not.
- **Commit them** if you want anyone to manufacture the board without installing KiCad. Put them in `fabrication/v2.x/gerbers/`.
- **Don't commit them** if you prefer to keep the repo lean and only store them in GitHub Releases. If so, add `*.gbr`, `*.drl`, `*.csv` to `.gitignore`.

The recommended approach for a professional project: do not commit gerbers to git, but attach them to every GitHub Release (see [section 7](#7-version-tagging-and-releases)).

---

## 5. Branch strategy

This project uses a simplified **GitFlow** model with four branch types.

### The four branches

| Branch | Purpose | Who commits here | Lifetime |
|--------|---------|-----------------|----------|
| `main` | Stable releases only. Always manufacturable. | Nobody directly — merge only | Permanent |
| `develop` | Integration branch. All features land here first. | Nobody directly — merge only | Permanent |
| `hw/description` | Hardware changes (KiCad schematic, PCB layout) | You, directly | Short — delete after merge |
| `feature/description` | New firmware features or capabilities | You, directly | Short — delete after merge |
| `fix/description` | Bug fixes, errata corrections | You, directly | Short — delete after merge |
| `docs/description` | Documentation only, no code or hardware | You, directly | Short — delete after merge |

### The golden rules

1. **Never commit directly to `main`.** It should only ever receive merges from `develop`.
2. **Never commit directly to `develop`.** It should only ever receive merges from feature/hw/fix branches.
3. **One concern per branch.** Don't fix a silkscreen error and add a new connector in the same branch. Make two branches.
4. **Delete branches after merging.** The commit history is preserved — you don't need the branch pointer anymore.
5. **Pull before you branch.** Always `git pull` on `develop` before creating a new branch from it, so you start from the latest state.

### Branch naming convention

```
hw/v2.2-sp491-transceiver       ← hardware change
hw/v2.2-rgb-led-routing         ← another hardware change  
feature/rs485-protocol          ← firmware feature
feature/limit-switch-isr        ← firmware feature
fix/xdir-timing                 ← bug fix
fix/reset-pullup-missing        ← errata fix
docs/bom-update                 ← documentation only
```

Use lowercase, hyphens not underscores, be descriptive enough that the branch name alone tells you what it is.

---

## 6. Daily workflow — step by step

### 6.1 First time setup

Do this once when you create the project.

```bash
# Create the repo locally
mkdir nexus-node
cd nexus-node
git init

# Create your folder structure
mkdir -p hardware/kicad hardware/fabrication firmware/src firmware/lib docs/images

# Create initial files
touch README.md CHANGELOG.md LICENSE .gitignore

# First commit on main
git add .
git commit -m "init: initial repo structure"

# Create the develop branch
git checkout -b develop

# Connect to GitHub (create the repo on GitHub first, then:)
git remote add origin https://github.com/yourusername/nexus-node.git
git push -u origin main
git push -u origin develop
```

### 6.2 Starting any piece of work

This is the sequence you follow every single time you start something new.

```bash
# Step 1: Make sure develop is up to date
git checkout develop
git pull

# Step 2: Create your branch from develop
git checkout -b hw/v2.2-sp491-transceiver

# You are now on the new branch. Start making changes in KiCad.
```

### 6.3 Committing your work

Commit early and often. Small commits with clear messages are far better than one giant commit at the end.

```bash
# See what files have changed
git status

# Stage specific files (preferred — be explicit about what you're committing)
git add hardware/kicad/nexus-node.kicad_sch
git commit -m "hw: replace MAX485 with SP491 full-duplex transceiver"

# Do more work in KiCad...

git add hardware/kicad/nexus-node.kicad_pcb
git commit -m "hw: route SP491 14-pin footprint, update net connections"

git add hardware/kicad/nexus-node.kicad_pcb
git commit -m "hw: update silkscreen labels for SP491 bus pins"

git add docs/bom.md docs/bom.xlsx
git commit -m "docs: update BOM — SP491 replaces MAX485, add DE GPIO note"
```

#### Writing good commit messages

The format is: `type: short description in present tense`

```
hw: add RESET pull-up resistor R9 on PF6
hw: fix SP491 decoupling cap placement near pin 14
fw: implement RS485 address parsing in Serial2 ISR
fix: correct UART0 pin assignment — PA0/PA1 not PA2/PA3
docs: add SP491 wiring diagram to README
```

Types to use:
- `hw:` — schematic or PCB layout changes
- `fw:` — firmware changes
- `fix:` — bug or error corrections  
- `docs:` — documentation only
- `init:` — initial setup commits
- `refactor:` — reorganising without changing function

**Never write vague messages like** `"update"`, `"changes"`, `"fix stuff"`. In six months you will have no idea what that commit did.

### 6.4 Pushing your branch and opening a Pull Request

When your work is ready for review (even if you're working alone, Pull Requests create a useful checkpoint):

```bash
# Push your branch to GitHub
git push origin hw/v2.2-sp491-transceiver

# Go to GitHub in your browser
# You will see a banner: "Compare & pull request"
# Click it, write a description of what changed and why
# Set base branch: develop  ←  compare branch: hw/v2.2-sp491-transceiver
# Create the pull request
```

Review the diff on GitHub. Look at your schematic changes as a sanity check. When satisfied, merge the PR on GitHub (use "Squash and merge" for clean history, or "Merge commit" to preserve individual commits).

### 6.5 After merging — clean up

```bash
# Get the merged result locally
git checkout develop
git pull

# Delete the local branch — it's been merged, you don't need it
git branch -d hw/v2.2-sp491-transceiver

# Also delete the remote branch (GitHub may offer to do this automatically)
git push origin --delete hw/v2.2-sp491-transceiver
```

### 6.6 Working on multiple things at once

Suppose you are mid-way through routing the PCB on `hw/v2.2-layout` and you suddenly spot an error in the CHANGELOG that needs fixing now.

```bash
# Save your current work-in-progress on your branch
git add hardware/kicad/nexus-node.kicad_pcb
git commit -m "hw: WIP — routing SP491 area, incomplete"

# Switch to a new branch for the docs fix
git checkout develop
git pull
git checkout -b docs/fix-changelog-v2.1-entry

# Make the docs change
git add CHANGELOG.md
git commit -m "docs: fix incorrect date in v2.1 changelog entry"
git push origin docs/fix-changelog-v2.1-entry
# Open PR → merge → delete branch

# Return to your PCB work
git checkout hw/v2.2-layout
# Continue routing where you left off
```

This is the correct way. Never try to do two unrelated things on the same branch.

---

## 7. Version tagging and releases

### 7.1 When to make a release

Make a hardware release when:
- The board is ready to send to the fab (gerbers generated, DRC clean)
- A significant bug has been fixed that changes the schematic or layout
- A component has been swapped (e.g. MAX485 → SP491)

Make a firmware release when:
- A stable, tested firmware version is ready for use on the manufactured board

### 7.2 The release process

```bash
# Step 1: Make sure develop is stable
git checkout develop
git pull
# Run your DRC in KiCad. Fix any errors. Commit them.

# Step 2: Merge develop into main
git checkout main
git pull
git merge develop
git push origin main

# Step 3: Create the annotated tag
git tag -a hw/v2.2 -m "hw/v2.2: SP491 full-duplex transceiver, corrected RGB LED pins PF3/4/5, RESET pull-up R9 added, AVDD connected"
git push origin hw/v2.2

# For firmware releases, use a separate tag prefix:
git tag -a fw/v1.0 -m "fw/v1.0: initial RS485 node firmware, Serial2 half-duplex protocol"
git push origin fw/v1.0
```

### 7.3 Version number meaning

```
hw/v2.1   →   hw/v2.2          Minor change
              ↑ increment       Layout, silkscreen, component swap
                                Schematic is functionally the same

hw/v2.x   →   hw/v3.0          Major change
              ↑ increment       New IC, different pinout, power architecture change
                                Schematic has changed significantly
```

Hardware and firmware are versioned independently:

| Tag | Meaning |
|-----|---------|
| `hw/v2.1` | First release of hardware revision 2 |
| `hw/v2.2` | Minor hardware update (layout, component swap) |
| `fw/v1.0` | First stable firmware release |
| `fw/v1.1` | Firmware bug fix (no hardware change required) |

### 7.4 GitHub Releases

After tagging, create a GitHub Release to attach the fabrication files. This is what makes the project professionally usable by others.

1. Go to GitHub → Releases → Draft a new release
2. Choose tag: `hw/v2.2`
3. Title: `Hardware v2.2 — SP491 full-duplex transceiver`
4. Description: copy from CHANGELOG.md
5. Attach these files:
   - `nexus-node-schematic-v2.2.pdf` (exported from KiCad)
   - `nexus-node-gerbers-v2.2.zip` (your fabrication folder zipped)
   - `nexus-node-bom-v2.2.xlsx`
   - `nexus-node-v2.2.step` (3D export)

Now anyone can manufacture your board at any version without cloning the repo or installing KiCad.

### 7.5 Going back to any version

```bash
# Check out exactly what the board looked like at v2.1
git checkout hw/v2.1

# Open KiCad — you will see the v2.1 schematic and layout exactly as it was
# This is read-only. Don't commit from here.

# Return to the latest
git checkout main
```

---

## 8. Handling merge conflicts

A conflict happens when two branches have both changed the same part of the same file. KiCad files (especially `.kicad_sch` and `.kicad_pcb`) are text-based, so git can sometimes merge them automatically — but not always.

### 8.1 What a conflict looks like

```
CONFLICT (content): Merge conflict in hardware/kicad/nexus-node.kicad_sch
Automatic merge failed; fix conflicts and then commit the result.
```

Opening the file, you will see conflict markers:

```
<<<<<<< HEAD
(net (code 47)(name "RS485_A")(node (ref "U2")(pin "13")))
=======
(net (code 47)(name "BUS_A")(node (ref "U2")(pin "13")))
>>>>>>> feature/rs485-rename-nets
```

`HEAD` is your current branch. Below `=======` is the incoming branch. You must decide which version is correct (or combine them) and remove the markers.

### 8.2 Resolving the conflict

For KiCad files, the safest approach is usually to use one complete version or the other — do not try to manually edit the binary-adjacent KiCad file structure. 

```bash
# Option A: keep your current branch's version
git checkout --ours hardware/kicad/nexus-node.kicad_sch

# Option B: keep the incoming branch's version  
git checkout --theirs hardware/kicad/nexus-node.kicad_sch

# After choosing, mark as resolved and commit
git add hardware/kicad/nexus-node.kicad_sch
git commit -m "merge: resolve net naming conflict, keep RS485_A naming convention"
```

### 8.3 Preventing conflicts

The best way to handle conflicts is to avoid them:

- Keep branches short-lived. A branch open for two weeks will conflict badly.
- Communicate. If two people are both editing the schematic, coordinate who does what.
- Merge `develop` into your long-running branch regularly to stay up to date:

```bash
# While on your hw/v2.2-layout branch:
git fetch origin
git merge origin/develop     # bring develop's latest into your branch
# Resolve any conflicts now, while they're small, rather than at the end
```

---

## 9. Working with GitHub and GitLab together

GitHub is excellent for open-source visibility, community, and Issues. GitLab has better built-in CI/CD pipelines for automated tasks like gerber export and DRC checking. You can use both.

### 9.1 Using GitLab as a pull mirror (simplest)

Keep GitHub as your primary remote. Set GitLab to automatically mirror it.

1. Create the repo on GitLab (same name: `nexus-node`)
2. GitLab → Settings → Repository → Mirroring repositories
3. Add mirror URL: `https://github.com/yourusername/nexus-node.git`
4. Direction: Pull
5. GitLab will sync automatically every hour

You push only to GitHub. GitLab stays in sync. You use GitLab's CI/CD pipelines on top of your GitHub code.

### 9.2 Pushing to both remotes manually

If you want explicit control:

```bash
# Add GitLab as a second remote
git remote add gitlab https://gitlab.com/yourusername/nexus-node.git

# Push to GitHub (origin)
git push origin main
git push origin hw/v2.2

# Push to GitLab
git push gitlab main
git push gitlab hw/v2.2
```

### 9.3 GitLab CI for automated gerber export

Create `.gitlab-ci.yml` at the root of your repo:

```yaml
stages:
  - fabrication

generate-gerbers:
  stage: fabrication
  image: kicad/kicad:8.0
  script:
    - kicad-cli pcb export gerbers
        --output hardware/fabrication/ci/
        hardware/kicad/nexus-node.kicad_pcb
    - kicad-cli pcb export drill
        --output hardware/fabrication/ci/
        hardware/kicad/nexus-node.kicad_pcb
    - kicad-cli sch export pdf
        --output docs/schematic-ci.pdf
        hardware/kicad/nexus-node.kicad_sch
  artifacts:
    name: "nexus-node-gerbers-$CI_COMMIT_TAG"
    paths:
      - hardware/fabrication/ci/
      - docs/schematic-ci.pdf
    expire_in: 90 days
  only:
    - tags     # only runs when you push a tag (hw/v2.x)
```

Now every time you push a `hw/v2.x` tag, GitLab automatically generates the gerbers and makes them available as a downloadable artifact — without you having to remember to export them manually.

---

## 10. Quick reference card

### Commands you use daily

```bash
# Where am I? What has changed?
git status
git branch                          # list local branches
git log --oneline --graph --all     # visual history

# Starting work
git checkout develop && git pull    # always start here
git checkout -b hw/description      # create new branch

# Saving work
git add hardware/kicad/file.kicad_sch
git commit -m "hw: describe what changed and why"
git push origin hw/description

# Finishing work (after PR is merged on GitHub)
git checkout develop && git pull
git branch -d hw/description

# Releases
git checkout main && git merge develop && git push
git tag -a hw/v2.2 -m "description of what changed"
git push origin hw/v2.2

# Going back in time
git checkout hw/v2.1               # view old version
git checkout main                  # come back to present

# Comparing versions
git diff hw/v2.1 hw/v2.2           # what changed between versions
git log hw/v2.1..hw/v2.2 --oneline # commits between two tags
```

### Branch naming cheatsheet

| Prefix | When to use | Example |
|--------|-------------|---------|
| `hw/` | KiCad schematic or PCB changes | `hw/v2.2-sp491` |
| `fw/` | Arduino firmware changes | `fw/rs485-protocol` |
| `feature/` | New capability (hw or fw) | `feature/rgb-led` |
| `fix/` | Bug or errata fix | `fix/reset-pullup` |
| `docs/` | Documentation only | `docs/pinout-update` |

### Tag naming cheatsheet

| Tag | When | Meaning |
|-----|------|---------|
| `hw/v2.1` | Board ready for fab | Hardware v2, first revision |
| `hw/v2.2` | Minor layout/component change | Hardware v2, second revision |
| `hw/v3.0` | Major schematic change | Hardware v3, first revision |
| `fw/v1.0` | Stable firmware | Firmware v1, first release |
| `fw/v1.1` | Firmware bug fix | Firmware v1, patch |

### The complete workflow in one picture

```
develop ──┬──────────────────────────────────────────┬──► (ongoing)
          │                                          │
          │  git checkout -b hw/v2.2-sp491           │
          │         │                                │
          │    (make KiCad changes)                  │
          │    (commit, commit, commit)               │
          │         │                                │
          │    git push → open PR on GitHub          │
          │         │                                │
          │    (review diff, approve)                │
          │         │                                │
          └─────────┘  (PR merged into develop)      │
                                                     │
main ────────────────────────────────────────────────┤
                                              git merge develop
                                              git tag hw/v2.2
                                              (GitHub Release)
```

---

## Appendix: CHANGELOG.md format

Keep a CHANGELOG.md using the [Keep a Changelog](https://keepachangelog.com) format. GitHub renders it beautifully and it becomes the permanent record of every design decision.

```markdown
# Changelog

All notable changes to nexus-node are documented here.
Hardware and firmware are versioned independently.

---

## [hw/v2.2] - 2025-06-01
### Added
- SP491 full-duplex RS485 transceiver (replaces MAX485)
- DE GPIO control on PD6 for slave nodes
- RESET pull-up resistor R9 (10kΩ) on PF6 — was floating in v2.1

### Changed
- RS485 moved to USART2 (PF0/PF1/PF3) from USART3
- RGB LED on PF3/PF4/PF5 for PWM via TCA0

### Fixed
- AVDD (pin 18) now connected to VDD rail
- Second GND (pin 19) now connected

---

## [hw/v2.1] - 2025-05-14
### Added
- Initial hardware release
- AVR128DB32 MCU in TQFP-32 package
- Half-duplex RS485 via MAX485ESA+
- SPI on PA4/PA5/PA6/PA7
- Limit switches on PD1/PD2 with RC debounce
- Debug LED on PF2
- UPDI header on dedicated pin 27
- 24V → 5V → 3.3V power chain

---

## [fw/v1.0] - 2025-05-20
### Added
- Initial firmware release
- RS485 half-duplex protocol via Serial2 (USART2)
- STEP/DIR stepper control on PC0/PC1
- Limit switch interrupt service routines
- RGB LED status indication (red=fault, green=idle, blue=moving)
```

---

*This document is part of the nexus-node project.*  
*Last updated: May 2025*
