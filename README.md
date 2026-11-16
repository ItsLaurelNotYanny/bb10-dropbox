# bb10-dropbox

English | [简体中文](README_cn.md)

Native Dropbox client and background synchronization service for BlackBerry 10 (BlackBerry Passport tested).

## Overview

`bb10-dropbox` is a lightweight, native Dropbox solution for BlackBerry 10 devices running QNX 6.6 / Cascades. It provides:
- Seamless manual file browsing, uploading, and downloading via Dropbox API v2.
- An independent background synchronization service capable of two-way folder synchronization and camera uploads.
- Minimal dependency footprint tailored for resource-constrained BB10 devices.

## Repository Architecture

This project adopts a dual-repository development model:
- **Private Repository** (`bb10-dropbox-private`): Tracks complete development history, internal milestones, engineering notes, and feasibility assessments.
- **Public Repository** (`bb10-dropbox`): Prepared open-source release tree providing clean, curated snapshots of the client, sync service, and public documentation.

Development proceeds within a single local worktree. The public release subset is generated and verified using `scripts/prepare_release.sh`.

## Repository Structure

```text
bb10-dropbox/
├── REUSE-ASSESSMENT.md      # Upstream reuse assessment & architectural decisions
├── REUSE-ASSESSMENT_cn.md   # Chinese translation of reuse assessment
├── README.md                # Project documentation (English)
├── README_cn.md             # Project documentation (Simplified Chinese)
├── check.sh                 # Project gate verification script
└── scripts/
    └── prepare_release.sh   # Clean public release preparation script
```

## Quick Start & Verification

Run the project gate:
```bash
# Fast gate check (syntax, formatting, static checks)
./check.sh --fast

# Full gate check
./check.sh
```

## Preparing Public Open-Source Release

To update the clean public `release` branch from current `main` branch without switching worktrees:
```bash
./scripts/prepare_release.sh
```

To push changes to GitHub remotes:
```bash
# Push full development tree to private repository
git push origin main

# Push clean release tree to public repository
git push public release:main
```

## License

To be determined upon initial open-source release.
