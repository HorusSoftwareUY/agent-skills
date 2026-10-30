> **Calibration run, regenerated 29-oct-2026** against the current `SKILL.md`. Fictional project.
> This is the case the skill exists for: **nothing blocks, and the payoff is partial.** A report that
> only ever said "migrate" would be worthless here.

# CoreCLR fit: Verado (net9.0-android35.0 → .NET 11)

## Summary

You can move Verado to .NET 11 whenever you like. The one thing you have to change first is a build
setting, not code.

The speed gain is narrow. Three catalog screens get noticeably faster. The rest of your app stays
roughly as it is today.

The work is six items across six files, and most of it is configuration rather than code. Three of them
pay off whether or not you upgrade.

You can stay on .NET 10 until support ends on 14 November 2028. So this is a planning decision, not an
urgent one.

```
┌──────────────────────────────────────────────────────────────────────┐
│  CoreCLR fit · Verado                                                │
│                                                                      │
│  Readiness   ███████████████░░░░░  73/100  upgrade on your schedule  │
│  Upside      ███████░░░░░░░░░░░░░  LOW     up to 44% less time on    │
│                                            catalog lists             │
│                                                                      │
│  → You're ready, and it won't pay for itself. Three catalog          │
│    screens get faster; the rest of your app measured ≈ 1×.           │
│                                                                      │
│  Build & TFM            ███████░░░  36/50   net9.0 · 32-bit RIDs     │
│  XAML toolchain         ████████░░  17/20   6 of 38 bare             │
│  Dependencies           ███████░░░  20/30   1 package behind · pins  │
│                                                                      │
│  ✖ Breaks the build: none          After the fix list: 73 → 100      │
│  ── 6 of 34 candidates verified · SDK 9.0.304 · net9.0-android35.0   │
│     44 xaml (38 bind) · 171 cs · coreclr-fit v1                      │
│     skills.horus.com.uy/coreclr-fit                                  │
└──────────────────────────────────────────────────────────────────────┘
```

**The gain is concentrated in three catalog screens.** The rest of the app is layout and native
handlers, where the runtime measured ≈ 1×. When you pitch this internally, say *"three screens get
faster"*, not *"the app gets faster"*. The second one gets disproved the first time someone opens the
settings screen with a stopwatch.

`✖` breaks the build · `◐` blocks the gain · `○` runs as-is

## 1 · What was analyzed

| | |
|---|---|
| App project | `Verado.App/Verado.App.csproj` |
| TFMs scored | `net9.0-android35.0` · `net9.0-ios18.0` |
| SDK | 9.0.304. The major version matches your TFM |
| Values | **Evaluated** per platform TFM, in Release and Debug |
| Counted | 44 `.xaml` (38 bind) · 171 `.cs` · 34 candidates, 6 opened |

**We assumed one thing.** Nobody answered the 32-bit question, so we assumed you don't need
armeabi-v7a. If that's wrong, this is a blocked upgrade rather than a 73. It's worth one re-run with
someone who can answer.

## 2 · Readiness by dimension

**Build & TFM — 36 / 50**

| Finding | `file:line` | Pts |
|---|---|---|
| Your TFM is `net9` | `Verado.App.csproj:9` | −7 |
| `RuntimeIdentifiers` still lists `android-arm` and `android-x86` | `Verado.App.csproj:17` | −7 |

What already passes: `SupportedOSPlatformVersion` is 24.0, `global.json` pins your SDK, and
`RunAOTCompilation` sits with `AndroidLinkMode=SdkOnly`.

**XAML toolchain — 17 / 20** · You have `x:DataType` on 32 of 38 binding files, so −3. Nothing breaks
the build.

**Dependencies — 20 / 30**

| Finding | `file:line` | Pts |
|---|---|---|
| `Xam.Plugin.Media` 5.0.1 ships `monoandroid81` and `xamarin.ios10` assets only, with no .NET TFM at all | `Directory.Packages.props:19` | −6 |
| AndroidX is pinned to explicit versions | `Directory.Packages.props:24-27` | −2 |
| MAUI is pinned per csproj | `Verado.App.csproj:26`, `Verado.Core.csproj:8` | −2 |

## 3 · Upside by signal — 34 / 100 · LOW

