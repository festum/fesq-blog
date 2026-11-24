---
title: "Shizuku Fork Zoo - Almost None of Them Are Worth Switching For"
description: "Seven Shizuku forks, one axis that decides whether any of them is an upgrade: what happens after a reboot."
pubDate: 2026-11-22
tags: [android, shizuku, engineering, android-17]
draft: false
---

## What happens after a reboot

Android 17 made reboot recovery a platform problem for rootless wireless ADB.

Four of the seven Shizuku forks document a start path after a reboot. Only Nightzuku's notes explain how its own boot job behaves once Android 17 redacts the wireless-debugging settings.

Three changes in Android 17 (SDK 37) close the doors that rootless wireless ADB used to walk through. In the shipped `framework.jar`, both `Settings.Global.ADB_ENABLED` and `Settings.Global.ADB_WIFI_ENABLED` carry `@Readable(redactedValue = "0")`, so every third-party read returns 0 even while wireless debugging is running. Writing `adb_enabled`, `adb_wifi_enabled`, or `adb_allowed_connection_time` now requires `android.permission.MANAGE_DEBUGGING`, a `signature|privileged` permission that `pm grant` cannot hand out. An app targeting API 37 also needs the new `ACCESS_LOCAL_NETWORK` runtime permission before it can discover anything over mDNS or reach a local port ([Android local network permission](https://developer.android.com/privacy-and-security/local-network-permission)).

A boot job that reads `adb_wifi_enabled` to decide whether to start is therefore reading a redacted zero, concluding that nothing is up, and trying to switch the system on through a permission it cannot hold.

[Nightzuku's compatibility notes](https://github.com/kerneldroid/Nightzuku/blob/main/docs/android-17-compatibility.md) state what that leaves. On API 37+, `AdbSettings.prepareWirelessAdbForBoot()` returns `REDACTION_DEFER_TO_MDNS`, and the boot job skips `Settings.Global` completely, going straight to mDNS discovery. The pairing reader that grabs the pairing code out of the system dialog, an accessibility service, runs only inside an explicit two-minute arm window, ignores any window that is not Settings, and disables itself once it has the code or the timer expires.

The evidence for everything below is repository code, README files, and release metadata; no device was tested.

![Scatter plot of seven Shizuku forks. The horizontal axis runs from one-time setup to constant maintenance, the vertical axis from a narrow API surface to a broad one. Shizuku-Next sits at low maintenance and high breadth, ShizukuPlus at the top for breadth with mid setup effort, Nightzuku at high breadth and very low maintenance, Shevery and Shizako mid-breadth with mid setup effort, Stellar mid-setup and narrow, Porter low on both axes and high on maintenance. A filled marker means the README documents an automatic start after a reboot; a hollow marker means it documents none.](/images/shizuku-fork-zoo/axes.svg)

## What each fork actually answers

- **[Shizuku-Next](https://github.com/rushiranpise/Shizuku-Next)** (2,200 stars) is the most hands-off. It keeps Shizuku's package name and API, so existing apps find it, and it handles wireless-debugging pairing on its own, including reading the pairing code on supported devices, with boot start, a watchdog, and restart recovery. It continues thedjchi's fork and uses a different signing key, so the previous app has to be uninstalled first.
- **[ShizukuPlus](https://github.com/thejaustin/ShizukuPlus)** (1,867) is the largest toolbox: a Service Doctor, a watchdog, per-app permission control, and its own Plus API suite for developers, with virtual-machine control, a privileged storage proxy, and NPU scheduling on top of Shizuku's API. It installs under its own package so it sits beside stock Shizuku, and it ships a Compat Hub for clients that hardcode `moe.shizuku.privileged.api`, or a Drop-In build that takes stock Shizuku's place entirely. Its Android 17 work descends from Nightzuku's hidden-API layer; the port through thedjchi's fork was verified end to end on a physical Android 17 device.
- **[Nightzuku](https://github.com/kerneldroid/Nightzuku)** (359) has the Android 17 cold-boot answer, plus certificate pinning, profile isolation that keeps Knox, Secure Folder, Private Space, and clone profiles out of enumeration, and per-UID capability masks checked before any forwarded transaction.
- **[Shizako](https://github.com/cr1437/Shizako)** (136) exists for rootless external app-data backup, install queues, and a file proxy that logs what it moves. Its own package and permission names let it coexist with stock, and it removed the placeholder package that used to occupy the upstream name.
- **[Shevery](https://github.com/HmnDev-Tech/shevery)** (1,758) is built around Device Owner management and biometric gating. Its independent package and isolated permission namespace allow side-by-side installs, and its compatibility stubs for older clients can only be enabled once stock Shizuku is removed.
- **[Porter](https://github.com/d4rken-org/porter)** (332) is the smallest maintained implementation, written by the developer of SD Maid SE and Butler because the original stopped being maintained. Its README states the reboot behavior without decoration: the service stops when the device restarts, and you start it again.
- **[Stellar](https://github.com/roro2239/Stellar)** (1,932) rebuilds the permission architecture, drops `rish` and Sui support, and ships a Shizuku compatibility layer so existing clients connect unchanged. It documents a reboot path as well: boot broadcasts, an accessibility service, or Root, plus a second daemon process that restarts it if it dies.

## The cost of forking

Signing keys diverge first. Shizuku-Next cannot be installed over the fork it continues, and the uninstall takes the app's stored access list with it.

Permission namespaces diverge next. Shizako and Shevery register their own package and permission names so they can live beside other implementations, which means legacy apps that look only for `moe.shizuku.privileged.api` need a compatibility stub, and that stub cannot always be enabled while another Shizuku implementation holds the same name. Shevery's own note and ShizukuPlus's Drop-In warning say exactly this.

Accessibility is powerful and fragile. A pairing helper that reads system dialogs is an accessibility service, and the same service that reads one dialog can keep reading every dialog on screen. Nightzuku disarms its helper as soon as it has what it came for.

Upstream lag is real. ShizukuPlus and Shizuku-Next are both built on thedjchi's fork, and Shizuku-Next's setup guide still sends readers to that fork's wiki for pairing.

Security claims are uneven. Some forks document their controls with the mechanism attached, while others inherit upstream behavior and add features on top.

Meanwhile the feature list keeps growing on the projects that answer the reboot question least. A German department store food counter works the same way: you can eat your way down the whole aisle and still go home with nothing to cook.

## Pick by constraint

Shizuku-Next for one-time setup with no further involvement. Nightzuku when Android 17 cold-boot reliability is the single requirement. ShizukuPlus when you want the widest privileged API surface and accept its pairing model. Shizako when rootless app-data backup, install queues, and a logged file proxy are the point. Porter for the smallest implementation, from a developer who builds on the API in shipped apps. Shevery when Device Owner management and biometric gating matter more than automation. Stellar for a rebuilt permission system with a documented restart path.

[Stock Shizuku](https://github.com/RikkaApps/Shizuku) has 34,375 stars and no commits since June 2025. thedjchi's fork has 7,046 and stopped in July 2026. The forks fill that vacuum, and most of them fill it with settings. Judge each one on what happens after a reboot and the field gets small quickly.

Forks are tools, not upgrades. Pick the one that solves your constraint and ignore the feature checklist.
