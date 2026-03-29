+++
author = "Eduard Marbach"
title = "Travo: A Modern Web UI for OpenWrt Travel Routers"
date = "2026-03-29"
image = "images/logo.webp"
description = "Introducing Travo—an open-source Go + React dashboard for OpenWrt travel routers, with WiFi, captive portals, VPN, AdGuard Home, and LuCI coexistence."
tags = [
    "openwrt",
    "travel-router",
    "travo",
    "self-hosted",
    "go",
    "react",
    "networking",
]
categories = [
    "projects",
    "networking",
]
+++

[Travo](https://github.com/raydak-labs/travo) is a mobile-first web UI for OpenWrt on compact travel routers (for example GL.iNet Beryl AX / Slate-class devices, or any recent aarch64 or x86_64 OpenWrt 23.05+ build). It wraps day-to-day tasks—WiFi, hotel captive portals, upstream links, VPN, DNS filtering—in a focused dashboard while **LuCI** stays available for full OpenWrt administration.

<!--more-->

## Stack and shape

The project is a **pnpm monorepo**: a **React** SPA (Vite, TanStack Query/Router, shared TypeScript types) in front of a **Go** API on **Fiber**. The backend talks to the router through the usual OpenWrt surfaces (UCI, ubus, package tooling), so behavior stays close to how the system is meant to be configured.

## Why not only LuCI?

LuCI is the reference UI for OpenWrt. On a phone, or when you only need “join this hotel WiFi” or “toggle WireGuard,” a slimmer UI helps. Travo is built to **coexist** with LuCI: the documented install path can put Travo on port 80 and move LuCI to another port (see the upstream README for the exact layout and ports, including optional **AdGuard Home**).

## What you get

- Dashboard with live stats and quick actions  
- **WiFi**: scan, connect, mode switching (client / repeater / AP), aligned with safe apply patterns where wireless changes are involved  
- **Captive portals**: detection and streamlined hotel login flows  
- **VPN**: WireGuard- and Tailscale-oriented workflows  
- **Services**: install and manage extras such as AdGuard Home from the UI  
- **System**: network, DNS, and firewall-related settings where the project exposes them  
- **Theming**: light / dark, responsive layout  

For the canonical feature list and security notes, use the [README on GitHub](https://github.com/raydak-labs/travo/blob/main/README.md).

## Install on a router

SSH to the device and run:

```sh
wget -O- https://raw.githubusercontent.com/raydak-labs/travo/main/scripts/install.sh | sh
```

Then **change any default credentials** immediately (the README documents defaults such as AdGuard’s initial login).

**Pre-built binaries and `.ipk` packages** for aarch64 and x86_64 are on [GitHub Releases](https://github.com/raydak-labs/travo/releases).

## Develop locally

Clone [github.com/raydak-labs/travo](https://github.com/raydak-labs/travo), install Node (≥ 20), pnpm (≥ 9), and Go (≥ 1.23) as documented there, then `pnpm install`, `cd backend && go mod tidy`, and `make dev`. Use `make test`, `make lint`, and `make build` before contributing; see `CONTRIBUTING.md` in the repo. Docker Compose is supported for a containerized dev loop.

## Links

- **Repository:** [github.com/raydak-labs/travo](https://github.com/raydak-labs/travo)  
- **Releases:** [github.com/raydak-labs/travo/releases](https://github.com/raydak-labs/travo/releases)  
- **License:** MIT  

Ports, install options, and rollout details can change; treat this post as an overview and follow the upstream README and deployment docs for the current picture.
