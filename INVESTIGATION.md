# Investigation: NVIDIA 591+ OLED Dimming Regression

This document explains the bug this fork addresses, the user-mode paths that were measured and ruled out, why the shipped fix uses `SetDeviceGammaRamp` with a slider remap, and a related half-broken feature (GameVisual color presets in dGPU mode).

---

## The bug

| | |
|---|---|
| Hardware affected | NVIDIA-dGPU ROG / ProArt laptops with OLED panels in **dGPU / Ultimate** mode (mux set to direct dGPU). **Directly tested in this fork:** RTX 40-series — ROG Zephyrus G16 (GU603VVI, GA605WV), ROG Strix G16 (G614PM), ProArt H7604JI, ROG Strix SCAR 17 (G733PY), ROG G634JZ (multiple users in [g-helper issues #4764, #4919, #5042, #5093](https://github.com/seerge/g-helper/issues?q=is%3Aissue+591) and [discussion #5228](https://github.com/seerge/g-helper/discussions/5228)). **Also reported affected by the wider community** (not directly tested by this fork): RTX 30-series and RTX 50-series ROG/ProArt laptops — same -104 NVAPI_NOT_SUPPORTED symptom on the same NVIDIA 59x driver branch, in community comments across the g-helper issue tracker, NVIDIA developer forums, and the [novideo_srgb issue tracker](https://github.com/ledoge/novideo_srgb/issues). The gate appears not to be RTX 40-specific. |
| Driver versions affected | NVIDIA 591.44 and every later driver tested up to and including **596.49** |
| Symptom | OLED brightness slider in upstream g-helper (and Armoury Crate, and ASCI itself) silently does nothing. AsusSplendid's process logs "succeed" but the panel stays at full brightness. |
| Root cause | The single NVAPI call `NvAPI_GPU_SetColorSpaceConversion` (function ID `0xFCABD23A`) returns **NVAPI_NOT_SUPPORTED (-104)** when called for the internal eDP panel via the dGPU. This is the call AsusSplendid's `SetNVGammaRamp` uses to write a 3×4 RGB scale matrix to the GPU output stage. |

The bug is **not** in AsusSplendid's code or its config files — it is gated at NVIDIA's driver-private API surface. Cross-confirmed by an unrelated tool, [novideo_srgb](https://github.com/ledoge/novideo_srgb/issues/129), which uses the same NVAPI function for monitor calibration and reports the identical `-104` regression on driver 595.71+.

Key evidence (reproducible on affected hardware):

| Probe | Result | Interpretation |
|---|---|---|
| `NvAPI_Initialize` / `EnumPhysicalGPUs` / `GPU_GetConnectedDisplayIds` | OK; finds internal panel at displayId `0x80061080`, connector `DisplayPort/eDP`, active, OS-visible, physically connected | Enumeration works; the dGPU and the internal panel are discoverable |
| `NvAPI_GPU_GetColorSpaceConversion(displayId)` | OK; returns current matrix data | API surface is alive for read |
| `NvAPI_GPU_SetColorSpaceConversion(displayId, csc_v2_identity)` | **-104 NVAPI_NOT_SUPPORTED** | Even a no-op identity write is rejected |
| `NvAPI_GPU_SetColorSpaceConversion(displayId, exact_data_returned_by_Get)` | **-104 NVAPI_NOT_SUPPORTED** | Writing back the *exact value the driver just produced* is rejected — proves it's not a struct-version, displayId, capability, or value-range mismatch. The write entry point itself is gated. |
| `NvAPI_GPU_GetDitherControl(displayId)` | OK | Other display-side APIs for the same displayId still work |

---

## All user-mode paths investigated

Every plausible mechanism for restoring linear panel dim was tested on the affected hardware (Win11 25H2, driver 596.49):

| # | Mechanism | Outcome |
|--:|---|---|
| 1 | `NvAPI_GPU_SetColorSpaceConversion` (Splendid's original NVAPI call) | **Broken** — returns -104 NVAPI_NOT_SUPPORTED on 591+ for the internal panel in dGPU mode |
| 2 | `SetDeviceGammaRamp` with linear ramp at gain ≥ ~0.50 | **Works** — naturally accepted by the Windows ICM driver. This is the shipped path. |
| 3 | `SetDeviceGammaRamp` with linear ramp at gain < ~0.50 | **Clamped** — Windows silently substitutes the previous accepted ramp for any value > 32,768 from identity at `i*256` |
| 4 | `GdiICMGammaRange = 256` registry tweak (documented clamp-disable) | **No effect on Win11 25H2** — round-trip test shows clamp still active with the registry value present. (Works on Win10 and earlier Win11 builds per f.lux/Redshift user reports — Microsoft appears to have hardened the clamp on 25H2.) |
| 5 | `D3DKMTSetGammaRamp` with RGB256x3x16 LUT (kernel-mode thunk in `gdi32.dll`) | **Returns NTSTATUS=0 but no visible effect** — the kernel layer either clamps independently or shadow-state masks the result; either way, panel doesn't dim |
| 6 | `D3DKMTSetGammaRamp` with MATRIX_3x4 type (Splendid-style 3×4 matrix via kernel) | **STATUS_INVALID_PARAMETER** — NVIDIA's display driver on this 596.49 build does not implement the MATRIX ramp types |
| 7 | `D3DKMTSetGammaRamp` with MATRIX_V2 (float matrix) | **STATUS_INVALID_PARAMETER** — same |
| 8 | `NvColorSetGammaRamp` / `NvColorSetGammaRampEx` exports from `nvcpl.dll` | **Stub functions** — disassembly at the export RVAs shows both log `"NOT IMPLEMENTED"` to a diagnostic stream and return. Dead exports kept for ABI compatibility. |
| 9 | `NvCplApiSetSetting(NVCPLAPI_SETTING_DESKTOP_PARAMETRIC)` — the API the NVIDIA App's Brightness slider uses | **Wrong shape** — only does brightness/contrast/gamma knobs (range 80..120), not uniform RGB scale. Slid down produces "grey becomes black but white stays white" gamma-curve squash. No NVCPLAPI setting accepts an arbitrary CSC matrix or LUT. |
| 10 | `MagSetFullscreenColorEffect` (Windows Magnification API) | **Works system-wide, no per-monitor option** — applies a 5×5 RGB matrix at the DWM compositor. Dims all monitors equally; cursor not reached (cursor is post-DWM). |
| 11 | Per-monitor topmost transparent layered window over the laptop panel | **Works for non-cursor content; cursor not dimmed, true-FSE games invisible, blocks taskbar auto-hide** — alpha-blended black with opacity = 1−gain is mathematically uniform RGB scale, but DWM-stage only. |
| 12 | ICC profile with custom VCG curve via `WcsAssociateColorProfileWithDevice` + `WcsSetDefaultColorProfile` | **VCG isn't applied at runtime** — profile install + associate + setdefault succeed, but Windows' calibration loader (which would push the VCG to the LUT) only runs at sign-in / display configuration change, not on runtime profile swaps. Even if forced, the loader's LUT-write path goes through `SetDeviceGammaRamp` and inherits the same clamp. (GameVisual works at runtime because it uses ICC's color matrix / TRC tags — Half A of the profile, applied per-window by color-aware code — not the VCG.) |
| 13 | D3D11 "wake context" — create a 1×1 hidden DXGI swap chain on the dGPU + call `Present(0,0)`, then call `NvAPI_GPU_SetColorSpaceConversion` | **Wake theory falsified.** With a live D3D11 device + DXGI swap chain bound to the dGPU and `Present(0,0)` returning `S_OK`, `SetColorSpaceConversion` still returns the identical `-104 NVAPI_NOT_SUPPORTED` (bit-for-bit the same status code, with no variance across runs). This rules out the speculation that the -104 is a power-gating / driver-idle quirk that an active D3D context would lift. The asymmetry between Get (succeeds) and Set (fails), the per-function selectivity (other display-side APIs work), and the per-display selectivity (the function works for external HDMI displays) together fingerprint a deterministic code-level gate inside the driver — almost certainly an `if (display.IsInternalPanel() && gpu.IsLaptopDgpuDirect()) return NVAPI_NOT_SUPPORTED;` early-return added in 591.x. No process-side context can influence it. |

The shipped fix uses path #2 (`SetDeviceGammaRamp` with a linear ramp), with the slider 0..100 remapped to dim level 52..100 so every position falls inside the ICM clamp's naturally-accepted window. Cost: roughly the bottom 20% of Splendid's original brightness range is unreachable — Splendid's slider-0 reaches linear gain 0.40 (range span 0.60), this fix reaches gain 0.52 (range span 0.48), so the 0.12 gap is 0.12 / 0.60 ≈ 20% of Splendid's range. Benefit: every slider position is responsive; whites and cursor dim correctly; works in fullscreen games.

---

## Mathematical equivalence to Splendid

Splendid sets a 3×4 RGB color matrix via NVAPI CSC; this fix sets a 256-entry × 3-channel × 16-bit LUT via GDI. For a uniform scale (which is what both apply), the two representations are interchangeable: a diagonal CSC matrix with `diag = g` is equivalent to a LUT with `lut[i] = i × g × 257`. Both produce `out_pixel = src_pixel × g` per channel.

The visible difference is therefore not in the math but in the brightness *range*: Splendid spans g = 0.40..1.00; this fix spans g = 0.52..1.00 because the Windows ICM clamp blocks gains below ~0.50 (see next section). The unreachable g = 0.40..0.52 strip is about 20% of Splendid's total range.

Both transforms apply at the **GPU display output / LUT stage**, downstream of DWM composition and the hardware cursor overlay, which is why both:
- Dim the cursor
- Affect only the laptop panel (we target it by HDC; other monitors keep identity)
- Are unaffected by DWM-composited / FSO / borderless games (the modern Win11 default)

The two paths differ on **true fullscreen-exclusive D3D games**, though: Splendid's CSC matrix lived at a stage independent of the OS gamma LUT, so a game's `IDXGIOutput::SetGammaControl(identity)` call at swapchain creation didn't touch it. This fix uses the OS gamma LUT, which a true-FSE game's swapchain init does reset — so the dim is wiped when such a game starts. Nudging the brightness slider once after the game has launched re-applies our ramp. Modern Win11 games almost never trigger this; it's mostly older or D3D9-era titles that opt out of Fullscreen Optimizations.

---

## Why gains below ~0.50 are unreachable from user-mode

The Windows ICM clamp lives inside `gdi32!SetDeviceGammaRamp`. For each of the 256 × 3 channel entries in the ramp, the OS computes the deviation from identity (`expected[i] = i * 256`) and rejects values where `|requested - expected| > 32768`.

For a uniform linear scale by gain `g`, the maximum deviation occurs at `i = 255`:

```
|65535·g − 65280|  ≈  65535 · (1 − g)
```

For this to be `≤ 32768`, we need `g ≥ ~0.498`. Below that, `SetDeviceGammaRamp` returns success but the OS silently substitutes the previously accepted ramp — that's why a naive linear dim "only works above 50%" and below that the slider appears to do nothing.

This 32-KB threshold is hard-coded into the gdi32 user-mode wrapper. The documented `GdiICMGammaRange = 256` registry value used to disable it (and still does on Win10 and older Win11 builds) is ineffective on Windows 11 25H2 — confirmed empirically. Even the kernel-mode thunk `D3DKMTSetGammaRamp` shows no visible effect when fed values below the clamp window, suggesting the restriction is reproduced or shadowed at multiple layers in 25H2.

NVAPI's `NvAPI_GPU_SetColorSpaceConversion` was Splendid's escape route from this clamp — it lived at a NVIDIA-private CSC stage that bypassed all OS gamma-ramp validation. With that API gated on 591+, no other user-mode path bypasses the clamp on this Windows build.

---

## Related: GameVisual color presets are also half-broken in dGPU mode

The same `NvAPI_GPU_SetColorSpaceConversion` regression explains a sibling user-visible issue: **GameVisual mode switches (Racing / Cinema / Vivid / Eyecare / E-Reading / RTS-RPG / FPS / Scenery / Disabled) produce only subtle color shifts in dGPU mode but dramatic, expected shifts in iGPU mode.**

A GameVisual mode switch in Splendid is implemented as two layered operations:

| Layer | Mechanism | Status on dGPU 591+ |
|---|---|---|
| **A — base color gamut** (sRGB / DCI-P3 / Display P3 / Native, plus per-preset variations in primaries and TRC) | ICC profile association via `AssociateColorProfileWithDevice` + `WcsSetDefaultColorProfile`. Applied by color-aware code (DWM, Edge, etc.) via the profile's `rXYZ`/`gXYZ`/`bXYZ`/`rTRC`/`gTRC`/`bTRC` tags | **Works** — ICC color tags travel through a path that's independent of NVAPI |
| **B — preset's signature look** (Racing's bright greens, Cinema's film tint, Vivid's saturation boost, Eyecare's blue reduction) | `NvAPI_GPU_SetColorSpaceConversion` (CSC matrix layered on top of the ICC base) + `NvAPI_SetDVCLevelEx` (Digital Vibrance) | **Broken** — same -104 NVAPI_NOT_SUPPORTED on 591+ |

In dGPU mode you get **only Layer A**: the base gamut profile applies, producing a subtle color shift between presets (because their matrix/TRC tags differ slightly), but Layer B — the dramatic per-preset color personality — is missing.

**This cannot be fixed in g-helper** without NVIDIA restoring the underlying API. When the NVAPI regression is fixed, both flicker-free dimming and GameVisual color presets will work correctly on dGPU mode in a single change.

---

## For bug reports to ASUS / NVIDIA

**To ASUS:** the regression is at NVIDIA's API surface, not in `AsusSplendid.exe`. The single broken call is `NvAPI_GPU_SetColorSpaceConversion`. AsusSplendid catches the -104 error internally (its outer-wrapper log reports `CCTAPI_NV return: 0`) but the call has no effect. A possible Splendid-side workaround would be to fall through to `SetDeviceGammaRamp` (which Splendid already implements as the AMD-side fallback) with a 52..100 dim-level clamp — the same approach this fork takes.

**To NVIDIA:**

- **GPU:** RTX 4060 Laptop (this fork directly tested). Per user reports, also affects at least RTX 4070 and RTX 4090 (Strix SCAR, ProArt). Community reports extend the regression to RTX 30-series and RTX 50-series laptop dGPUs on the same NVIDIA 59x driver branch — the gate is not RTX 40-specific.
- **Driver:** 596.49 (`nvapi64.dll` FileVersion 32.0.15.9649); broken since 591.44 (December 2025)
- **Display:** internal eDP, displayId `0x80061080` on this machine
- **Function:** `NvAPI_GPU_SetColorSpaceConversion` (hash ID `0xFCABD23A`)
- **Symptom:** returns `-104 NVAPI_NOT_SUPPORTED` for any `Set`, even `Set(data_just_returned_by_Get)`. `Get` succeeds. Other display-side APIs (`GPU_GetDitherControl`, `GPU_GetColorSpaceConversion`) succeed for the same displayId.
- **Last known good driver:** 581.94 — same function, same hardware, works correctly
- **Cross-confirmed regression scope:** every RTX 40-series ROG laptop tested by users in the issue threads above, with community reports of the same symptom extending to RTX 30 and RTX 50-series laptops on the 59x driver branch, plus the unrelated `novideo_srgb` tool ([issue #129](https://github.com/ledoge/novideo_srgb/issues/129)) — the gate is not single-vendor or single-generation.
