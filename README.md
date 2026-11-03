# ClipWget (`clipwget`)

A lightweight, clipboard-driven media downloader for Windows written in pure Go (`CGO_ENABLED=0`).

## Documentation

- [Software Design Specification (`docs/software-design.md`)](docs/software-design.md) — Dual-track clipboard detection (`x.com` social links & `.m3u8` / DevTools `cURL` HLS streams), smart filename formatting, volatile in-memory inbox with persistent completed-download history, hybrid FFmpeg strategy, and Windows native Toast/Tray/WebUI architecture.
- [Implementation Plan (`docs/implementation-plan.md`)](docs/implementation-plan.md) — Package layout, inter-package contracts, 7-step milestone checklist, and verification matrix.
