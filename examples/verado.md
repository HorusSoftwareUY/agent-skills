> **Calibration run, regenerated 16-oct-2026** against the current `SKILL.md`. Fictional project.
> This is the case the skill exists for: **nothing blocks, and the payoff is partial.** A report that
> only ever said "migrate" would be worthless here.

```
┌──────────────────────────────────────────────────────────────────────┐
│  CoreCLR fit · Verado                                                │
│                                                                      │
│  Readiness   ███████████████░░░░░  73/100  upgrade on your schedule  │
│  Upside      █████████░░░░░░░░░░░  MEDIUM  up to 44% less time on    │
│                                            catalog lists             │
│                                                                      │
│  → Worth doing. Don't sell it internally as "the app gets            │
│    faster" — three screens get faster.                               │
│                                                                      │
│  Build & TFM            ███████░░░  36/50   net9.0 · 32-bit RIDs     │
│  XAML toolchain         ████████░░  17/20   6 of 38 bare             │
│  Dependencies           ███████░░░  20/30   1 Xamarin-era plugin     │
│                                                                      │
│  ✖ Breaks the build: none          After the fix list: 73 → 100      │
│  ── 6 of 34 candidates verified · SDK 9.0.304 · net9.0-android35.0   │
│     44 xaml (38 bind) · 171 cs · coreclr-fit v1                      │
│     skills.horus.com.uy/coreclr-fit                                  │
└──────────────────────────────────────────────────────────────────────┘
```

**Worth doing, on your own schedule.** Nothing in Verado stops a `net11` build. But the gain is
concentrated in three catalog screens, and the rest of the app is layout and native handlers, where the
runtime measured ≈ 1×. The honest internal pitch is *"three screens get faster"*, not *"the app gets
faster"* — and the difference matters, because the second one gets disproved the first time someone
opens the settings screen with a stopwatch.

`✖` breaks the build · `◐` blocks the gain · `○` runs as-is

## 1 · What was analyzed

| | |
|---|---|
| App project | `Verado.App/Verado.App.csproj` |
| TFMs scored | `net9.0-android35.0` · `net9.0-ios18.0` |
| SDK | 9.0.304. Major matches the TFM |
| Values | **Evaluated** per platform TFM, Release and Debug |
| Counted | 44 `.xaml` (38 bind) · 171 `.cs` · 34 candidates, 6 opened |

**One assumption taken.** Nobody answered the 32-bit question, so it is assumed they do not need
armeabi-v7a. If that is wrong this becomes a blocked upgrade, not a 73. Worth one re-run with someone
available to answer.

## 2 · Readiness by dimension

**Build & TFM — 36 / 50**

| Finding | `file:line` | Pts |
|---|---|---|
| TFM is `net9` | `Verado.App.csproj:9` | −7 |
| `RuntimeIdentifiers` still lists `android-arm` and `android-x86` | `Verado.App.csproj:17` | −7 |

What passed: `SupportedOSPlatformVersion` is already 24.0, `global.json` pins the SDK, and
`RunAOTCompilation` sits with `AndroidLinkMode=SdkOnly`.

**XAML toolchain — 17 / 20** · `x:DataType` on 32 of 38 binding files, −3. No build breaks.

**Dependencies — 20 / 30**

| Finding | `file:line` | Pts |
|---|---|---|
| `Xam.Plugin.Media` 5.0.1 | `Directory.Packages.props:19` | −6 |
| AndroidX pinned to explicit versions | `Directory.Packages.props:24-27` | −2 |
| MAUI pinned per csproj | `Verado.App.csproj:26`, `Verado.Core.csproj:8` | −2 |

## 3 · Upside by signal — 46 / 100 · MEDIUM

| Signal | Earned | The count behind it |
|---|---|---|
| CPU-bound work on the critical path | **20 / 45** | 24 LINQ call sites, **11 on paths the user waits for** and all eleven inside the three catalog view models. The other 13 are startup configuration and tests, and do not count |
| Serialization that can use STJ source-gen | **12 / 25** | STJ without a context. 9 DTOs, none with custom converters. Half points: mechanical to add |
| Session length | **8 / 18** | Retail browsing, seconds at a time, then the OS kills it |
| Android reach | **6 / 12** | Roughly half the installs, answered at step 3 |

**This is why the band is MEDIUM and the report says so plainly.** The work is real but narrow, and the
session pattern means most users never reach the steady-state number at all.

**Absent axes:** the Android reach row is Android-only by construction, so the iOS half of this product
has a structurally lower ceiling than the band suggests.

## 4 · Reported, never scored

| | Finding | `file:line` | Scope |
|---|---|---|---|
| ◐ | `JsonSerializer.Deserialize` without a source-generated context | `Services/CatalogClient.cs:44` | 9 DTOs on the catalog path |
| ○ | Analytics SDK initialised synchronously before the first frame | `MauiProgram.cs:38` | 1 call · `50–200 ms` measured |

> **The `Full` trimming note.** Under `SdkOnly` the reflection-serialized DTOs run as they are. Under
> `Full` the trimmer walks the app assembly too, and a property only ever touched by the serializer
> looks unreachable. It fails in the ugliest way: the DTO you display survives, the one you only
> serialize comes back with nulls, and it passes QA because QA walks the screens. **We did not measure
> this**; it is the mechanism, not a result. Verado is on `SdkOnly` and has never built `Full`, so this
> is what changes if they ever switch, not what is happening now.

## 5 · Fix list

**Tier 1 — 5 items · 6 files · takes Readiness from 73 to 94.**

