# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

clipwget is a lightweight, clipboard-driven media downloader for Windows, written in Go (MIT licensed).

## Current state

The repository contains design and implementation documentation in `docs/` (`docs/software-design.md` and `docs/implementation-plan.md`), along with `README.md`, `LICENSE`, `AGENTS.md`, and the standard GitHub Go `.gitignore`. Source code and `go.mod` (`github.com/jinhoOps/clipwget`) are ready to be implemented following the 7-step roadmap in `docs/implementation-plan.md`. When code is added, update this file with the active entry points and verified commands.

## Design & Collaboration Docs

- Always consult `docs/software-design.md` for architecture, database schema, filename formatting rules, and Windows integration details.
- Track implementation milestones and inter-package contracts in `docs/implementation-plan.md`.

## Notes

- Target platform is Windows; development also happens on Windows (PowerShell / Git Bash).
- `.gitignore` excludes `*.exe`, test binaries, coverage files, `go.work`, and `.env` — don't commit build outputs.
