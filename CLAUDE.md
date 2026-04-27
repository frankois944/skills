# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A **Claude skill** for the `spmForKmp` Gradle plugin (`io.github.frankois944.spmForKmp`) — a replacement for the deprecated Kotlin CocoaPods plugin that integrates Swift Package Manager dependencies into Kotlin Multiplatform projects targeting Apple platforms.

The repo contains one skill (`spmforkmp/`) with three layers:
- `SKILL.md` — skill frontmatter (trigger conditions), scenario routing logic, and non-negotiable rules
- `references/*.md` — modular technical recipes loaded by scenario (not all loaded at once)
- `evals/evals.json` — evaluation test cases for validating skill outputs

## Architecture: Scenario-Driven Reference Loading

The skill is structured around three mutually exclusive scenarios. `SKILL.md` routes to the right reference files; the references are not meant to be read all at once:

- **Scenario A** (basic, no external SPM): `setup.md` + `bridge.md`
- **Scenario B** (external SPM package): `dependencies.md` + `exporting.md` (always both)
- **Scenario C** (migration from CocoaPods plugin): `migration.md`

Additional references (`interoperability.md`, `troubleshooting.md`) are loaded on demand.

Each reference file is designed to be self-contained and independently useful — when editing one, don't assume the reader has read the others.

## Key Conventions

**Plugin version**: Always use the latest published version of the plugin. Before editing any code snippet that contains a version string, fetch [plugins.gradle.org/plugin/io.github.frankois944.spmForKmp](https://plugins.gradle.org/plugin/io.github.frankois944.spmForKmp) to confirm the current version.

**Definition of done**: Every integration task in this skill is only complete when `xcrun simctl launch` returns a PID. Gradle BUILD SUCCESSFUL is necessary but never sufficient — this is documented in `SKILL.md` and `migration.md` and must be preserved in all edits.

**Swift bridge constraints**: The ObjC boundary is the central constraint of the entire skill. `@objcMembers public class ... : NSObject` is required for any Swift type crossing to Kotlin. Typed throws (`throws(MyError)`) cause a compile error in Xcode 26.4+; use untyped `throws` in bridge code.

**`exportToKotlin` is always explicit**: Never omit it; the detection logic lives in `exporting.md`. When in doubt, `false` is the safe default.

## Editing Guidelines

- Code examples in references must be complete and production-ready — no partial snippets.
- The 9 non-negotiable rules in `SKILL.md` are load-bearing. Changes to rules must be consistent across all affected reference files.
- `evals/evals.json` contains assertions against expected skill outputs. When adding new patterns or rules, consider whether the eval cases need updating.