| Signal | Earned | The count behind it |
|---|---|---|
| CPU-bound work on the critical path | **12 / 45** | You have 24 LINQ call sites. **Eleven sit on paths your users wait for**, and all eleven live in the three catalog view models. The other 13 are startup configuration and tests, so they don't count. Eleven verified hits lands in the `6 to 25` band. There's no date parsing or string building anywhere, and no step down, because 12 is the floor |
| Serialization that can use STJ source-gen | **12 / 25** | You're on STJ without a context. 9 DTOs, none with custom converters. Half points, because adding it is mechanical |
| Session length | **4 / 18** | Retail browsing. Seconds at a time, then the OS kills the app. Short |
| Android reach | **6 / 12** | Roughly half your installs, from your answer at step 3 |

**Here's why the band is LOW, and why we'd rather say it plainly.** The work is real, but it's narrow:
eleven call sites in three view models, over catalog pages of tens of items, in sessions that end before
tier-1 ever warms up. You can move to .NET 11 whenever you like. Just don't expect the move to pay for
itself. *Ready, and it won't pay you* is the sentence nobody else will write for you, and here it's
worth more than a number that flatters the upgrade.

**One axis doesn't apply to all of your product.** Android reach is Android-only by construction, so the
iOS half has a lower ceiling than the band suggests.

## 4 · Reported, never scored

| | Finding | `file:line` | Scope |
|---|---|---|---|
| ◐ | `JsonSerializer.Deserialize` runs without a source-generated context | `Services/CatalogClient.cs:44` | 9 DTOs on the catalog path |
| ○ | Your analytics SDK initializes synchronously before the first frame | `MauiProgram.cs:38` | 1 call · `50–200 ms` measured |

> **About `Full` trimming.** Under `SdkOnly`, your reflection-serialized DTOs run as they are. Under
> `Full`, the trimmer walks your app assembly too, and a property that only the serializer ever touches
> looks unreachable. It fails in the ugliest way: the DTO you display survives, the one you only
> serialize comes back with nulls, and it passes QA, because QA walks the screens. **We didn't measure
> this.** It's the mechanism, not a result. You're on `SdkOnly` and have never built `Full`, so this is
> what changes if you switch, not what's happening now.

## 5 · Fix list

**Tier 1 — 5 items · 6 files · takes Readiness from 73 to 94.**

**1 · ◐ Move your TFM to `net11` (+7)**
```xml
<TargetFrameworks>net11.0-android35.0;net11.0-ios18.0</TargetFrameworks>
```
*Scope: 1 file · 2 TFMs, plus the workload bump.*

**2 · ◐ Drop the 32-bit RIDs (+7)** — CoreCLR on Android is 64-bit only, and there's no configuration
around it. We expect this to stop your build, but we haven't confirmed it against a compiler, so it
doesn't carry the build-break mark.
```xml
<RuntimeIdentifiers>android-arm64;android-x64</RuntimeIdentifiers>
```
*Scope: 1 file · 1 property. Confirm the 32-bit question first.*

**3 · ◐ Add `x:DataType` to the 6 remaining views (+3)** · **4 · ○ Let AndroidX come through MAUI's
graph (+2)** · **5 · ○ Use a single `MauiVersion` (+2)**
*Scope: 6 xaml · 2 props · 2 csproj.*

**Tier 2 — 1 item · takes Readiness from 94 to 100.** Swap `Xam.Plugin.Media` for MAUI Essentials'
`MediaPicker` (+6).
```csharp
var photo = await MediaPicker.Default.CapturePhotoAsync();
```
*Scope: 1 package · 4 call sites in 2 files.*

**Nothing is deliberately off this list.** The ceiling is 100.

## 6 · Projected gains

Setup: .NET 11 preview 5, Release arm64, on a physical Xiaomi Mi 9T Pro and a moto g54 5G.

> Measured on .NET 11 preview 5 with both runtimes on the same code. Your baseline is **`net9.0-android35.0`**
> with Mono, which we did not measure — we assume the Mono side behaves comparably for this kind of
> work. Treat these as approximations until you measure your own.

You're on .NET 9, two versions further back than the baseline we measured. Read every row as a shape
rather than a figure.

### What the runtime gives you, with no work

| | Measured | Why it applies to you |
|---|---|---|
| A full data load, first time in the process | 363 → 272 ms · **1,33×** · −25% | **The row that matters most to you.** Retail sessions are short, so most of your users only ever see the first load |
| The same load a few more times (avg of 5) | 252 → 166 ms · **1,52×** · −34% | Someone browsing several categories in one session |
| Warm steady state | 238 → 90 ms · **2,6×** · −62% | **Most of your users never get here.** Tier-1 state dies with the process, and the OS kills this app often |
| by phase · **LINQ** | 38 → 21 ms · **1,8×** · −44% | Your 11 call sites in the catalog view models |
| by phase · **JSON** | 176 → 60 ms · **2,9×** · −66% | **Not available to you today.** You need a source-generated context on the 9 DTOs first |

