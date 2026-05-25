# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **documentation-only repository** — no build system, no tests, no dependencies. All content lives in `README.md`.

The README documents Oracle Linux in-place upgrades (OL7 → OL8) using the LEAPP tool, covering the full upgrade procedure and post-upgrade boot failure troubleshooting. Troubleshooting covers: LVM volume group name mismatches, stale initramfs, UEFI vs BIOS GRUB config path differences, and manual sysroot mounting from a dracut emergency shell.

## Making Changes

Edit `README.md` directly. When updating:

- Keep the Table of Contents in sync with section headings
- Shell commands must be copy-pasteable and tested against OL7/OL8 — avoid paraphrasing commands
- The troubleshooting section follows a structured Phase 1 → 2 → 3 flow (access → evidence → fix); preserve that ordering
- Fix table rows reference anchors (`Fix A`, `Fix B`, etc.) that must stay consistent with the heading names in Phase 3
- Any `grub2-mkconfig` command must show both paths: BIOS (`/boot/grub2/grub.cfg`) and UEFI (`/boot/efi/EFI/redhat/grub.cfg`) — never show only one

## Publishing

Changes go directly to `main` on GitHub (`mss-wiki/aeon-oraclelinux-leapp-upgrade`). No PR process — commit and push.
