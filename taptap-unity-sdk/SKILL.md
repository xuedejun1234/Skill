---
name: taptap-unity-sdk
description: Integrate and troubleshoot TapTap Unity SDK login, compliance, age checks, and compliant simulated purchases; use for TapSDK package, initialization, Android/iOS build, and scene-gating issues.
---

# TapTap Unity SDK

Use this skill for the C# and Unity project side of TapTap SDK work. Keep TapTap Developer Center configuration as a separate operator task; do not claim a setting is correct merely because Unity compiles.

## Project precedent

This project previously used Unity 2022.3.62f3c1 with TapSDK Core, Login, and Compliance 4.11.1, and Android application ID `com.tq.bfxyx`. Treat these as historical project facts, not universal defaults. Confirm the current project's Unity version, package manifest, target platform, final application ID, and SDK package versions before changing anything.

## Workflow

1. Inspect `Packages/manifest.json`, Package Manager, `GameSDKManager`, login flow, Player Settings, and the actual build target. Check for duplicate SDK installations before importing packages.
2. Keep TapSDK Core, Login, and Compliance on the same compatible version. For this project's known 4.11.1 baseline, official Git dependencies use `/Core#4.11.1`, `/Login#4.11.1`, and `/Compliance#4.11.1`. Confirm current official docs and the installed package API before applying this to another version.
3. Put SDK initialization in one persistent manager. Initialize Core with the correct app `clientId` and `clientToken`, configure Compliance, register its callback, and prevent Login/Compliance calls until initialization is ready according to that version's API. Never add arbitrary delays as a readiness fix.
4. After TapTap login, use the TapTap account's `openId` as the compliance `userIdentifier`; do not substitute a local CSV account ID. Call compliance startup once and wait for its callback before allowing the main scene. Handle restriction, network/configuration, logout, account-switch, and user-cancel callbacks distinctly.
5. For a purchase flow, check the TapTap compliance limit before granting simulated currency. Keep monetary amounts in the units required by the installed API (the current official Unity guide documents fen). Only submit a completed transaction; do not report a canceled or failed purchase. Do not implement a competing age-limit table when TapTap is the authority.
6. Validate in layers: C# compilation, SDK initialization logs, login result, compliance callback, Android/iOS build, then device behavior. Capture the full first relevant exception and surrounding Editor/player logs; do not diagnose from a single secondary warning.
7. Protect credentials. Do not print or paste `clientToken`, keystore passwords, or signing secrets in logs, chat, Git, or generated troubleshooting notes. Avoid modifying TapTap Developer Center or release settings unless the user explicitly asks.

Read [references/integration-and-troubleshooting.md](references/integration-and-troubleshooting.md) for the concrete failures encountered in the SanXiao project and their checks.