### Where it doesn't win, or loses

| | Measured | |
|---|---|---|
| First load, small phases | dates 25 → **61 ms** · bind 8 → **14 ms** | **Most of your app is this.** Your product, cart and account screens do little managed work and open once. Expect nothing there, and a little worse |
| UI-thread work | 2 → 1 ms · **≈ 1×** | Scrolling the catalog won't feel different |
| Raw cold start | **−4%** | At the noise floor |
| APK size | ~29 → ~30 MB | A slight regression |

**Read this block before the one above it when you present this internally.** It's the part that decides
whether three faster screens are worth a sprint.

### What build config gives you, on Android, in an afternoon

A custom startup MIBC profile gets you **−21% cold start**. That's worth more to you than the runtime
change, because short sessions mean startup is most of what your users experience. End to end: **−25%**.

### What your own code gives you, with no upgrade

| | Measured | |
|---|---|---|
| Lazy first page, Debug-only logging, lazy flyout | **−52%** on Mono · **−53%** on CoreCLR | Your analytics init at `MauiProgram.cs:38` is the one finding here, worth `50–200 ms` |
| And the runtime still pays on top of it | 1.850 → 1.380 ms · **−25%** | After that work, on both runtimes, CoreCLR is still a quarter faster |

### Null results

SkSL shaders: **no difference**. Scroll FPS and navigation timing: **not measured**.

These are projections from a reference benchmark, not measurements of your app. The upgrade steps
include the side-by-side TFM setup, so you can measure before you commit.

## 7 · What happens if you do nothing

This is a legitimate answer for you, and we'd rather say it out loud. **.NET 9 is out of support, but
.NET 10 is LTS until 14 November 2028.** Moving to 10 instead of 11 gets you back into support, keeps
Mono, and costs you only item 1 of the fix list.

What you'd give up: the three catalog screens stay as they are, and the MIBC profile, the biggest single
lever in this report, doesn't exist on Mono.

What you'd keep: everything in the last block of section 6. The analytics init is worth fixing either
way.

## 8 · Upgrade steps

```xml
<PropertyGroup Condition="'$(Configuration)' == 'Release' and $(TargetFramework.StartsWith('net11.0-android'))">
  <AndroidLinkMode>SdkOnly</AndroidLinkMode>
  <PublishReadyToRun>true</PublishReadyToRun>
</PropertyGroup>
```

```xml
<TargetFrameworks>net9.0-android35.0;net11.0-android35.0</TargetFrameworks>

<PropertyGroup Condition="$(TargetFramework.Contains('-android'))">
  <RuntimeIdentifiers>android-arm64;android-x64</RuntimeIdentifiers>
  <ApplicationId Condition="$(TargetFramework.StartsWith('net9.'))">com.verado.app.net9</ApplicationId>
  <ApplicationId Condition="$(TargetFramework.StartsWith('net11.'))">com.verado.app.net11</ApplicationId>
</PropertyGroup>
```

**Measure the catalog screens specifically**, not the app as a whole. The aggregate will show you almost
nothing, because most of your app is the block that doesn't win.

## 9 · Preview notes

We measured on a preview SDK. Re-check at GA: the `SdkOnly`-on-Mono regression (`dotnet/maui#33032`),
whether `x:Code` still fails to compile, and the MIBC tooling.

These figures come from the `coreclr-fit v1` evidence set, measured on .NET 11 preview 5. They are
re-measured when the runtime changes; the current set is always at `skills.horus.com.uy/coreclr-fit`.

*Produced by Claude Code running Claude Opus, self-reported.*

---

```
   ▄██▀▀██▄
 ▄█▀   ██▀  ▄     h o r u s   s t u d i o .
██  ▄▄▄   ▄███
██ ██▀██▄██ ██    CoreCLR fit · numbers from an upgrade we ran
███▀   ▀▀▀  ██    ourselves on .NET 11, measured on two phones
 ▀  ▄██   ▄█▀     Need a hand with this? horus.com.uy/contact
   ▀██▄▄██▀       skills.horus.com.uy/coreclr-fit
```
