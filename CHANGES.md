# Patch manifest — this fork vs upstream `seerge/g-helper`

All changes are additive. **Original g-helper code logic is not modified.**

## New files

| Path | Purpose |
|---|---|
| `app/Display/GdiDimmer.cs` | Dimmer implementation: builds a per-channel linear gamma ramp and writes it to the laptop panel via Win32 `SetDeviceGammaRamp`. ~145 lines. |
| `README.md` | Fork entry point — TL;DR, install, build, acknowledgements. |
| `CHANGES.md` (this file) | The patch manifest. |
| `INVESTIGATION.md` | Technical summary: the bug, the user-mode paths that were measured and ruled out, why the fix lives at gain ≥ 0.52, and the GameVisual side-effect of the same NVAPI regression. |

## Modified files (additive only)

### `app/Display/DisplayNative.cs`

Three additions between two existing P/Invoke declarations (around line 472):

```csharp
[DllImport("gdi32")]
public static extern bool DeleteDC(IntPtr hdc);

[DllImport("gdi32")]
public static extern bool SetDeviceGammaRamp(IntPtr hdc, ref GAMMA_RAMP ramp);

/// <summary>
/// 256 entries per channel x 3 channels (R,G,B), 16-bit each.
/// Layout matches what Win32 expects for SetDeviceGammaRamp.
/// </summary>
[StructLayout(LayoutKind.Sequential)]
public unsafe struct GAMMA_RAMP
{
    public fixed ushort table[3 * 256];
}
```

Total: ~15 lines inserted. No surrounding pre-existing code touched.

### `app/Display/VisualControl.cs`

Three small edits, all in/adjacent to `BrightnessTimerTimer_Elapsed`:

1. **Top of file (line 1)** — added `using GHelper.Gpu;` so the new method can reference `GPUModeControl.gpuMode`.

2. **Inside `BrightnessTimerTimer_Elapsed`** — three identical, minimal extensions of existing exit points. Each existing `return;` line becomes `{ ApplyDimmerForDGpu(); return; }`, and one trailing `ApplyDimmerForDGpu();` call is added at the natural fall-through. Net effect: the new helper runs at every exit, with **no change to the original control flow or any existing logic**:

   ```csharp
   if (RunSplendid(dimmingCommand, 0, dimmingLevel) == 0) { ApplyDimmerForDGpu(); return; }   // was: return;
   ...
       if (RunSplendid(dimmingCommand, 0, dimmingLevel) == 0) { ApplyDimmerForDGpu(); return; }   // was: return;
   ...
   ApplyDimmerForDGpu();   // new trailing line at end of method
   ```

3. **After `BrightnessTimerTimer_Elapsed`** — new private method `ApplyDimmerForDGpu()` appended (~20 lines). Pure addition.

The `ApplyDimmerForDGpu` body:

```csharp
private static void ApplyDimmerForDGpu()
{
    try
    {
        // Opt-in: AppConfig key "oled_dgpu_dimmer_workaround" (default 0).
        // Off by default so upstream behaviour is unchanged on pre-591
        // drivers (where Splendid's CSC works and stacking with our GDI
        // ramp would over-dim).
        if (!GdiDimmer.Enabled)
        {
            if (GdiDimmer.IsActive) GdiDimmer.Reset();
            return;
        }
        if (GPUModeControl.gpuMode != AsusACPI.GPUModeUltimate ||
            !AppConfig.IsOLED() || _brightness >= 100)
        {
            if (GdiDimmer.IsActive) GdiDimmer.Reset();
            return;
        }
        // slider 0..100 -> dimLevel 52..100 (linear, ICM-clamp-safe)
        GdiDimmer.Apply((int)Math.Round(52 + _brightness * 0.48));
    }
    catch (Exception ex)
    {
        Logger.WriteLine("Dimmer error: " + ex.Message);
    }
}
```

### `app/Program.cs`

One line added inside the existing `OnExit` method (around line 464):

```csharp
GdiDimmer.Shutdown();
```

Restores the laptop panel's gamma ramp to identity on application exit. No other change to `OnExit` or anywhere else in `Program.cs`.

## Summary of touch on original g-helper code

| Location | Edit |
|---|---|
| `app/Display/VisualControl.cs:1` | 1 line — added `using GHelper.Gpu;` |
| `app/Display/VisualControl.cs` inside `BrightnessTimerTimer_Elapsed` | 2 existing `return;` lines wrapped: `return;` → `{ ApplyDimmerForDGpu(); return; }` |
| `app/Display/VisualControl.cs` end of `BrightnessTimerTimer_Elapsed` | 1 line added — `ApplyDimmerForDGpu();` |
| `app/Program.cs` inside `OnExit` | 1 line added — `GdiDimmer.Shutdown();` |
| `app/Display/DisplayNative.cs` | 15 lines inserted between two existing P/Invoke blocks |

Net pre-existing-code modifications: **5 line-touches across 2 files** — 1 `using` directive, 2 existing `return;` statements wrapped, 1 trailing helper call appended, and 1 `GdiDimmer.Shutdown();` line in `OnExit`. Everything else is in new files or appended methods.

## New configuration key

| AppConfig key | Type | Default | Effect |
|---|---|---|---|
| `oled_dgpu_dimmer_workaround` | int (0 or 1) | `0` (off) | When `1` and the system is in Ultimate/dGPU mode with an OLED panel, applies a per-display GDI gamma ramp on the laptop panel via `SetDeviceGammaRamp` after each slider movement, restoring functional flicker-free dim under NVIDIA driver 591.44+. When `0` or in any non-Ultimate / non-OLED scenario, the new code is a no-op and behaviour is byte-identical with upstream. |

Lives in `%APPDATA%\GHelper\config.json`. Off-by-default protects pre-591 driver users: if both Splendid's CSC and our GDI ramp fired together, the effective gain would be g² (multiplicative) and the panel would over-dim. The opt-in is the user's confirmation that "Splendid's CSC isn't working for me, so applying the GDI ramp on top isn't compounding anything."

See `README.md` "Enabling the workaround" for the user-facing one-liner.

## Why these specific touches

- The new `ApplyDimmerForDGpu()` call needs to fire after **every** exit of `BrightnessTimerTimer_Elapsed`, because upstream's three exit paths (success, success after re-init, fall-through after re-init) all need the new dim applied. Wrapping the two `return;` statements and adding a trailing call is the minimum-touch way to achieve "always run after the existing logic completes". A `try/finally` wrapper would have required indenting the entire body of the original method, which is a bigger structural touch.
- `GdiDimmer.Shutdown()` in `OnExit` ensures the panel returns to identity gamma when g-helper closes (rather than leaving the dim baked in after exit).
- `using GHelper.Gpu;` is the unavoidable cost of `ApplyDimmerForDGpu()` referencing `GPUModeControl.gpuMode`.

## Build

Same command upstream uses ([release.yml](.github/workflows/release.yml#L25)):

```powershell
dotnet publish app/GHelper.sln --configuration Release --runtime win-x64 -p:PublishSingleFile=true --no-self-contained
```

Output: `app/bin/x64/Release/net8.0-windows/win-x64/publish/GHelper.exe` (~5.8 MB, framework-dependent on .NET 8 Desktop Runtime).
