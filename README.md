# g-helper (OLED DC Dimming Fix for NVIDIA 591+)

A fork of [seerge/g-helper](https://github.com/seerge/g-helper) that restores OLED flicker-free dimming on ROG laptops when the dGPU drives the internal panel — broken since NVIDIA driver 591.44.

## TL;DR

| | Behaviour |
|---|---|
| **Setup** | ROG / ProArt laptop with an OLED panel and an NVIDIA dGPU. Directly tested on RTX 40-series; community reports indicate the same NVIDIA 59x regression also affects RTX 30 and RTX 50-series, so this fork is likely useful for those too. |
| **Bug** | On NVIDIA driver 591.44+ in **dGPU / Ultimate mode**, the OLED brightness slider in upstream g-helper (and Armoury Crate, and ASCI itself) silently does nothing. The panel stays at full brightness. |
| **Why** | `AsusSplendid.exe` uses the private NVAPI call `NvAPI_GPU_SetColorSpaceConversion` to apply a 3×4 RGB scale matrix at the GPU display output stage. NVIDIA's 59x driver series gates this call for the internal eDP panel in dGPU passthrough mode (returns `NVAPI_NOT_SUPPORTED` / -104). |
| **This fork** | Adds a small `GdiDimmer` that writes a linear gamma ramp via Win32 `SetDeviceGammaRamp` on the laptop panel only, when the system is in Ultimate mode and the panel is OLED. Visible dim returns. |
| **Limitation** | The bottom ~20% of Splendid's brightness range can't be reached: Splendid's slider-0 reaches linear gain 0.40, ours reaches 0.52, because the Windows ICM clamp rejects gamma ramps with gain < ~0.50. The slider 0..100 is remapped to dim level 52..100 so every position is still responsive. |
| **iGPU mode** | Completely untouched — `AsusSplendid`'s AMD ADL/ADLX path still handles dimming natively. |

## Install

1. Download / build `app/bin/x64/Release/net8.0-windows/win-x64/publish/GHelper.exe` (~5.8 MB, framework-dependent).
2. Replace your existing `GHelper.exe` if you installed upstream's build.
3. Requires the **.NET 8 Desktop Runtime** to be installed (same as upstream).

If you ran a dev build during testing and the `GHelper` / `GHelperCharge` scheduled tasks now point at the dev path, open g-helper Settings and toggle the **Run on Startup** checkbox off, then back on. That re-registers both tasks against the new exe path. (Simply re-launching the installed copy is not enough — g-helper's on-launch check only re-schedules when the old path is missing or the running version is *newer* than the scheduled one, so a same-version reinstall at a different path is ignored.)

## Enabling the workaround (opt-in)

The dGPU dimming workaround is **off by default** so users on NVIDIA drivers older than 591.44 — where AsusSplendid's CSC still works correctly — see no behavioural change. Enabling the workaround on a working-driver setup would over-dim the panel (Splendid's scale and this fork's GDI ramp would compound multiplicatively).

If you're on the affected driver branch (591.44+), enable the workaround like this:

1. Close GHelper (right-click the tray icon → Quit).
2. Open `%APPDATA%\GHelper\config.json` in a text editor. Create the file if it doesn't exist (it will after GHelper has run at least once).
3. Add the key `"oled_dgpu_dimmer_workaround": 1` to the JSON object. For example:
   ```json
   {
     "model": "...",
     "oled_dgpu_dimmer_workaround": 1,
     ...
   }
   ```
4. Save and restart GHelper.

To disable it again, change the value to `0` (or remove the key) and restart.

A one-liner that does it for you (PowerShell, run with GHelper closed):

```powershell
$cfg = "$env:APPDATA\GHelper\config.json"
$j = if (Test-Path $cfg) { Get-Content $cfg -Raw | ConvertFrom-Json } else { @{} }
$j | Add-Member -NotePropertyName 'oled_dgpu_dimmer_workaround' -NotePropertyValue 1 -Force
$j | ConvertTo-Json -Depth 10 | Set-Content $cfg -Encoding utf8
```

## What works and what doesn't

| Scenario | Pre-591 (Splendid CSC) | This fix |
|---|---|---|
| Whites dim alongside everything | ✓ | ✓ |
| Cursor dims | ✓ | ✓ (LUT stage is post-cursor) |
| Per-monitor (laptop only) | ✓ | ✓ (per-HDC ramp) |
| FSO / borderless games (Win11 default) | ✓ | ✓ |
| True fullscreen-exclusive D3D games (rare on Win11) | ✓ | partial — ramp gets wiped on swapchain init; see caveat |
| HDR mode active | ✗ (already disabled in Splendid) | ✗ (Windows ignores `SetDeviceGammaRamp` in HDR) |
| Slider range | 0..100 → scale **0.40**..1.00 | 0..100 → scale **0.52**..1.00 |
| Visible dim feel | Linear RGB scale | **Identical** within the 0.52..1.00 range |

**Fullscreen-exclusive caveat:** modern Win11 games use Fullscreen Optimizations (DWM-composited borderless) and never reset the gamma ramp themselves. Only the rare true-FSE D3D game calls `IDXGIOutput::SetGammaControl(identity)` on swapchain creation, which wipes our ramp. If that happens, nudge the brightness slider once after the game has started — that re-applies the ramp.

**GameVisual color presets are *also* half-broken in dGPU mode** (subtle color shifts where iGPU shows dramatic ones). Same NVAPI CSC regression — see [`INVESTIGATION.md`](INVESTIGATION.md) for the analysis. Not fixable from g-helper; needs NVIDIA to restore the API.

## Build from source

Same command upstream uses ([`.github/workflows/release.yml`](.github/workflows/release.yml)):

```powershell
dotnet publish app/GHelper.sln `
    --configuration Release --runtime win-x64 `
    -p:PublishSingleFile=true --no-self-contained
```

Output: `app/bin/x64/Release/net8.0-windows/win-x64/publish/GHelper.exe`.

Requires .NET 8 SDK to build.

## Documents

- [`CHANGES.md`](CHANGES.md) — exact line-level patch manifest of this fork vs upstream.
- [`INVESTIGATION.md`](INVESTIGATION.md) — technical summary: the bug, the user-mode paths that were measured and ruled out, why we landed on `SetDeviceGammaRamp` constrained to gain ≥ 0.52, and the GameVisual side-effect of the same NVAPI regression.
- [`docs/README.md`](docs/README.md) — upstream g-helper's original README (unchanged).

## Upstream

This fork tracks [seerge/g-helper](https://github.com/seerge/g-helper). The OLED dimming patch is the only difference — see [`CHANGES.md`](CHANGES.md) for the file-level diff.

## Acknowledgements

Developed in pair-programming sessions with **Anthropic's Claude** (Sonnet and Opus). Claude drafted the `GdiDimmer` implementation, the integration into `VisualControl`, and the documentation; the hardware testing and design calls (slider remap to 52..100, rejecting the ICM registry tweak after seeing it ineffective on Win11 25H2) were driven by the user on a ROG Zephyrus G16 GA605WV with RTX 4060 and driver 596.49.

The independent project [ledoge/novideo_srgb](https://github.com/ledoge/novideo_srgb) was load-bearing for the investigation: its [issue #129](https://github.com/ledoge/novideo_srgb/issues/129) cross-confirmed the same NVAPI regression on driver 595.71+, and its source code provided the verified function-ID `0xFCABD23A` and `Csc` struct layout used to characterise the broken NVAPI call.