**1 · ◐ TFM to `net11` (+7)**
```xml
<TargetFrameworks>net11.0-android35.0;net11.0-ios18.0</TargetFrameworks>
```
*Scope: 1 file · 2 TFMs, plus the workload bump.*

**2 · ✖ Drop the 32-bit RIDs (+7)** — CoreCLR on Android is 64-bit only; there is no configuration
around it.
```xml
<RuntimeIdentifiers>android-arm64;android-x64</RuntimeIdentifiers>
```
*Scope: 1 file · 1 property. Confirm the 32-bit question first.*

**3 · ◐ `x:DataType` on the 6 remaining views (+3)** · **4 · ○ AndroidX through MAUI's graph (+2)** ·
**5 · ○ One `MauiVersion` (+2)**
*Scope: 6 xaml · 2 props · 2 csproj.*

**Tier 2 — 1 item · takes Readiness from 94 to 100.** Replace `Xam.Plugin.Media` with MAUI Essentials'
`MediaPicker` (+6).
```csharp
var photo = await MediaPicker.Default.CapturePhotoAsync();
```
*Scope: 1 package · 4 call sites in 2 files.*

**Nothing is deliberately off the list.** The ceiling is 100.

## 6 · Projected gains

Setup: .NET 11 preview 5, Release arm64, physical Xiaomi Mi 9T Pro and moto g54 5G.

> Measured on .NET 11 preview 5 with both runtimes on the same code. Your baseline is **`net9.0-android35.0`**
> with Mono, which we did not measure — we assume the Mono side behaves comparably for this kind of
> work. Treat these as approximations until you measure your own.

Verado is on .NET 9, two versions further back than the measured baseline. Read every row as a shape
rather than a figure.

### What the runtime gives you — nothing to do

| | Measured | Why it applies to Verado |
|---|---|---|
| A full data load, first time in the process | 363 → 272 ms · **1,33×** · −25% | **The row that matters most here.** Retail sessions are short, so most users only ever see the first load |
| The same load a few more times (avg of 5) | 252 → 166 ms · **1,52×** · −34% | Someone browsing several categories in one session |
| Warm steady state | 238 → 90 ms · **2,6×** · −62% | **Most Verado users never get here.** Tier-1 state dies with the process, and the OS kills this app often |
| by phase · **LINQ** | 38 → 21 ms · **1,8×** · −44% | The 11 call sites in the catalog view models |
| by phase · **JSON** | 176 → 60 ms · **2,9×** · −66% | **Not available today** — needs a source-generated context on the 9 DTOs first |

### Where it does not win, or loses

| | Measured | |
|---|---|---|
| First load, small phases | dates 25 → **61 ms** · bind 8 → **14 ms** | **Most of Verado is this.** The product, cart and account screens do little managed work and open once. Expect nothing there, and a little worse |
| UI-thread work | 2 → 1 ms · **≈ 1×** | Scrolling the catalog will not feel different |
| Raw cold start | **−4%** | At the noise floor |
| APK size | ~29 → ~30 MB | Slight regression |

**Read this block before the one above it when you present this internally.** It is the part that
decides whether the three faster screens are worth a sprint.

### What build config gives you — Android only, an afternoon

Custom startup MIBC profile: **−21% cold start**, worth more here than the runtime change, because
short sessions mean startup is most of what users experience. End to end: **−25%**.

### What their own code gives you — no upgrade required

| | Measured | |
|---|---|---|
| Lazy first page, Debug-only logging, lazy flyout | **−52%** on Mono · **−53%** on CoreCLR | The analytics init at `MauiProgram.cs:38` is the one finding here, worth `50–200 ms` |
| And the runtime still pays on top of it | 1.850 → 1.380 ms · **−25%** | After that work, on both runtimes, CoreCLR is still a quarter faster |

### Null results

SkSL shaders: **no difference**. Scroll FPS and navigation timing: **not measured**.

These are projections from a reference benchmark, not measurements of your app — the upgrade steps
include the side-by-side TFM setup so you can measure before committing.

## 7 · What happens if you do nothing

A legitimate answer for Verado, and the report should say it out loud. **.NET 9 is out of support, but
.NET 10 is LTS until 14 November 2028.** Moving to 10 instead of 11 gets them back into support, keeps
Mono, and costs only item 1 of the fix list.

What they would give up: the three catalog screens stay as they are, and the MIBC profile — the biggest
single lever in this report — does not exist on Mono.

What they would keep: everything in § 6's last block. The analytics init is worth fixing either way.

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

**Measure the catalog screens specifically**, not the app as a whole. The aggregate will show almost
nothing, because most of the app is the block that does not win.

## 9 · Preview notes

Measured on a preview SDK. Re-check at GA: the `SdkOnly`-on-Mono regression (`dotnet/maui#33032`),
whether `x:Code` still fails to compile, and the MIBC tooling.

These figures come from the `coreclr-fit v1` evidence set, measured on .NET 11 preview 5. They are
re-measured when the runtime changes; the current set is always at `skills.horus.com.uy/coreclr-fit`.

## 10 · Badge

```markdown
![CoreCLR readiness](https://img.shields.io/badge/CoreCLR_readiness-73%25-green)
```

---

```
   ▄██▀▀██▄
 ▄█▀   ██▀  ▄     h o r u s   s t u d i o .
██  ▄▄▄   ▄███
██ ██▀██▄██ ██    CoreCLR fit · built from a real Mono → CoreCLR
███▀   ▀▀▀  ██    upgrade on .NET 11, measured on device
 ▀  ▄██   ▄█▀     Need a hand with this? horus.com.uy/contact
   ▀██▄▄██▀       skills.horus.com.uy/coreclr-fit
```
