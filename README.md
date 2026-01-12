# Base44 + Claude Code Integration Guide

A practical guide on how to connect Base44 (low-code platform) to Claude Code for local development with real data sync.

## Live Guide

**[View the full guide](https://lior6221.github.io/base44-claude-code-guide/)**

## What's Inside

- Step-by-step integration instructions
- Dev mode configuration for local development with real data
- REST API setup for data synchronization
- Backend functions modification pattern
- Common issues and solutions (with real screenshots)

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CODE SYNC (GitHub)                        │
├─────────────────────────────────────────────────────────────┤
│   Base44 App  ◄────── GitHub Sync ──────►  Claude Code      │
├─────────────────────────────────────────────────────────────┤
│                    DATA SYNC (REST API)                      │
├─────────────────────────────────────────────────────────────┤
│   Base44 API  ◄────── REST API ──────►  Local Dev Server    │
└─────────────────────────────────────────────────────────────┘
```

## Key Features

- **50 minutes setup** - Including trial and error
- **Two-way sync** - Local changes reflect in Base44, and Base44 changes appear locally (after refresh)
- **$0 extra cost** - Beyond existing Base44 and Claude subscriptions
- **Backwards compatible** - Production code unaffected

## Quick Start

1. Set up GitHub sync with Base44
2. Clone repo locally for Claude Code
3. Configure dev mode with API credentials
4. Update backend functions for dual auth support

## Technologies

- Base44 (Low-code platform)
- Claude Code (AI coding assistant)
- REST API (Direct API integration)
- Vite + React

---

Built by **Lior Levi** with **Claude Code** | January 2026
