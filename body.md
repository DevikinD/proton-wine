Consolidated build of all ten Proton / GE-Proton layers at **versionCode 9**, built on the [`build-bionic-layers-20260917-wayland-v8`](https://github.com/The412Banner/proton-wine/releases/tag/build-bionic-layers-20260917-wayland-v8) sources. Every Wine 11 layer is its v8 build plus **touch on Wayland**, **opt-in ntsync**, two **crypto crash fixes** and a dormant **Steam bridge**. The two **Wine 10** layers — left out of v8 — now get **Wayland and HDR10** for the first time, backported onto Wine 10. Each layer is a single `.wcp` that runs on both 4 KB- and 16 KB-page devices.

> ✅ **This is the current release and the in-app catalog default.** These wcp install into a new coexisting `<version>-arm64ec-9` slot (and `-x86_64-9`) next to the older ones — nothing is overwritten. A container already on a v8 (or, for Proton 10, v7) layer of the same line is offered this as an **in-place update** with a revert snapshot.
>
> **Device-tested** on an AYANEO Pocket FIT (Adreno 750): **Proton 11.0-2** — *DiRT Showdown* on default esync and again with `WINENTSYNC=1` (userspace ntsync confirmed live), *Counter-Strike: Source* through SteamLite (online server browser) and Goldberg, and a Raw Steam launch, with the Steam bridge confirmed dormant in all three; **Proton 10.0-4** on Wayland and X11 and **GE-Proton 10.0-34** on Wayland — cursor and *Insane 2*; the remaining layers were each launched and checked. **Not yet tested:** touch inside a Windows game — the touch path from the app to the guest is in place, but no touch-aware game has been tried on these layers.

## What's new

- 👆 **Touch on Wayland** — every finger now reaches the game as its own Windows pointer (`WM_POINTERDOWN/UPDATE/UP`, one id per finger), so multi-touch and pinch work in touch-aware games, and older games that use `WM_TOUCH` get it too. Needs the app in **Touchscreen** mode on a Wayland container. *(Wine 11 arm64ec layers, CachyOS keeps its own upstream version of the same thing, and both Wine 10 layers.)*
- ⚡ **ntsync — opt-in, off by default** — a faster, more Windows-accurate way for games to coordinate their threads, using GameNative's userspace ntsync (no kernel driver needed). **Nothing changes unless you turn it on:** without the switch every layer runs on esync exactly as in v8. *(All eight Wine 11 layers, arm64ec and x86_64.)* See **Turning on ntsync** below.
- 🔐 **Two crypto crash fixes (`rsaenh`)** — a game asking for an HMAC with a hash Wine doesn't implement no longer crashes in `CryptHashData` (seen with *Aniimo*), and a signature check now holds on to its public key so another thread destroying it mid-check can't crash the game (seen in Steam networking). The first fix is by **bl4ckh4ck5**; the second is our corrected version of his. *(All ten layers.)*
- 🌊 **Wayland + HDR10 on Proton 10.0-4 and GE-Proton 10.0-34** — the v8 Wayland work backported to Wine 10: `winewayland.drv`, the eight bundled Wayland Turnip drivers, the Bannerlator desktop protocol and HDR10 through the EDID. Two Wine-10-only problems found on device were fixed before release: the desktop locked the mouse pointer so no cursor showed, and 32-bit games on `noexec` storage failed to map their data files (*Insane 2*: *"Can't initialize file server"*). The builtin `amd_ags_x64` is not on Wine 10, so DXGI HDR there follows `DXVK_HDR`.
- 🎮 **Steam bridge (`lsteamclient`) — built in, dormant** — the piece a Windows game needs to talk to a native Android Steam client, for a future launch option next to SteamLite. It is **completely inactive** unless `WINE_LSTEAMCLIENT=1` is set, which nothing sets today; SteamLite, Goldberg and Raw launches are unchanged (verified on device). No `steam.exe` is shipped. *(Wine 11 arm64ec layers only.)*
- 🔢 **versionCode → 9** on every layer — new coexisting install slot `<version>-arm64ec-9` / `-x86_64-9`.
- ♻️ **Carried over from v8** — Wayland + HDR10 on the Wine 11 layers · Wine XP desktop (Luna taskbar, start menu with the Control Panel fix, XP window frames, `winexp.msstyles`, dark mode) · XInput update-thread fix · `RtlIsEcCode` bounds check · DirectAudio v1.3.2 · ws2_32 dual-stack DNS · nsiproxy default route · gdiplus span clamp · realized-font-handle cap `32768` · GE game-fix tiers · SD-card boot fix (`noexec` / `force_anon`) · drive-root copy fix · `C.UTF-8` locale · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender · one wcp per layer (4 KB + 16 KB pages).

## Turning on ntsync

Add this to the container's (or the game's) **environment variables**:

