# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a documentation and context repo for a personal homelab journey. It serves as the central hub — journal entries, learning notes, and a map of content (MOC) linking to the repos where actual configuration and code live.

## Map of Content (MOC)

| Repo | Path | Purpose | Key files |
|------|------|---------|-----------|
| **nix-config** | `~/projects/nix-config` | Nix flake: nix-darwin + home-manager system config | `CLAUDE.md`, `RUNBOOK.md`, `flake.nix` |

As new repos are added (e.g., NixOS host configs, services, infrastructure), add them here so the agent and human both have a single place to orient.

## Conventions

- `JOURNAL.md` — Chronological learning log. Each entry covers what was done, what went wrong, and what was learned.
- Related repos have their own `CLAUDE.md` (agent instructions) and `RUNBOOK.md` (operational knowledge). This repo links to them but doesn't duplicate their content.
- **When working in another repo's context, always read that repo's `CLAUDE.md` and `RUNBOOK.md` first.** Use the MOC table above to find the paths.
