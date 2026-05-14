# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Single-file static web app for tracking Japan credit-card spending across multiple Taiwanese credit cards, computing cashback per card based on each card's tiered reward structure (base rate + capped bonus rate). Mobile-first, designed as a PWA-style app (apple-touch-icon, viewport meta, safe-area insets).

Despite the parent directory name (`VueProjects`), this is **not** a Vue project — it's a single `index.html` with inline CSS and vanilla JS. No build step, no package.json, no tests. To run it, just open `index.html` in a browser (or serve the directory with any static server).

## Architecture

All logic lives in [index.html](index.html). Two pieces drive everything:

**`CARDS` config object** — each card declares `baseRate`, `bonusRate` (or `null`), `bonusCapTwd` (or `null`), `bonusLabel`, and a human `note`. Cards with `bonusRate: null` are flat-rate (no tier, no cap, "符合加碼" checkbox is hidden when that card is active). Card rates are **time-sensitive promotional values for 2026 H1** — when promo periods roll over, the `CARDS` config and the header subtitle need updating.

**`computeCardStats(cardId)`** — this is the cashback engine. Iterates that card's transactions in order, and for tiered cards: while `bonusSpent < bonusCapTwd`, the portion that fits in remaining room earns `bonusRate`, the overflow earns `baseRate`. Order matters because the cap fills sequentially. The function is called per-card from `render()` and also from `updatePreview()` to estimate the next transaction's cashback given current cap usage.

**Persistence** — single localStorage key `jp_card_tracker_v2` holds `{ transactions, fxRate, activeCardId }`. `loadState()` migrates from the old `dbs_eco_tracker_v1` key (legacy single-card schema) by tagging every old tx with `cardId: 'dbs_eco'`. Any tx missing or with an unknown `cardId` also falls back to `dbs_eco`. If you change the `KEY` constant, write a migration path or existing users lose data.

**UI is fully re-rendered** on every state change — no diffing, no virtual DOM. `render()` rebuilds chips, cap card, stat grid, summary list, and tx list from `state`. Cheap because the data is small. The progress bar is **hidden** (not zeroed or full) for cards without `bonusCapTwd` — setting it to 100% looks like "cap reached" and is wrong.

## Workflow

After making any file changes, **always suggest a Chinese commit subject** (繁體中文) at the end of the response. Format: short imperative line under ~50 characters, e.g. `修正無上限卡進度條顯示為滿格的問題` or `新增 4 張日本回饋信用卡`. Do not commit automatically — just suggest the message so the user can copy-paste.

## Conventions

- Currency: JPY entered by user, converted via `state.fxRate` (default 0.21) to TWD at entry time and stored as `tx.twd`. Changing the FX rate later does NOT recompute past transactions — this is intentional (each tx locks in the rate it was entered at).
- All comments and user-facing strings are in Traditional Chinese (zh-Hant).
- The `bonus` boolean on a transaction means different things per card: for eco/eco極簡 it's "qualifies for 5% overseas bonus", for 熊本熊 it's "at a designated store". The card's `bonusLabel` is what's shown in the UI.