```
WINENTSYNC=1
```

The layer uses the kernel's `/dev/ntsync` if the device has a usable one, and otherwise GameNative's **userspace ntsync** (shared memory + futex). The Wine log then shows `ntsync: WINENTSYNC set, no usable /dev/ntsync, using userspace ntsync.` and there is no `esync: up and running` line. If ntsync can't start, the layer quietly falls back to the normal sync.

| Variable | Effect |
|---|---|
| `WINENTSYNC=1` | turn ntsync on (off when unset or `0`) |
| `PROTON_NO_NTSYNC=1` | force it off even if `WINENTSYNC=1` is set |
| `NTSYNC_SHM=<path>` | where the shared region lives (default `$TMPDIR/ntsync_userspace.v9.shm`) |
| `NTSYNC_DEBUG=1` | print statistics every 10 s |
| `NTSYNC_SPIN_ITERS=<n>` | spin before sleeping on a wait (default `0`) |
| `NTSYNC_SWEEP_INTERVAL_SEC=<n>` | how often objects of dead processes are cleaned up (default `30`) |

Limits: 16384 sync objects and 64 objects per wait. Test it per game — it is new, and some games may prefer esync. Userspace ntsync is [`GameNative/ntsync-android`](https://github.com/GameNative/ntsync-android) by **[Joshua Tam (@joshuatam)](https://github.com/joshuatam)**, pinned at `7ce6435`, LGPL-3.0; its licence ships in each layer under `share/licenses/ntsync-android/`. Not available on the two Wine 10 layers.

**Scope of the changes versus v8:** `winewayland.drv` (touch; and on Wine 10 the whole Wayland driver plus the `win32u` / `winevulkan` / `ntdll` pieces it needs), `ntdll` and `wineserver` sync (ntsync, Wine 11 only), `rsaenh`, the `lsteamclient` module and its `ntdll` loader gate (Wine 11 arm64ec only), and the per-layer CI checks. No FEX, DXVK, audio or input changes.

## Layers

<details>
<summary><b>GE-Proton 11.0-7.1</b> &nbsp;·&nbsp; arm64ec + <b>x86_64</b> · Wine 11 (bleeding-edge Valve base) · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | GloriousEggroll **[GE-Proton11-7](https://github.com/GloriousEggroll/proton-ge-custom/releases/tag/GE-Proton11-7)** game-fix tier on **ValveSoftware/wine `46b29104`** (bleeding-edge, 2026-09-15) |
| **Installs as** | `11.0-7.1-arm64ec-9` · `11.0-7.1-x86_64-9` |
| **Wayland *(arm64ec only)*** | `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol |
| **HDR10 *(arm64ec only)*** | screen peak / frame-average / black level reported to Windows through a built CTA-861.3 EDID, DXGI and DisplayConfig advanced colour, plus the builtin `amd_ags_x64` |
| **Touch *(arm64ec only)*** | **new** — `wl_touch` fingers become Windows `WM_POINTERDOWN/UPDATE/UP` (one id per finger, so multi-touch and pinch), and Wine turns those into `WM_TOUCH` for older touch-aware games |
| **ntsync (opt-in)** | **new, off by default** — `WINENTSYNC=1` turns it on (kernel `/dev/ntsync` if the device has one, otherwise userspace ntsync); unset = esync exactly as before; `PROTON_NO_NTSYNC=1` forces it off |
| **Steam bridge *(arm64ec only)*** | **new, dormant** — `lsteamclient` (64- and 32-bit) for a future native-Steam launch option; does nothing unless `WINE_LSTEAMCLIENT=1` is set |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route · gdiplus span clamp |
| **DirectAudio** | v1.3.2 (vendored source) — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `GE-proton-11.0-7.1-arm64ec.wcp` · `GE-proton-11.0-7.1-x86_64.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`noexec` / `force_anon`) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes** — `ai-limit` · `assettocorsa` · `black-desert` · `dai_xinput` · `eac` · `maplestory` · `max-payne` · `pso2` · `return-to-krondor` · `silence-starcitizen` · `vgsoh` · `WM_ACTIVATEAPP`

> ⚠️ The **x86_64** wcp carries every v9 fix **except Wayland, touch and the Steam bridge** (all arm64ec-only). ntsync is opt-in there too.
>
> ℹ️ Keeps its own install slot (`11.0-7.1`) apart from the current-base GE 11.0-7, so the two can sit side by side.

</details>

<details>
<summary><b>GE-Proton 11.0-7</b> &nbsp;·&nbsp; arm64ec + <b>x86_64</b> · Wine 11 · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | GloriousEggroll **[GE-Proton11-7](https://github.com/GloriousEggroll/proton-ge-custom/releases/tag/GE-Proton11-7)** game-fix tier on Valve **[Proton 11.0-1](https://github.com/ValveSoftware/Proton/releases/tag/proton-11.0-1)** (Wine 11.0-1) |
| **Installs as** | `11.0-7-arm64ec-9` · `11.0-7-x86_64-9` |
| **Wayland *(arm64ec only)*** | `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol |
| **HDR10 *(arm64ec only)*** | screen peak / frame-average / black level reported to Windows through a built CTA-861.3 EDID, DXGI and DisplayConfig advanced colour, plus the builtin `amd_ags_x64` |
| **Touch *(arm64ec only)*** | **new** — `wl_touch` fingers become Windows `WM_POINTERDOWN/UPDATE/UP` (one id per finger, so multi-touch and pinch), and Wine turns those into `WM_TOUCH` for older touch-aware games |
| **ntsync (opt-in)** | **new, off by default** — `WINENTSYNC=1` turns it on (kernel `/dev/ntsync` if the device has one, otherwise userspace ntsync); unset = esync exactly as before; `PROTON_NO_NTSYNC=1` forces it off |
| **Steam bridge *(arm64ec only)*** | **new, dormant** — `lsteamclient` (64- and 32-bit) for a future native-Steam launch option; does nothing unless `WINE_LSTEAMCLIENT=1` is set |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route · gdiplus span clamp |
| **DirectAudio** | v1.3.2 (vendored source) — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `GE-proton-11.0-7-arm64ec.wcp` · `GE-proton-11.0-7-x86_64.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`noexec` / `force_anon`) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes** — `maplestory` · `dai_xinput` · `eac` · `pso2` · `assettocorsa` · `silence-starcitizen` · `vgsoh` · `WM_ACTIVATEAPP` · `black-desert fullscreen` · `max-payne cpu detection` · `ai-limit dx12 compute fallback`

> ⚠️ The **x86_64** wcp carries every v9 fix **except Wayland, touch and the Steam bridge** (all arm64ec-only). ntsync is opt-in there too.

</details>

<details>
<summary><b>GE-Proton 11.0-6</b> &nbsp;·&nbsp; arm64ec · Wine 11 · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | GloriousEggroll **[GE-Proton11-6](https://github.com/GloriousEggroll/proton-ge-custom/releases/tag/GE-Proton11-6)** game-fix tier on Valve **[Proton 11.0-1](https://github.com/ValveSoftware/Proton/releases/tag/proton-11.0-1)** (Wine 11.0-1) |
| **Installs as** | `11.0-6-arm64ec-9` |
| **Wayland** | `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol |
| **HDR10** | screen peak / frame-average / black level reported to Windows through a built CTA-861.3 EDID, DXGI and DisplayConfig advanced colour, plus the builtin `amd_ags_x64` |
| **Touch** | **new** — `wl_touch` fingers become Windows `WM_POINTERDOWN/UPDATE/UP` (one id per finger, so multi-touch and pinch), and Wine turns those into `WM_TOUCH` for older touch-aware games |
| **ntsync (opt-in)** | **new, off by default** — `WINENTSYNC=1` turns it on (kernel `/dev/ntsync` if the device has one, otherwise userspace ntsync); unset = esync exactly as before; `PROTON_NO_NTSYNC=1` forces it off |
| **Steam bridge** | **new, dormant** — `lsteamclient` (64- and 32-bit) for a future native-Steam launch option; does nothing unless `WINE_LSTEAMCLIENT=1` is set |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route · gdiplus span clamp |
| **DirectAudio** | v1.3.2 (vendored source) — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `GE-proton-11.0-6-arm64ec.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`noexec` / `force_anon`) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes** — `maplestory` · `dai_xinput` · `eac` · `pso2` · `assettocorsa` · `silence-starcitizen` · `vgsoh` · `WM_ACTIVATEAPP`

</details>

<details>
<summary><b>GE-Proton 11.0-5</b> &nbsp;·&nbsp; arm64ec · Wine 11 · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | GloriousEggroll **[GE-Proton11-5](https://github.com/GloriousEggroll/proton-ge-custom/releases/tag/GE-Proton11-5)** game-fix tier on Valve **[Proton 11.0-1](https://github.com/ValveSoftware/Proton/releases/tag/proton-11.0-1)** (Wine 11.0-1) |
| **Installs as** | `11.0-5-arm64ec-9` |
| **Wayland** | `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol |
| **HDR10** | screen peak / frame-average / black level reported to Windows through a built CTA-861.3 EDID, DXGI and DisplayConfig advanced colour, plus the builtin `amd_ags_x64` |
| **Touch** | **new** — `wl_touch` fingers become Windows `WM_POINTERDOWN/UPDATE/UP` (one id per finger, so multi-touch and pinch), and Wine turns those into `WM_TOUCH` for older touch-aware games |
| **ntsync (opt-in)** | **new, off by default** — `WINENTSYNC=1` turns it on (kernel `/dev/ntsync` if the device has one, otherwise userspace ntsync); unset = esync exactly as before; `PROTON_NO_NTSYNC=1` forces it off |
| **Steam bridge** | **new, dormant** — `lsteamclient` (64- and 32-bit) for a future native-Steam launch option; does nothing unless `WINE_LSTEAMCLIENT=1` is set |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route · gdiplus span clamp |
| **DirectAudio** | v1.3.2 (vendored source) — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `GE-proton-11.0-5-arm64ec.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`noexec` / `force_anon`) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes** — `battlenet` · `maplestory` · `dai_xinput` · `eac` · `pso2` · `assettocorsa` · `silence-starcitizen` · `vgsoh` · `WM_ACTIVATEAPP`

</details>

<details>
<summary><b>GE-Proton 11.0-3</b> &nbsp;·&nbsp; arm64ec · Wine 11 · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | GloriousEggroll **[GE-Proton11-3](https://github.com/GloriousEggroll/proton-ge-custom/releases/tag/GE-Proton11-3)** game-fix tier on Valve **[Proton 11.0-1](https://github.com/ValveSoftware/Proton/releases/tag/proton-11.0-1)** (Wine 11.0-1) |
| **Installs as** | `11.0-3-arm64ec-9` |
| **Wayland** | `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol |
| **HDR10** | screen peak / frame-average / black level reported to Windows through a built CTA-861.3 EDID, DXGI and DisplayConfig advanced colour, plus the builtin `amd_ags_x64` |
| **Touch** | **new** — `wl_touch` fingers become Windows `WM_POINTERDOWN/UPDATE/UP` (one id per finger, so multi-touch and pinch), and Wine turns those into `WM_TOUCH` for older touch-aware games |
| **ntsync (opt-in)** | **new, off by default** — `WINENTSYNC=1` turns it on (kernel `/dev/ntsync` if the device has one, otherwise userspace ntsync); unset = esync exactly as before; `PROTON_NO_NTSYNC=1` forces it off |
| **Steam bridge** | **new, dormant** — `lsteamclient` (64- and 32-bit) for a future native-Steam launch option; does nothing unless `WINE_LSTEAMCLIENT=1` is set |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route · gdiplus span clamp |
| **DirectAudio** | v1.3.2 (vendored source) — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `GE-proton-11.0-3-arm64ec.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`noexec` / `force_anon`) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes** — `battlenet` · `maplestory` · `dai_xinput` · `eac` · `pso2` · `assettocorsa` · `silence-starcitizen` · `vgsoh` · `WM_ACTIVATEAPP`

</details>

<details>
<summary><b>Proton 11.0-2</b> &nbsp;·&nbsp; arm64ec + <b>x86_64</b> · Wine 11 · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | Valve **Proton 11.0-2** (Wine 11.0) — the line every v9 feature was developed and device-proven on |
| **Installs as** | `11.0-2-arm64ec-9` · `11.0-2-x86_64-9` |
| **Wayland *(arm64ec only)*** | `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol |
| **HDR10 *(arm64ec only)*** | screen peak / frame-average / black level reported to Windows through a built CTA-861.3 EDID, DXGI and DisplayConfig advanced colour, plus the builtin `amd_ags_x64` |
| **Touch *(arm64ec only)*** | **new** — `wl_touch` fingers become Windows `WM_POINTERDOWN/UPDATE/UP` (one id per finger, so multi-touch and pinch), and Wine turns those into `WM_TOUCH` for older touch-aware games |
| **ntsync (opt-in)** | **new, off by default** — `WINENTSYNC=1` turns it on (kernel `/dev/ntsync` if the device has one, otherwise userspace ntsync); unset = esync exactly as before; `PROTON_NO_NTSYNC=1` forces it off |
| **Steam bridge *(arm64ec only)*** | **new, dormant** — `lsteamclient` (64- and 32-bit) for a future native-Steam launch option; does nothing unless `WINE_LSTEAMCLIENT=1` is set |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route · gdiplus span clamp |
| **DirectAudio** | v1.3.2 (vendored source) — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `proton-11.0-2-arm64ec.wcp` · `proton-11.0-2-x86_64.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`noexec` / `force_anon`) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes** — none (plain Proton)

