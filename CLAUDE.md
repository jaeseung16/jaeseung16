# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a GitHub profile repository (`jaeseung16`). The `README.md` renders as the public-facing GitHub profile page for @jaeseung16. There is no build system, test suite, or application code.

## Structure

- `README.md` — GitHub profile page content (Markdown with embedded SVG image references)
- `profile/*.svg` — Auto-generated GitHub stats cards (stats, top-langs, pin); do not hand-edit these
- `.github/workflows/grs.yml` — GitHub Action that regenerates the SVG cards weekly (Mondays at 5 AM ET) via `readme-tools/github-readme-stats-action`

## Workflow

The SVG cards in `profile/` are committed automatically by the `Update README cards` GitHub Action. To trigger a manual regeneration, use the `workflow_dispatch` event on that workflow.

To update the profile, edit `README.md` directly. The SVGs are referenced as relative paths (`./profile/stats.svg`, etc.) and rendered inline by GitHub.
