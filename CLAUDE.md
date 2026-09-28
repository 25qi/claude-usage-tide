# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-target SwiftPM macOS menu bar app (macOS 13+, Swift 5.9+, AppKit, no Xcode project, no dependencies). It shows Claude subscription usage by reading rate-limit headers off a 1-token Messages API call. There are no tests and no linter configured.

## Commands

```bash
swift build                      # debug build
swift build -c release           # release build
./.build/release/ClaudeUsageTide --probe   # one fetch, print 5h/7d readings, exit (no UI; never prints the token)
./install.sh                     # release build → bundle → install to ~/Applications → relaunch (also the update path)
./bundle.sh <binary> <dest.app> [--homebrew]   # wrap a built binary into a signed .app
```

The app must run from a signed `.app` bundle for `SMAppService` (Launch at Login) to work; running the raw binary is fine for `--probe` or quick UI checks. Bundle ID is `com.qi.claude-usage-tide`; the `compactTitle` preference lives in that defaults domain.

`bundle.sh` is shared with the Homebrew formula (in the separate `25qi/homebrew-tap` repo), so both install paths produce the same bundle. Version numbers (`CFBundleShortVersionString`, `CFBundleVersion`) are hard-coded in `bundle.sh`'s Info.plist heredoc.

## Repository layout

This repo is a GitHub fork of `hamed-elfayome/Claude-Usage-Tracker`, but the code here shares no history with upstream.

- `tide` is the default branch and holds this app. Upstream's branches (`main`, `next-release`, `gh-pages`) are upstream code, so never merge them into `tide`.
- `feature/reset-notifications` and `fix/oauth-refresh-deadlock` are earlier changes to the upstream app, kept for reference.
- Release tags are `tide-vX.Y.Z`, because upstream already owns `vX.Y.Z` in this fork. A release means updating the formula's `url` and `sha256` in `25qi/homebrew-tap` to the new tag's archive.

## Architecture

Four files in `Sources/ClaudeUsageTide/`, one flow:

`AppDelegate.refresh()` (main.swift) → `UsageFetcher.fetch()` → `Credentials.accessToken()` → POST to `api.anthropic.com/v1/messages` → parse `anthropic-ratelimit-unified-{5h,7d}-{utilization,reset}` headers → `render()` rebuilds the status item title and the whole `NSMenu`.

On `FetchError.unauthorized`, `refresh()` calls `TokenRenewal.attempt()` and retries the fetch exactly once.

Key invariants (each is documented in the source; don't "fix" them away):

- **Credentials are strictly read-only.** Never redeem the refresh token or write to the keychain: refresh tokens rotate on use, and a second process refreshing them logs Claude Code out. Renewal is delegated to the `claude` CLI via `TokenRenewal`: first `claude auth status` (free), then a minimal `claude -p` (~1k tokens) rate-limited to once per hour.
- **Credential lookup picks the freshest token** across the keychain (`Claude Code-credentials` and the `-<sha256(configDir)[:8]>` suffixed variant for Claude Code v2.1.52+) and `~/.claude/.credentials.json`, comparing `expiresAt`. Respects `CLAUDE_CONFIG_DIR`. Keychain is read by shelling out to `/usr/bin/security`, with a regex fallback because that CLI truncates large payloads.
- **A 429 is a valid reading**, not an error: the headers still arrive.
- **Failures are never fatal.** The last good reading stays on screen and is greyed out (`tertiaryLabelColor`) while `lastError` is set; the 5-minute timer keeps running, and a wake notification triggers an immediate refresh.
- **The status item is created exactly once** with `autosaveName` so the user's ⌘-drag position survives; don't recreate it.
- **`ManagedByHomebrew`** (an Info.plist key set by `bundle.sh --homebrew`) hides the Launch at Login toggle, because `brew services` owns login launch and both would start the app twice.
- `claude` is located by hardcoded paths in `TokenRenewal.locateCLI()`, since a menu bar app inherits launchd's bare PATH.
- `AppDelegate` is `@MainActor`; timer and notification callbacks use `MainActor.assumeIsolated`. `--probe` uses `Task.detached` + `dispatchMain()` so the work does not queue behind the parked main thread.

Token cost per poll (~11 tokens, ~3,200/day) and the header table are documented in README.md. Keep README in sync when polling interval, model, or renewal behavior changes.