> ⚠️ The **x86_64** wcp carries every v9 fix **except Wayland, touch and the Steam bridge** (all arm64ec-only). ntsync is opt-in there too. Proton 11 under box64 also still does not render a window — the arm64ec layer is the one to use.

</details>

<details>
<summary><b>Proton 11.0-1</b> &nbsp;·&nbsp; arm64ec · Wine 11 · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | Valve **[Proton 11.0-1](https://github.com/ValveSoftware/Proton/releases/tag/proton-11.0-1)** (Wine 11.0-1), stock |
| **Installs as** | `11.0-1-arm64ec-9` |
| **Wayland** | `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol |
| **HDR10** | screen peak / frame-average / black level reported to Windows through a built CTA-861.3 EDID, DXGI and DisplayConfig advanced colour, plus the builtin `amd_ags_x64` |
| **Touch** | **new** — `wl_touch` fingers become Windows `WM_POINTERDOWN/UPDATE/UP` (one id per finger, so multi-touch and pinch), and Wine turns those into `WM_TOUCH` for older touch-aware games |
| **ntsync (opt-in)** | **new, off by default** — `WINENTSYNC=1` turns it on (kernel `/dev/ntsync` if the device has one, otherwise userspace ntsync); unset = esync exactly as before; `PROTON_NO_NTSYNC=1` forces it off |
| **Steam bridge** | **new, dormant** — `lsteamclient` (64- and 32-bit) for a future native-Steam launch option; does nothing unless `WINE_LSTEAMCLIENT=1` is set |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route · gdiplus span clamp |
| **DirectAudio** | v1.3.2 (vendored source) — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `proton-11.0-1-arm64ec.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`noexec` / `force_anon`) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes** — none (plain Proton)

</details>

<details>
<summary><b>Proton-CachyOS 11.0-20260703</b> &nbsp;·&nbsp; arm64ec + <b>x86_64</b> · Wine 11 (CachyOS) · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | **[wine-cachyos](https://github.com/CachyOS/wine-cachyos) `b5f2dc7b590`** (release `cachyos-11.0-20260703-slr`: Valve Proton experimental-11.0 + the CachyOS patch set) + our full Android stack |
| **Installs as** | `11.0-20260703-arm64ec-9` · `11.0-20260703-x86_64-9` |
| **Wayland *(arm64ec only)*** | `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol (plus CachyOS's own upstream `alpha-modifier` / `color-management` / `content-type` additions) |
| **HDR10 *(arm64ec only)*** | screen peak / frame-average / black level reported to Windows through a built CTA-861.3 EDID, DXGI and DisplayConfig advanced colour, plus the builtin `amd_ags_x64` |
| **Touch *(arm64ec only)*** | CachyOS's own upstream `winewayland` touch support (same `WM_POINTER*` design) — kept as is |
| **ntsync (opt-in)** | **new, off by default** — `WINENTSYNC=1` turns it on (kernel `/dev/ntsync` if the device has one, otherwise userspace ntsync); unset = esync exactly as before; `PROTON_NO_NTSYNC=1` forces it off |
| **Steam bridge *(arm64ec only)*** | **new, dormant** — `lsteamclient` (64- and 32-bit) for a future native-Steam launch option; does nothing unless `WINE_LSTEAMCLIENT=1` is set |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route · gdiplus span clamp |
| **DirectAudio** | v1.3.2 (vendored source) — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `proton-cachyos-11.0-20260703-arm64ec.wcp` · `proton-cachyos-11.0-20260703-x86_64.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`noexec` / `force_anon`) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes** — none (CachyOS patch set, not a GE tier)

> ⚠️ The **x86_64** wcp carries every v9 fix **except Wayland, touch and the Steam bridge** (all arm64ec-only). ntsync is opt-in there too.
>
> ℹ️ On Wayland this layer needs xdg-shell v3 from the compositor, which Bannerlator does not offer yet — use it on **X11** for now.

</details>

<details>
<summary><b>Proton 10.0-4</b> &nbsp;·&nbsp; arm64ec · <b>Wine 10</b> · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | Stock Valve **[Proton 10.0-4](https://github.com/ValveSoftware/Proton/releases/tag/proton-10.0-4)** (Wine 10) — plain Proton, no GE game-fixes |
| **Installs as** | `10.0-4-arm64ec-9` |
| **Wayland** | **new on Proton 10** — `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol |
| **HDR10** | **new on Proton 10** — screen peak / frame-average / black level reported through a built CTA-861.3 EDID and DisplayConfig advanced colour; DXGI HDR follows `DXVK_HDR` (no builtin `amd_ags_x64` on Wine 10) |
| **Touch** | **new** — `wl_touch` fingers become Windows `WM_POINTERDOWN/UPDATE/UP` (one id per finger, so multi-touch and pinch), and Wine turns those into `WM_TOUCH` for older touch-aware games |
| **Proton 10 Wayland fixes** | **new** — the desktop no longer locks the mouse pointer at startup (cursor visible) · data files on `noexec` storage (e.g. games on shared storage / SD card) map again for 32-bit games — fixes *"Can't initialize file server"* in Insane 2 |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route (Wine 10's gdiplus has no span assertion — no clamp needed) |
| **DirectAudio** | v1.3.2 **Wine-10 ABI port** — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `proton-10.0-4-arm64ec.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`force_anon`, GameNative's original Wine-10 diff) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes** — none (plain Proton)

> ℹ️ Jumps from v7 straight to v9 (it was left out of v8). No ntsync and no Steam bridge on Wine 10.

</details>

<details>
<summary><b>GE-Proton 10.0-34</b> &nbsp;·&nbsp; arm64ec · <b>Wine 10</b> · versionCode <code>9</code></summary>

<br>

| | |
|---|---|
| **Base** | Valve **[Proton 10.0-4](https://github.com/ValveSoftware/Proton/releases/tag/proton-10.0-4)** (Wine 10) with the **[GE-Proton10-34](https://github.com/GloriousEggroll/proton-ge-custom/releases/tag/GE-Proton10-34)** game-fix tier layered on (GE's game-patch files only, not GE's Wine tree) |
| **Installs as** | `10.0-34-arm64ec-9` |
| **Wayland** | **new on Proton 10** — `winewayland.drv` (unix `.so` + `aarch64-windows` and `i386-windows` PE) · 8 bundled Wayland Turnip drivers (`plain`, `a7xx`, `a8xx`, `a8xx-perf`, `a8xx-gen8`, `a8xx-smxz`, `a8xx-white`, `a8xx-upstream`) · `banner-desktop-v1` protocol |
| **HDR10** | **new on Proton 10** — screen peak / frame-average / black level reported through a built CTA-861.3 EDID and DisplayConfig advanced colour; DXGI HDR follows `DXVK_HDR` (no builtin `amd_ags_x64` on Wine 10) |
| **Touch** | **new** — `wl_touch` fingers become Windows `WM_POINTERDOWN/UPDATE/UP` (one id per finger, so multi-touch and pinch), and Wine turns those into `WM_TOUCH` for older touch-aware games |
| **Proton 10 Wayland fixes** | **new** — the desktop no longer locks the mouse pointer at startup (cursor visible) · data files on `noexec` storage (e.g. games on shared storage / SD card) map again for 32-bit games — fixes *"Can't initialize file server"* in Insane 2 |
| **Crypto fixes** | **new** — two `rsaenh` crash fixes: unsupported HMAC inner hash (`CryptHashData` null crash) · public key held during `CPVerifySignature` (Steam networking race) |
| **ntdll fix** | `RtlIsEcCode` bounds check (Denuvo unwind loop) |
| **EA fixes** | ws2_32 dual-stack DNS · nsiproxy default route (Wine 10's gdiplus has no span assertion — no clamp needed) |
| **DirectAudio** | v1.3.2 **Wine-10 ABI port** — opt-in via registry `Audio=directaudio`; mic capture opt-in via `BANNER_AUDIO_DIRECT_MIC=1` |
| **XInput fix** | update thread survives transient wait failures (controllers no longer die mid-game) |
| **Wine XP desktop** | Luna taskbar + start menu (with the Control Panel fix) · XP window frames · `winexp.msstyles` visual style — Blue / Olive Green / Silver, navy / moss / graphite in dark mode |
| **Assets** | `GE-proton-10.0-34-arm64ec.wcp` (4 KB + 16 KB pages) |

**Android compatibility fixes** — SD-card boot (`force_anon`, Wine-10 diff) · drive-root copy · `C.UTF-8` locale
**Runtime** — realized-font-handle cap `32768` · `WINEVMEMMAXSIZE` cap · fast-yield gate · FEX-unixlib loader · XRandR / XRender
**Build** — `-g0 -O2` release build, `llvm-strip` on both the PE DLLs/EXEs and the unix `.so` loaders · zstd-compressed `.wcp` · ccache in CI (build speed only, not in the layer)
**Inherited bionic base** — the Winlator-bionic / GameNative Android patch set every layer is built on: esync/fsync, winex11 driver (window/keyboard/mouse/OpenGL/bitblt), preloader, clipboard, winemenubuilder, MIDI, DNS resolver, wow64 syscall path
**GE game-fixes (10 — superset)**
- **GE-Proton10-34 tier:** `assettocorsa` · `dai_xinput` · `eac` · `pso2` · `silence-starcitizen` · `vgsoh`
- **plus our extras:** `battlenet` · `maplestory-charprev` · `maplestory-stickykeys` · `WM_ACTIVATEAPP`

> ℹ️ Jumps from v7 straight to v9 (it was left out of v8). No ntsync and no Steam bridge on Wine 10.

</details>

## Credits

- **Userspace ntsync — [Joshua Tam (@joshuatam)](https://github.com/joshuatam), [GameNative](https://github.com/GameNative).** The shared-memory + futex ntsync implementation, [`ntsync-android`](https://github.com/joshuatam/ntsync-android) (LGPL-3.0; GameNative's copy: [`GameNative/ntsync-android`](https://github.com/GameNative/ntsync-android)), pinned in these layers at [`7ce6435`](https://github.com/GameNative/ntsync-android/commit/7ce6435e5979b1cb5341aa4b299f31e8937fe121). Our Wine wiring follows his GameNative Proton 11.0-2 integration, adapted so esync stays the default and ntsync is opt-in:
  - [`962a379`](https://github.com/GameNative/proton-wine/commit/962a379708a762df79b1aac99f2e0e01a3a53809) — Proton 11.0-2 port with userspace ntsync
  - [`d67ac1e`](https://github.com/GameNative/proton-wine/commit/d67ac1e0d83c9c6bb6a43bb4dc8459bc45d6b937) — runtime kernel / userspace backend detection
  - [`0971187`](https://github.com/GameNative/proton-wine/commit/0971187883d7488b1686770015f6e6fd6bf342da) — force-userspace switch
- **Steam bridge — [Joshua Tam (@joshuatam)](https://github.com/joshuatam), [GameNative](https://github.com/GameNative).** GameNative's lsteamclient integration for their native Android Steam client, [`dafe413`](https://github.com/GameNative/proton-wine/commit/dafe413ae06a11ce0cebea2bc1681b16dbc34c84), is the model for ours; its `owned_dlcs` DLC-ownership override is ported from that commit. The `lsteamclient` module itself is [Valve's](https://github.com/ValveSoftware/Proton) (Proton 11.0-2).
- **Crypto crash fixes — [bl4ckh4ck5 (@hackoclipse)](https://github.com/hackoclipse).** [`b4fc579`](https://github.com/hackoclipse/proton-wine/commit/b4fc579416adb0a8d343496caf0885e34ed8d8cd) — HMAC with an unsupported inner hash (taken as is; adapted to Wine 10's hashing on the two Proton 10 layers) · [`a0d20d6`](https://github.com/hackoclipse/proton-wine/commit/a0d20d6c2c60bbcaac7a64e374ffe148297ab6f5) — hold the public key during `CPVerifySignature` (our version takes the reference under the handle-table lock and releases only the reference, leaving the caller's handle in place).
- **Inherited base — [GameNative](https://github.com/GameNative/proton-wine) and Winlator-bionic.** The Android patch set every layer is built on, including the Wine-10 SD-card boot fix; plus [Valve](https://github.com/ValveSoftware/Proton), [GloriousEggroll](https://github.com/GloriousEggroll/proton-ge-custom) (GE game-fix tiers) and [CachyOS](https://github.com/CachyOS/wine-cachyos) for the bases, and Etaash Mathamsetty for upstream `winewayland` touch in the CachyOS layer.


---

*versionCode 9 · built 2026-09-30 · current release and in-app catalog default. Report problems against the layer name and slot shown on the container card.*
