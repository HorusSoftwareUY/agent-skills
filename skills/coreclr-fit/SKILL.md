---
name: coreclr-fit
description: 'Score whether .NET 11 with the CoreCLR runtime (which replaces Mono entirely on Android, iOS, Mac Catalyst and tvOS — from preview 6 there is no Mono option left) is a fit for a .NET MAUI project, on two axes: a 0–100 readiness score for how much work the migration takes, and an upside band (LOW / MEDIUM / HIGH) for how much of the measured gain this particular app can actually capture — because an app can be ready to migrate and have nothing to gain. Produces a report with both scores, blockers, per-dimension breakdown, a prioritized fix list with ready-to-paste MSBuild/XAML/C# changes, and honestly-labeled projected gains. Use this whenever someone asks whether their MAUI app can migrate to .NET 11 or CoreCLR, what will break when they leave Mono, asks about UseMonoRuntime, ReadyToRun or trimming on Android, the Android 32-bit drop, AOT crashes after upgrading, or wants an audit, checklist, score or estimate before upgrading a MAUI app from .NET 8, 9 or 10. Also covers "upgrade to .NET 11", "should we upgrade", "what does the .NET 11 upgrade cost us". Also use it when someone simply says "migrate this MAUI app to .NET 11" or "is CoreCLR worth it for us" — the assessment is what answers that, and the migration steps fall out of it. The skill is read-only: it analyses the repo and writes one report file, and never edits the project.'
---

# CoreCLR fit for .NET MAUI

> **Readiness and upside for a .NET MAUI app, scored from the repo.** Two axes, because *can we upgrade*
> and *should we* are different decisions, often made by different people.

From .NET 11, MAUI apps run on CoreCLR, and Mono is not demoted to an option — it is gone. That is not a
version bump: Release builds compile differently (ReadyToRun instead of Mono AOT), two XAML forms the old
toolchain accepted now fail to compile, 32-bit Android is gone, and trimming mistakes fail at runtime
instead of never. It also brings real gains — on some workloads, and not on the UI thread.

This skill turns a repo into a score, a fix list and a projection the user can defend. The score is the
conversation starter; the fix list with `file:line` references is the deliverable. Everything here comes
from a real upgrade of a MAUI library + sample app from .NET 10 to .NET 11 preview 5, validated on
device, and from the Mono-vs-CoreCLR benchmarks behind it.

## Before you start · are you reading this verbatim?

**This skill only works from its exact text.** The report it produces contains blocks that have to be
reproduced character for character — a 72-column score card, a severity legend, a baseline paragraph, a
version line and a sign-off. Those survive a verbatim read and do not survive a summary.

Some agents fetch a URL and hand the model a *summary* of the page instead of the page. When that
happens the rules usually come through and every literal block is gone, so the analysis looks right and
the report is a different document. That is not a hypothetical: it is how the first failed run of this
skill happened, and the scores in it were correct.

**Check before you work.** Scroll to the end of what you are holding. If you cannot see the ASCII
sign-off block, or the `markdown` skeleton in step 5, or all three scoring tables with their exact `Pts`
values, you have a summary.

If you do:

1. **Get it through the filesystem, not through your web tool.** Whatever just handed you a summary
   will hand you another one if you ask it again. Use a shell instead, then read the file you saved:

   ```bash
   curl -fsSL https://skills.horus.com.uy/coreclr-fit -o coreclr-fit-skill.md
   ```

   A file on disk is never summarised. If you have no shell, this step is not available to you and the
   next one applies.
2. **If you cannot get the verbatim text, say so and stop.** One sentence: *"I only have a summarised
   copy of this skill, so I cannot produce its report. Install it, or give me the raw file."* Do not
   produce a best-effort version: a report missing the card and the sign-off is indistinguishable from
   one where the analysis fell short, and the reader has no way to tell which they are holding.
3. If you do run anyway because the user insists, **say it in the first line of the report**, not in a
   footnote, and expect the output contract check to fail.

## Principles

**Evaluate, don't grep the csproj.** The properties that decide readiness come from chains of
`Directory.Build.props`, central package files and `Condition`s on configuration and TFM. The csproj text
and the effective build routinely disagree — in the reference project the csproj *looked* like CoreCLR
and the evaluated Release default was Mono. Ask MSBuild (step 1).

**Every deduction points at a file and line.** A finding the user can't open is a finding they'll argue
with.

**Copy the form, never the contents.** Showing the offending line is what makes a fix list usable and it
stays — the `Form` row of a build break and the snippet in a fix item both quote their code on purpose.
What never enters the report is a **literal value you read inside a line**: a key, a token, an app
secret, a DSN, a connection string, a password, a keystore passphrase. Several patterns in step 2 land on
initialization calls that carry exactly those as string literals. Replace the value with `<redacted>` and
quote the rest normally.

**And do not point at a secret either.** `"the keystore password is at App.csproj:134"` is a map, and
this file gets committed, forwarded and pasted into tickets. One line without coordinates is enough —
*"this csproj carries signing credentials in plain text; worth moving them out"* — and it goes under
*Reported, never scored*, with no `file:line` and no deduction. **This is not a security review and must
never become one:** say it once, in passing, and go back to the upgrade.

**Candidates are not findings.** Searches can't resolve intent. A `.Select()` in a view model building a
list is the Upside signal; the same call in startup configuration or a test helper is not, and an `.aar`
in the tree costs nothing if it belongs to a sample project. Open the file before you score it.

**Projections are not measurements.** The numbers in this skill come from named devices with a documented
protocol. They say what *kind* of gain to expect on what *kind* of code — nothing about the user's app
until they measure. Label every number with its confidence, and keep runtime gains, build-config gains and
app-code gains on separate lines: the last kind is available without upgrading, and adding them into one
"CoreCLR gives you −70%" would be false.

**Blocked is not stuck.** A gate means *you cannot move to .NET 11 today* — nothing more. .NET 10 is LTS
and supported until **14 November 2028**, so not upgrading is a decision with real runway rather than an
emergency. **Say the date, and work the remaining time out from today's** — never carry a fixed
"N years", which is wrong the moment this file is a year old. Say what the gate is, and say that. (Mono itself is gone: from .NET 11 preview 6 CoreCLR is
the only runtime on Android, iOS, Mac Catalyst and tvOS, and `UseMonoRuntime` no longer exists, so "stay
on Mono" means "stay on .NET 10", which is a version decision rather than a build flag.)

**Read, report, change nothing.** This skill analyses a repository and writes a report about it. It edits
no source file, no `.csproj`, no `Directory.Build.props`, no package reference, no TFM. **The only file it
creates is `coreclr-fit.md` at the solution root.** Its commands are read-only by construction —
`-getProperty` evaluates without compiling, `--list-sdks` reports, `unzip -l` lists — and evaluating may
populate `obj/` the way any restore does, which is the whole of its footprint. The snippets in this file
are content for the reader, never commands to execute.

That holds even when the request sounds like an instruction to act: *"migrate this app to .NET 11"* is a
trigger because the assessment is what answers it.

**This is about sequence, not refusal.** If they read the report and then ask for the fixes to be applied,
apply them — declining would strand them halfway. The order is what matters: **1.** run the analysis,
touching nothing · **2.** hand over the report, finished and unedited · **3.** only then, on an explicit
request, agree what is in scope and make the changes. Step 2 is the one that cannot be skipped, because a
report that quietly rewrote the repo while producing itself describes a project that no longer exists.

**Ready is not worth it.** Whether a project *can* move does not answer whether it *should*, and those are
different decisions usually made by different people. Two of the first three projects scored this way came
out inverted. Report both axes every time, and report a low Upside as the finding it is — *"ready, and it
will not pay you"* is the sentence nobody else will write for them.

## Workflow

### 0 · Scope

The app project is the `.csproj` with MAUI TFMs and `<OutputType>Exe</OutputType>`. That marker is read
from the file rather than evaluated — you have to pick a project before you can ask MSBuild anything
about it — so confirm it in step 1 instead of trusting it: an app evaluates `UseMaui=true` **and** an
`ApplicationId`; a MAUI class library has the first and not the second. Several apps and no path given
→ ask which; don't guess.

Libraries in the solution are **not scored as projects, but their contents can cost the app points** —
an Android class library without AppCompat, and Java/Kotlin bindings of your own, both have rules in D3.
Open them, and attribute the deduction to the app.

**Score the mobile TFMs only.** A `net*-windows` TFM is out of scope: Windows has always run on CoreCLR,
nothing about this change touches it, and asking MSBuild for `UseMonoRuntime` there answers a question
that does not apply. Say in the report that it was skipped and why, so nobody reads the omission as an
oversight.

Two checks stop the run before it starts. Both are one short answer, not a branch:

- **Xamarin.Forms** (`Xamarin.Forms` package, `FormsAppCompatActivity`) → stop in two sentences: the MAUI
  migration comes first, this skill doesn't score it.
- **Already on `net11` or later** → the upgrade already happened, so there is nothing to score. Scoring
  it anyway produces a high number that means nothing, because the largest block of D1 is distance
  travelled and this project has none left. Stop the *score*, not the tool — someone who upgraded and
  broke something is a real case, and the `description` lists "AOT crashes after upgrading". Say:

  > *You are already on net11 — the upgrade happened, so there is nothing to score. If something broke
  > after it, say so and I will run the source search and give you the findings without the score.*

### 1 · Effective build configuration

Get the TFMs first. Ask for **both spellings**: a single-target app declares `TargetFramework` and only
a multi-target one declares `TargetFrameworks`, so asking for the plural alone comes back empty on an
Android-only project and leaves you with nothing.

```bash
dotnet msbuild App.csproj -getProperty:TargetFrameworks -getProperty:TargetFramework
```

Then evaluate the properties that matter for Release **once per platform TFM** — the values differ per
leg and so do the rules they feed. `<TFM>` is one of theirs, never one of ours: a net8 or net9 project
is evaluated on its own TFM.

```bash
dotnet msbuild App.csproj -p:TargetFramework=<TFM> -p:Configuration=Release \
  -getProperty:UseMaui -getProperty:OutputType -getProperty:ApplicationId \
  -getProperty:UseMonoRuntime -getProperty:RunAOTCompilation -getProperty:PublishReadyToRun \
  -getProperty:PublishTrimmed -getProperty:TrimMode \
  -getProperty:EnableTrimAnalyzer -getProperty:SuppressTrimAnalysisWarnings \
  -getProperty:SupportedOSPlatformVersion -getProperty:RuntimeIdentifiers \
  -getProperty:MauiVersion -getProperty:ManagePackageVersionsCentrally \
  -getProperty:AndroidLinkMode -getProperty:AndroidEnableProfiledAot \
  -getProperty:AndroidAotProfiles -getProperty:AotAssemblies -getProperty:MtouchLink
```

A second run with `-p:Configuration=Debug` is worth it when Release looks surprising — the diff is often
the finding.

**The four `Android*`/`Aot*` properties are Android-only and `MtouchLink` is Apple-only. Expect them
empty on the other legs, and do not report an empty one as a finding.** It does have a consequence worth
stating: the Android rules in D1 cannot fire on a project with no Android TFM, so its Build & TFM score
is structurally higher than an Android project of the same quality. Say so where the dimension is
explained — the same way the Upside table states that an iOS-first product has a lower ceiling on the
Android reach row.

Also note `dotnet --version` and the nearest `global.json`, and **compare the SDK major with the
TFM major**. A project evaluated by a newer SDK gets that SDK's defaults, not its own: a net8 app
evaluated by SDK 9+ shows `RuntimeIdentifiers=android-arm64;android-x64`, but the .NET 8 Android SDK
defaults to `android-arm;android-arm64;android-x86;android-x64` (64-bit-only defaults started in
.NET 9). When they differ, say so in the report and don't score defaults that the project's own SDK
would set differently — check `dotnet --list-sdks` and prefer running with the matching one.

For every property that costs points, check whether it is **explicit** (set in the csproj or a
`Directory.Build.props`) or just the **evaluated default**. `SuppressTrimAnalysisWarnings=true` and
`AndroidEnableProfiledAot=true` are MAUI defaults nobody decided; a default is a note, an explicit
value is a finding. The rules below say which.

`-getProperty` returns the value and never its origin, so settle this with a search rather than a guess:
grep the property name across the app `.csproj` and every `Directory.Build.props` /
`Directory.Packages.props` up the tree. Absent from all of them → evaluated default → a note. Present
→ explicit → a finding, and you have the `file:line` the deduction needs. Only the properties that cost
points are worth the two searches.

If evaluation is impossible (no SDK, missing workload, restore fails and can't be fixed), read the csproj
and every `Directory.Build.props` / `Directory.Packages.props` up the tree by hand — and say in the
report that values were read, not evaluated.

### 2 · Source search

Search the solution (skip `bin/`, `obj/`, generated `*.g.cs`, test projects) with whatever search tool
you have. Record count and `file:line` for each:

| Key | Pattern (regex) | Files |
|---|---|---|
| Type enumeration | `\.(GetTypes\|GetExportedTypes\|DefinedTypes)\b` | `*.cs` |
| Type by name | `Type\.GetType\("`, `Assembly\.Load`, `Activator\.CreateInstance` | `*.cs` |
| Generic over runtime types | `MakeGenericType\|MakeGenericMethod` | `*.cs` |
| Member by string | `Get(Property\|Method\|Field)\("` | `*.cs` |
| DI by assembly scan | `\.Scan\(\|Scrutor\|FromAssembl` | `*.cs`, `*.csproj` |
| Reflection mapper / ORM | `AddAutoMapper\|: Profile\b\|Mapster`, `sqlite-net-pcl\|LiteDB\|Realm` | `*.cs`, `*.csproj` |
| Newtonsoft | `Newtonsoft\.Json` | `*.csproj`, `*.props` |
| STJ source-gen present | `JsonSerializerContext` | `*.cs` |
| Trim annotations present | `DynamicallyAccessedMembers\|RequiresUnreferencedCode\|TrimmerRootDescriptor` | `*.cs`, `*.csproj` |
| Compiled bindings | `x:DataType=` | `*.xaml` |
| XAML that binds (the `x:DataType` denominator) | `\{Binding\|\{TemplateBinding` | `*.xaml` |
| `x:Code` blocks | `<x:Code` | `*.xaml` |
| `RelativeSource` element form | `<Binding[^>]*RelativeSource=` | `*.xaml` |
| **Every** `PackageReference` — the D3 package rule tests assets, so you need the whole list, not a filtered one | `<PackageReference Include="([^"]+)" *Version="([^"]+)"` | `*.csproj`, `*.props` |
| Names worth opening first (a hint, never the verdict) | `Xamarin\.\|Xam\.Plugin\|Plugin\.(Permissions\|Connectivity\|Geolocator\|Media\|Settings)\|monoandroid` | `*.csproj`, `*.props` |
| Native bindings | `<(AndroidLibrary\|LibraryProjectZip\|EmbeddedNativeLibrary\|NativeReference)`, plus any `*.aar` / `*.jar` / `*.so` in the tree | `*.csproj` |
| AndroidX pinned explicitly | `Include="Xamarin\.AndroidX[\w.]*" *Version=` | `*.csproj`, `*.props` |
| Crash/analytics SDKs **as packages** | `Include="[\w.]*(Firebase\|Crashlytics\|AppCenter)` | `*.csproj`, `*.props` |
| MAUI pinned per project | `Include="Microsoft\.Maui[\w.]*" *Version=` | `*.csproj` |
| Startup reflection | type enumeration patterns above | `MauiProgram.cs`, `App.xaml.cs`, `AppShell.xaml.cs` |
| LINQ over collections | `\.(Select\|Where\|OrderBy\|OrderByDescending\|GroupBy\|SelectMany\|Aggregate)\(` | `*.cs` |
| Date parsing / formatting | `(TryParseExact\|ParseExact\|ToString\("[dMyHms]` | `*.cs` |
| String building / mapping | `new StringBuilder\|\.Append(Line\|Format)?\(\|string\.Join\(` | `*.cs` |
| Regex | `new Regex\(\|Regex\.(Match\|Replace\|IsMatch\|Split)\(\|\[GeneratedRegex` | `*.cs` |
| Startup logging | `Console\.WriteLine\|EnableDebug\(` outside `#if DEBUG` | same three files |
| Sync native SDK init | `(Firebase\|Crashlytics\|AppCenter\|Sentry\|OneSignal)[\w.]*\.(Init\|Start\|Configure)` | `MauiProgram.cs`, `MainActivity.cs`, `MainApplication.cs`, `AppDelegate.cs` |
| Blocking at startup | `\.Result\b\|\.Wait\(\)\|GetAwaiter\(\)\.GetResult` | `MauiProgram.cs`, `App.xaml.cs` |
| Second TFM for comparison | a `net11.0-*` TFM alongside the current one | `*.csproj`, `*.props` |
| Startup marker | `MauiProfilingMarker\|ProfilingHelper\|Stopwatch` near first `OnNavigatedTo` | `*.cs` |

Then open the first page (`MainPage.xaml` or the Shell start route) and count elements (`<[A-Za-z]`),
noting how many are custom views that contain real controls — those weigh far more than the count says.
If the XAML looks empty, read the code-behind: a `TabbedPage`/`FlyoutPage` whose constructor does
`Children.Add(new Page())` nine times inflates nine pages before the first frame, and the XAML count
says 1. Count what the constructor builds.

Both XAML rules are found by grep alone — `<Binding RelativeSource=` in element form and `x:Code` — so
no type resolution is needed. **Do not go looking for non-string properties bound to string targets:**
that rule was tested on both runtimes and refuted, so the hits are not findings (see D2).

For the candidate rows, verify before scoring: open each reflection hit, and check that a LINQ or date
hit sits on a path the user waits for rather than in startup configuration or a test helper.

**Three rows feed something other than a deduction, and the report should say what.** Trim annotations
already in the code mean the team has done part of the `Full` trimming work, which changes the tone of
that note from a warning to a credit. A `net11.0-*` TFM already sitting next to the current one, and a
startup marker already in the code, are what decide the measurability paragraph: both present means they
can A/B today, and neither is worth points either way.

### 3 · Ask what the search can't know

Only the questions that change the verdict, asked together — seven lines in one message, not an interview:

| Ask | What it decides | If nobody answers |
|---|---|---|
| Must you ship 32-bit Android (armeabi-v7a / x86)? Does any native dependency lack arm64? | **Gate G1** | Assume **no**. Assuming yes would return "Blocked" on every unattended run, which is the one wrong answer that costs them the whole report |
| Can Android `minSdk` go to 24? | **Gate G2**, or a −3 in D1 if it can | Assume **it can**, and say the deduction is provisional |
| Are you on `Full` trimming (`AndroidLinkMode=Full` / `TrimMode=full`), and has a Release build been validated on a device? | The **tone** of the `Full` trimming note, not whether it appears | Assume `SdkOnly` and never validated, which is both the common case and the milder claim |
| What share of your installs is Android vs iOS? | The **Android reach** row of Upside (0–12), which is a share of users, not of TFMs — the csproj says both platforms exist, never which one the product lives on | Split it evenly across the platform TFMs and say you did |
| Is the pain startup, or throughput once the app is running? | **Which projection to cite** | Assume throughput: it is where the measured gains are |
| Are sessions short (seconds, then killed) or long? | The **session length** row of Upside (0–18) | Assume short, which scores lower |
| Is this a production app, or a sample/demo? | How the **measurability** paragraph reads — a production app that cannot A/B should be told so plainly; nobody A/Bs a demo. Never points, either way | Assume production, which is the reading that owes them more |

**Ask first, and ask twice before assuming.** The defaults above exist for the run nobody is watching,
not as a shortcut past a conversation — every one of them is a guess at something only they know. If the
first message goes unanswered, say plainly that these seven answers change the verdict and ask again.

Only then fall back to the defaults, and when you do, mark them as what they are. In the report, under
*What was analyzed*, write the assumptions under a heading that says they were **taken automatically**,
and close with: *"Nobody answered these, so they were assumed. If any of them is wrong the verdict moves
— the 32-bit question alone can turn this into a blocked upgrade. Worth one re-run with someone
available to answer."* An assumption the reader does not know was automatic is worse than no answer at
all.

### 4 · Score

Three dimensions, 100 points (`50 · 20 · 30`), plus gates. Two whole categories that earlier versions scored — reflection and startup — are now reported and never scored; see below. Deduct from each dimension's maximum, never below 0. Show the arithmetic so the user can recompute it. When a rule
clearly doesn't fit (a `GetTypes()` behind `#if DEBUG`), skip it and say why — judgment beats the table.

**Terminology — say "upgrade", not "migration".** .NET 9 or 10 → .NET 11 is an *upgrade*: same stack, a
version forward. In this ecosystem "migration" already means Xamarin.Forms → MAUI, and borrowing that
word makes a week of work sound like a quarter of it. Three uses are deliberate: Xamarin.Forms → MAUI,
Newtonsoft → System.Text.Json or any move between two libraries, and the `description`, which is what the
skill *listens* for rather than what it *says*. It is not a plain version bump either, and the report
should not imply it is — the runtime changes underneath them whether they planned it or not.

**Each rule fires once, present or absent — never per occurrence.** One `x:Code` block or six deduct the
same: **this axis counts how many kinds of hazard you have, not how much of each.** Volume decides how
long it takes, which is what the fix list's scope line is for. Two rules are banded by count and both say
so in the table — packages left behind, where each one is a separate "find the replacement" task with its
own maintainer, and `x:DataType` coverage, which is a proportion by construction.

**Every rule set sums to less than its dimension's maximum**, so a heavier repo always scores below a
lighter one instead of both hitting the floor. The worst case allowed by each table is −49 of 50, −19 of
20 and −25 of 30, and that margin is load-bearing — **re-check it whenever a rule value changes**.

**That −49 assumes an Android TFM exists, and on a project without one it is wrong.** Four D1 rules are
Android-only — `minSdk`, the 32-bit RIDs, the invalid AOT combination and Mono AOT tuning, 21 points
between them — and both gates are Android-only too. An iOS-only project therefore cannot score below 24
on D1 and can never come back "Blocked", not because it is in better shape but because most of the table
cannot reach it.

**The scale is not renormalised for this.** The card exists to be compared against another
project's, and a score computed on a different denominator in each report destroys exactly that. Declare
it instead. One Android TFM is enough for the rules to be live, so this applies only to a project with no
Android leg at all, and then the dimension's section says so in full:

> *No Android TFM in this project, so 21 points of this dimension could not fire: `minSdk`, 32-bit RIDs,
> the AOT combination and Mono AOT tuning are Android rules. Read 43/50 as "nothing wrong in what
> applies", not as "better than an Android project that scored 38".*

That last sentence is the one doing the work: without it the number lies by omission. The Upside axis
carries the mirror image of this on its Android reach row.

**What the points measure.** Readiness scores the *work and risk* between this repo and a CoreCLR build
that runs, weighted by what it costs to close — including the research needed to find it. **It does not
score urgency.** Urgency is carried by the severity marks and the "Breaks the build" section, which is
why the two can look inverted and both be right: a build-breaking XAML form worth 7 points opens the
report while a 15-point pile of Xamarin-era packages that builds fine today sits further down. **When the
score and the marks seem to disagree, the marks answer "what do I do on Monday".** Say so in the report
rather than letting the reader reconcile it alone.

**Gates — cap the total at 30, verdict "Blocked"**

- **G1 · 32-bit Android is required** (business rule, or a native dependency with no `arm64-v8a`).
  CoreCLR on Android is 64-bit only; there is no configuration around it. Listing 32-bit RIDs is not
  the gate — needing them is.
- **G2 · Android `minSdk` < 24 and it cannot be raised.** .NET 11 requires API 24. If it *can* be
  raised it is not a gate, just the −3 in D1.

**Every value in the `Pts` column is exact. There are no ranges to interpolate inside.** A rule either
fires at its stated value or does not fire at all. Two rules in this whole skill are *banded* — they are
marked `banded` and the band is a function of a count you can verify, not of judgment. The three TFM
rows look like a range and are not: they are three mutually exclusive rules and exactly one fires.

**If you cannot determine whether a rule fires, it did not fire.** Say so, name the rule and say what
you could not check. **Never estimate a value, never interpolate one, and never invent a rule that is
not in these three tables.** A score built from estimates is not this skill's score, and the card
promises it is reconstructible — a reader who knows the rules will check.

**D1 · Build & TFM — 50** (the dominant factor in whether you can move at all: distance plus the
configuration that has to change. Mechanical and predictable — every point here is a known edit with a
known cost, none of it is a research task)

| Rule | Pts | Why |
|---|---|---|
| TFM is `net9` | −7 | Distance to travel, not a defect: one version back. |
| TFM is `net8` | −15 | Two versions, each with its own breaking changes and workload bumps. |
| TFM is `net7` or older | −25 | Expect an intermediate hop; nobody goes straight to 11. |
| Android `SupportedOSPlatformVersion` < 24 | −3 | Build fails otherwise; trivial change, needs sign-off on dropping Android 6. |
| `RuntimeIdentifiers` lists `android-x86`/`android-arm` alongside arm64 | −7 | CoreCLR on Android is 64-bit only. Must drop them. |
| `RunAOTCompilation=true` with `AndroidLinkMode=None` or `PublishTrimmed=false` in Release | −9 | Invalid by design (XA1030). Means Release was never validated on device. |
| `PublishReadyToRun=true` not conditioned on the TFM, **and a second older TFM is present** | −2 | Fine on net11, breaks the older leg kept for comparison. Without that second leg nothing is broken and this does not fire — the `Second TFM` row of step 2 is what tells you. |
| Mono AOT tuning set **explicitly** (`AndroidAotProfiles`, `AotAssemblies`) | −2 | Does nothing on CoreCLR; the equivalent is R2R + a MIBC profile. An evaluated default is not this. |
| No `global.json` / SDK not pinned | −1 | A preview SDK needs `allowPrerelease` and a pinned version or teammates build different things. |
| Central Package Management in use | +0 | Mention as a positive: the MAUI 11 bump is one line. |

**D2 · XAML the .NET 11 toolchain rejects — 20** (small but absolute: if either of the first two fires,
nothing compiles until it is fixed)

| Rule | Pts | Why |
|---|---|---|
| `<Binding RelativeSource="…">` element form | −7 | MAUIX2002: not a bindable property in element syntax. Use `{Binding …, Source={RelativeSource …}}`. Reproduced on preview 5. |
| `x:Code` block | −6 | MAUIG1001 from inside the XAML generator, plus a cascading CS8795. Fails even with `EnablePreviewFeatures=true`. **Preview-5 state and the likeliest of these to disappear** — re-check at GA. |
| `x:DataType` coverage · `banded` | 0 all views · −3 most · −4 fewer than half · −6 none | Not a break. Compiled bindings turn a type mismatch into a build error instead of a silent default — and, as it happens, they are also what keeps reflection-serialized properties alive under aggressive trimming. |

> **The denominator is XAML that binds, not every `.xaml` in the repo.** `App.xaml`, `AppShell.xaml`,
> resource dictionaries and style files have no binding context and nothing to type; counting them
> produces a deduction the team cannot fix, which is the fastest way to lose a reader. Build the
> denominator from files that contain `{Binding` and have a `BindingContext` — pages, views, and a
> `ControlTemplate` or `DataTemplate` that binds. In the reference run this was the difference between
> "8 of 29 untyped" and full coverage with a single `ControlTemplate` left over.

> **Not a rule here: type coercion in bindings.** Binding a non-string property to a string target used
> to deduct, on the premise that CoreCLR stopped coercing. It was reproduced twice on preview 5 and
> nothing threw on either runtime, so it was removed. **Do not reintroduce it without a reproduction.**

**D3 · Dependencies — 30** (a package with no net11 or arm64 assets genuinely blocks, and unlike
everything else here the fix is often outside your control: you escalate to a maintainer)

| Rule | Pts | Why |
|---|---|---|
| Packages left behind · `banded` · **test by assets, not by name** | 0 none · −6 one · −10 two or three · −15 four or more | Old TFMs resolving via fallback; may lack arm64/net11 assets or trim-clean code. Each is a "find the replacement" task, and several have no maintainer. **How to decide which packages count is below the table** — the name does not tell you. |
| Java/Kotlin bindings or embedded native libs of your own **with no `arm64-v8a` slice, or none you could verify** | −4 | They must be rebuilt for arm64, and it fails at runtime rather than at compile time. Open the archive before scoring — `unzip -l lib.aar \| grep jni/` — because one that already ships arm64 costs nothing. If you cannot inspect it, deduct and say the check is pending. |
| AndroidX packages pinned to explicit versions instead of resolving through MAUI's graph | −2 | `NU1608` at restore when the MAUI bump moves the pinned range: MAUI pins the AndroidX line, an explicit pin that sits outside it fails the restore, and realigning them is part of the upgrade. `Lifecycle.LiveData.*` is the one that shows up most. Not a runtime failure. |
| Firebase / Crashlytics / AppCenter | −2 | Binding-heavy, historically the slowest to publish new-TFM assets. |
| MAUI versions pinned per csproj instead of one `MauiVersion` | −2 | The workload and the pin drift apart; the bump becomes N edits instead of one. |

> **Which packages are "left behind" — open the assets, do not read the name.** The name is a hint for
> where to look first and nothing more, and it is wrong in both directions. `Microcharts.Maui` is named
> for the current framework and ships `net6.0-*` only. `Xamarin.AndroidX.Core` carries the old brand and
> ships `net10.0-android36.0`. Scoring either one from its name is a mistake the reader can see.
>
> The assets are on disk after any restore, so this is a read-only look, no network:
>
> ```bash
> ls ~/.nuget/packages/<package.id>/<version>/lib/
> ```
>
> The folder names are the frameworks it publishes for. **A package counts when either of these is
> true:**
>
> - It ships `monoandroid*`, `xamarin.ios*` or `xamarin.mac*` assets. Genuinely Xamarin-era: those
>   resolve onto a .NET TFM by fallback and were built before arm64-only and before trimming mattered.
> - Its newest platform asset (`net*-android`, `net*-ios`) is **three or more majors behind** the TFM they
>   are moving to — `net8.0-android` or older against a `net11.0-android` target. Three release cycles
>   untouched, whatever it is called. **A package one or two majors back is current**: shipping
>   `net10.0-android` the month .NET 11 lands is normal maintenance, not neglect, and deducting for it
>   would punish almost every healthy dependency.
>
> **`netstandard2.0` or `netstandard2.1` alone does not count.** That is a portable library by design, it
> resolves cleanly on every modern TFM, and deducting for it would hit half the ecosystem.
>
> **Do not count what another rule already counts.** Firebase, Crashlytics and AppCenter have their own
> −2 row, and explicitly pinned AndroidX packages have theirs. A package caught by one of those is not
> also counted here — *do not deduct twice for one cause*, which is the same rule that keeps reflection
> out of this axis.
>
> **A package with no replacement still counts, and the report says so.** The deduction is for *this is
> not keeping up and you do not control it*, which is true whether or not something exists to swap in.
> What changes is the fix, not the risk — so put each counted package in one of three situations, by
> name, in the fix list:
>
> - **A current replacement exists** → swapping it is ordinary work, and the scope line says how much.
> - **No replacement, maintainer still active** → do not plan a rewrite. Test it against net11 first and
>   open an issue early; most of these publish within a release.
> - **No replacement, nothing published in years** → the one that can actually block. Say it plainly and
>   early, because the answer may be writing the binding yourself or dropping the feature, and that is a
>   product decision with a lead time.
>
> **Never write "replace this" for a package in the second or third situation.** It is advice the team
> cannot act on, and it is how a fix list loses its reader.
>
> **If you cannot look** — the package was never restored, or there is no `lib/` because it is a native
> or build-only package — say the check is pending and name the package, do not deduct. An unverified
> deduction is exactly what this rule was rewritten to stop. List those packages under *Reported, never
> scored* so the reader knows the first thing the net11 restore will tell them.

> **Not a rule here: a missing `Xamarin.AndroidX.AppCompat` reference.** It used to deduct as a net11
> breakage and no source supports that — a library that uses AppCompat types needs the reference on any
> TFM. The one real case is a *binding* project wrapping Java classes that extend `AppCompatActivity`:
> report it if the binding fails, score nothing.

**Write it out like this, and let the rows produce the total.** This is the form the report uses, and
the reason it uses it is that an invented value cannot survive it — the column of rules is checkable
against the three tables above, and the column of `file:line` is checkable against the repo.

```
D1 · Build & TFM                                              50
  TFM is net9                              −7   App.csproj:4
  Android SupportedOSPlatformVersion 21     −3   App.csproj:25
  RuntimeIdentifiers lists android-arm      −7   App.csproj:137
  no global.json                            −1   —
                                          ────
                                           −18   →  32/50

D2 · XAML                                                     20
  x:DataType · banded · 9 of 24 bare → most  −3   24 files that bind
                                          ────
                                            −3   →  17/20

D3 · Dependencies                                             30
  Xamarin-era · banded · 2 found → two or three  −10   App.csproj:180,186
  AndroidX pinned explicitly                      −2   App.csproj:186-195
  Firebase                                        −2   App.csproj:180
                                               ────
                                                −14   →  16/30

Readiness  32 + 17 + 16  =  65/100
```

Four things that example is showing on purpose. Every value appears verbatim in a table above. The two
banded rules print the count that selected the band, so the choice is auditable. A rule with no
`file:line` says why (`global.json` is an absence; there is no line to point at). And the dimension
totals are a sum of the rows above them, never a figure chosen first.

### Reported, never scored

Two whole categories are deliberately absent from this axis, and the reasoning is the same for both:
**they do not affect whether the project can move.** Readiness answers *can we*; scoring things that run
fine today made the number say "you are not ready" to a team that could ship the upgrade tomorrow.

**Reflection and serialization.** `GetTypes()`, `Activator`, reflection mappers and ORMs, Newtonsoft,
STJ without a context — under `SdkOnly`, which is CoreCLR's production configuration today, the trimmer
leaves the app's own assembly alone and **all of this runs as it is**. What it does is cost you the gain,
and that already has an axis: the serialization row of Upside is exactly this, and object-to-object
mapping sits inside the CPU-bound row. Report every finding with `file:line` and scope, tag it `◐`, and
let Upside carry the consequence. **Do not deduct twice for one cause.**

**Startup path.** An eager first page, startup reflection, sync native init, blocking calls. These are
worth fixing — the largest single win in the whole investigation, **−52%** of cold start on Mono and
**−53%** on CoreCLR — but that gain **lands on whatever they ship today**, with no upgrade involved. Report them as `○` with scope *and the magnitude
we measured*, so the number is theirs to act on without waiting for anything: a first page that builds
its whole tree up front was `~1.5 s of a 2.9 s` cold start, and each synchronous native SDK init sitting
on the startup path was `50–200 ms`. Say plainly that neither one requires this upgrade, and score
nothing.

Same for **measurability**: whether they can A/B is advisory, never points.

### The Full trimming note — include it whenever reflection findings exist

**Reflection findings decide whether this note appears; step 3's trimming answer decides its
temperature.** On `SdkOnly`, which is where almost everyone is, it reads as *this runs fine today, and
here is what changes if you ever switch* — a hypothetical. On a project already building `Full` without
having validated a Release on a device, the same words are a live risk and should be written as one:
*this can be happening now and you would not know*. Do not drop the note because they are on `SdkOnly`;
the mechanism is what makes the source-generated fix worth doing.

`SdkOnly` is today's configuration and everything above runs under it. `Full` or NativeAOT is a different
story, and the report owes the reader an accurate version of it rather than a scare:

> Under `SdkOnly` this runs as it is. Under `Full` the trimmer walks your assembly too, and a property
> that is only ever touched by a reflection serializer looks unreachable — nothing in code calls the
> getter. Whether it survives depends on something *else* keeping it alive: a compiled binding, a log
> line, a `ToString()`. That is not a property anyone designed, and it fails in the ugliest way — the
> DTO you display survives, the one you only serialize comes back with nulls, and it passes QA because
> QA walks the screens. **We did not measure this**; it is the mechanism, not a result. A
> source-generated STJ context removes the question entirely, because the generator emits ordinary code
> the trimmer can see.

### 4b · Score the second axis: Upside

Readiness answers *can we upgrade*; it does not answer *should we*. A single number hides exactly the
thing the team needs to decide.

Upside is a **weighting of the measured evidence by this app's profile**: how much of the gain we
measured can this app actually capture. It is not a prediction of speed, and it is reported as a band —
`LOW` / `MEDIUM` / `HIGH` — precisely because a percentage invites "percent of what?" and gets quoted as
if it were a speed figure. Score 100 points internally (they make runs and projects comparable, and they
drive the delta on a re-run), show the band, and put the points in the Upside table in the body:

**These four values are banded, exactly like the D2 and D3 rules, and for the same reason.** Earlier
versions gave a range per signal and nothing else, so two honest runs of the same repo could differ by
more than twenty points on one row and land in different bands — which is the whole verdict. A range is
not a rule. **Pick the band from a count you can show, never a number that feels right.**

| Signal | Pts | How to read it |
|---|---|---|
| CPU-bound managed work on the critical path (parsing, transforming, sorting collections, formatting) · `banded` | 0 · 12 · 25 · 34 · 45 | **What counts, with what we measured for each:** LINQ over collections — `Select`, `Where`, `OrderBy`, `GroupBy` (`−44%` time, `1.8×`) · date parsing and formatting (`−53%`, `2.1×`) · string building and object-to-object mapping (no figure of their own; same shape). **Score from counted hits, not impression** — this is the heaviest row on the axis and the easiest to inflate. The step 2 patterns are what earn it, and a hit only counts when it sits on a path the user waits for (a view model building a list, a service shaping a response), not in startup config or tests. It is the widest row because this work is everywhere and runs on every screen, even though each case gains less than JSON. An app whose screens are layout and native handlers scores near 0 — the UI thread measured ≈ 1×. |
| Serialization that can use STJ source-gen · `banded` | 0 · 2 · 12 · 25 | Full points when STJ + a context is already there. Half when it's STJ without a context — mechanical to add, and it lands in the measured configuration. Half too on Newtonsoft over own models, but for a different reason: that gain was **never measured**, and the points are for the trim-visible destination being reachable, not for a figure. Near 0 when there's barely any serialization. |
| Session length · `banded` | 4 · 10 · 18 | Long-lived sessions reach tier-1 and collect the steady-state gain; apps opened for seconds and killed live on the cold-start row of the gains table instead, and score low. |
| Android reach (the MIBC startup profile) · `banded` | 0 · 2 · 6 · 12 | Score it by how much of the product is Android — all of it → 12, one of two platforms → ~6, a minor platform → ~2 — and ask, because this is a share of installs, not of TFMs. **Why it belongs on an upgrade axis:** the gain comes from a custom MIBC profile feeding `crossgen2`, which is a CoreCLR mechanism. The figures live in the MIBC rows of the gains table; quote them from there and never from here. Two shortcuts to avoid: the Mono baseline was **not** handicapped, it ran profiled AOT with the SDK's own default `aotprofile`, and Mono does have an equivalent mechanism (a custom `*.aotprofile`) that **we did not measure**, expected to gain less because its default profile is already well tuned and it has no tiered JIT to recover afterwards. So: *measured on CoreCLR, Mono counterpart unmeasured and expected smaller* — never *it only exists on CoreCLR*. |

**The bands, and what selects each one.**

`CPU-bound managed work` — count the **verified** hits on paths the user waits for: LINQ over
collections, date parsing and formatting, string building, object-to-object mapping. A hit in startup
config, a test helper or a background job does not count, and step 2 says to open the file before
counting it.

| Verified hits on waiting paths | Pts |
|---|---|
| none — the screens are layout and native handlers | 0 |
| 1 to 5 | 4 |
| 6 to 25 | 12 |
| 26 to 100 | 25 |
| 101 to 300 | 34 |
| more than 300 | 45 |

> **Then take one step down when the collections are small** — tens to low hundreds of items: API pages,
> pickers, settings lists, a cart. The measured gains came from shaping real lists, and an app that runs
> `Select` three hundred times over twelve items each is not the app those figures describe.
>
> Two limits on that step. **There is no step up** — the top band already assumes volume. And **it never
> takes a repo with real hits down to 0**: the floor is the `1 to 5` band, because "no gain at all" is a
> claim about an app with nothing on its waiting paths, not about one with small lists. Say in the report
> which band the count selected **and whether the step down was applied**, so the reader can check both.

`Serialization` — **25** when STJ with a `JsonSerializerContext` is already in place · **12** when it is
STJ without a context, or Newtonsoft over your own models · **2** when serialization is incidental
(a settings blob, a cached token) · **0** when there is essentially none.

`Session length` — **18** long-lived (a shift, a workday, a session someone leaves open) · **10** mixed
or unknown after asking · **4** short (opened, used for seconds, killed). This is step 3's question; the
default when nobody answers is short, which is the lower score.

`Android reach` — **12** Android only · **6** one of two platforms · **2** Android is the minor platform
· **0** no Android TFM. Score it by **share of installs**, not share of TFMs, and ask — the csproj says
both platforms exist, never which one the product lives on.

> **Recoverable startup weight is deliberately not a signal here.** It was the single largest win in the
> investigation, but it is app code and **lands on whatever they ship today**. Scoring it would let a
> project cross into a higher band on work it can do without upgrading at all, while the caption beside
> the band promises runtime gains. It stays in the fix list and in its own row of the gains table, where
> its independence is stated.

Bands: **LOW** 0–35 (it will not pay you) · **MEDIUM** 36–65 (some screens gain) · **HIGH** 66–100
(the profile matches what we measured).

> **These four rows are not the whole universe, and the report should not imply they are.** Three axes
> are deliberately absent because nothing was measured well enough to score them, and saying so is worth
> more than a fifth row that pretends otherwise:
>
> - **GC and allocation pressure.** Measured, but the result is ambiguous — proportionally larger spikes,
>   smaller in absolute terms. An allocation-heavy app could land either way. Report the finding, score
>   nothing.
> - **Regex.** Not measured at all, and the abstraction law predicts it would be one of the larger gains.
>   Step 2 counts it, so say how many hits there are and that it is a likely win we have no number for —
>   a count with no figure is still worth more than silence.
> - **iOS.** The Android reach row is Android-only by construction: its gain comes from a MIBC profile
>   feeding `crossgen2`, which has no iOS counterpart in what we measured. An iOS-first product therefore
>   has a structurally lower ceiling on this axis. State it rather than letting the band look like a
>   verdict on the whole product.

Score it honestly in both directions: a low Upside is a useful answer, not a failure of the analysis.
"Ready, but it won't pay you" is exactly the sentence nobody else will write for them.

**Never print either axis without its caption.** `HIGH` on its own says nothing about what improves;
the right-hand caption (`up to 44% less time on list work`) is what makes it concrete and what
stops it being read as a blanket speed claim. Same for Readiness: the caption says what kind of work is
left.

**The caption is the conservative line, not the headline.** Its whole job is to stop the band being read
as a speed promise, so it may not carry the biggest number in the table. Two rules, both mechanical:

**1 · It quotes only what this app gets on the code it has today.** Anything that needs work first goes
in the body with the work named, never in the caption.

**2 · It never carries the JSON figure, and never the ceiling.** The JSON row is the heaviest in the
table and the most conditional: `2.9×` in the published session, `9.9×` in a longer one — 10,000
identical objects with tier-1 fully warm, conditions no real screen reproduces. In the gains table those
travel with the "applies when" column that makes them true. **A caption travels alone**, into a
screenshot and a meeting, and that is where either figure stops being a measurement and becomes a
promise. Name the row this repo actually earned instead — `up to 44% less time on list work` for one
that sorts and filters, `up to 53% less time on date work` for one that parses dates in bulk — and let
the body say what the JSON path is worth and what it costs to get there. The reader still gets that
figure two paragraphs down, with its conditions attached, which is the only form in which it is true.

**On a reflection serializer there is no JSON figure at all.** Newtonsoft, reflection mappers and ORMs
were never run on both runtimes, and the steady-state number is no safer a fallback, because the JSON
phase is very nearly the whole of that gap. Report the path, not a number. And say the part that is
easy to leave out: **moving off the reflection serializer is app-code work that pays on the version they
are on today**,
with the measured runtime gain sitting *after* it. Two steps, and only the second one is this upgrade —
same shape as the startup findings, and it gets the same plain treatment.

## Fix snippets

**Everything below is content for the report, not commands to run.** These blocks get adapted to their
names and pasted into the fix list and the upgrade steps; the reader decides when to apply them. Two of
them — the A/B build and the `adb` timing commands — compile the app and deploy it to a device, and the
A/B also needs a second TFM added to the `.csproj` first. **Never execute any of this** (see
*Read, report, change nothing*). The only commands this skill runs are the read-only ones in step 1.

Adapt names; keep the conditions — most upgrade bugs were a property applying to the wrong runtime.

**Release Android matrix, per TFM.** From .NET 11 the runtime follows the TFM: `net11.0-android` is
CoreCLR and there is no switch. `<OLD>` below is the TFM they are on today — `net9.0-android`,
`net8.0-android`, whichever it is — and that block is only worth keeping while they hold it to compare
against.

```xml
<!-- net11 · CoreCLR, the only runtime -->
<PropertyGroup Condition="'$(Configuration)' == 'Release' and $(TargetFramework.StartsWith('net11.0-android'))">
  <AndroidLinkMode>SdkOnly</AndroidLinkMode>
  <PublishReadyToRun>true</PublishReadyToRun>
</PropertyGroup>

<!-- <OLD> and earlier · Mono. Full because SdkOnly has a verified Mono regression
     in the .NET 10/11 preview (dotnet/maui#33032). -->
<PropertyGroup Condition="'$(Configuration)' == 'Release' and $(TargetFramework.StartsWith('<OLD>'))">
  <AndroidLinkMode>Full</AndroidLinkMode>
  <RunAOTCompilation>true</RunAOTCompilation>
</PropertyGroup>
```

> **Deliberately not in that block: `EmbedAssembliesIntoApk` and `AndroidUseAssemblyStore`.** Both were
> set on both runtimes to make the benchmark APKs comparable, never measured as recommendations, and
> `AndroidUseAssemblyStore=false` opts out of a default that speeds assembly loading. Do not suggest
> them. If the reader already has them set, say they were measurement settings, not advice.

**`MauiEnableFullReadyToRun` — the one new lever this upgrade brings.** In Release, CoreCLR does
**partial** ReadyToRun by default; the full pass is opt-in:

```xml
<MauiEnableFullReadyToRun>true</MauiEnableFullReadyToRun>
```

It trades APK size for startup, in that direction. **We did not measure it** — our runs predate the
partial default — so offer it as a lever to try with the A/B below, never as a figure. It belongs in the
fix list next to the MIBC profile: both are build-config work, both Android, both measurable by them in
an afternoon.

**Side-by-side A/B — one line plus IDs.** Their baseline is the app they ship today, so there is no
switch to build: add the new TFM next to the current one and give each a distinct `ApplicationId` so
both can be installed at once. Use their TFM and their package id, not these placeholders.

```xml
<TargetFrameworks><OLD>;net11.0-android</TargetFrameworks>

<PropertyGroup Condition="$(TargetFramework.Contains('-android'))">
  <RuntimeIdentifiers>android-arm64;android-x64</RuntimeIdentifiers>
  <ApplicationId Condition="$(TargetFramework.StartsWith('<OLD-PREFIX>'))">com.company.app.old</ApplicationId>
  <ApplicationId Condition="$(TargetFramework.StartsWith('net11.'))">com.company.app.net11</ApplicationId>
</PropertyGroup>
```

```bash
dotnet build -c Release -f <OLD> -t:Run              # what they ship today · Mono
dotnet build -c Release -f net11.0-android -t:Run    # after the upgrade · CoreCLR
```

This measures **the upgrade**, not the runtime in isolation — MAUI and the BCL move with it. That is
also the decision they are actually making.

Measure each one the same way, from a cold process:

```bash
adb shell am force-stop com.company.app.net11
adb shell am start -W -n com.company.app.net11/<activity> | grep TotalTime
```

Ten cold starts per TFM, median and range — never the mean; GC spikes and JIT tiering hit the runtimes differently.

**Android minimum + 64-bit only:**

```xml
<SupportedOSPlatformVersion Condition="$(TargetFramework.Contains('-android'))">24.0</SupportedOSPlatformVersion>
```

**`x:Code`** — no fix beyond removing it: it does not compile on preview 5 even with
`EnablePreviewFeatures=true`. Move the code to the code-behind partial.

**MAUIX2002** — `<Binding Path="Title" RelativeSource="…"/>` → `{Binding Title, Source={RelativeSource AncestorType={x:Type v:CardView}}}`.

**Route registration without `GetTypes()`:**

```csharp
Routing.RegisterRoute(nameof(OrdersPage), typeof(OrdersPage));   // explicit, trim-safe, ~free at startup
```

**STJ source-gen** (replaces Newtonsoft / reflection STJ):

```csharp
[JsonSerializable(typeof(List<OrderDto>))]
internal sealed partial class AppJsonContext : JsonSerializerContext { }
var orders = JsonSerializer.Deserialize(json, AppJsonContext.Default.ListOrderDto);
```

**See trim warnings before CoreCLR shows them as crashes:** `<EnableTrimAnalyzer>true</EnableTrimAnalyzer>`
and `<SuppressTrimAnalysisWarnings>false</SuppressTrimAnalysisWarnings>` in Release; every `IL2xxx` is a
work item.

**Lazy first page** — a `DataTemplate` captures the XAML recipe without instantiating it (CommunityToolkit's
`LazyView<T>` only defers a *type* via `new T()`, so many instances of one view with different inline
content come out identical and empty):

```csharp
[ContentProperty(nameof(Template))]
public sealed class LazyContentView : ContentView
{
    public DataTemplate? Template { get; set; }
    public void Load() { if (Content is null && Template is not null) Content = (View)Template.CreateContent(); }
}
```

```xml
<v:LazyContentView x:Name="Card7"><DataTemplate><v:HomeCardView Title="…"/></DataTemplate></v:LazyContentView>
```

Materialize in small batches from `OnNavigatedTo` via `Dispatcher.DispatchDelayed(…)` after the first
frame. Logging: `#if DEBUG` around `options.EnableDebug()` and friends.

## Things the reference upgrade hit that are easy to miss

The first three are troubleshooting knowledge — use them when the symptom comes up, don't pad the report
with them. The fourth belongs in **Preview notes** on every report.

- "Random crashes on Release after upgrading" → check `RunAOTCompilation` + `AndroidLinkMode=None` first.
- A XAML build error names a file and an error code, never a cause. `MAUIG1001` in particular comes from inside the generator and says nothing useful — search the code, not the message.
- `pidof` doesn't prove which package is on screen when two variants are installed; `dumpsys window | grep mCurrentFocus` does. Worth passing on with the A/B snippet, since that is where two variants exist.
- Numbers above are from a **preview** SDK; the SdkOnly-on-Mono regression, XA1030 not firing and the MIBC tooling workaround may all change by GA. Tell the user to re-check anything preview-specific they depend on.

## When the user only wants the upgrade steps

Run steps 1–2 anyway — they are the cheap part and nothing downstream is right without them — and hand
back the steps that apply plus the two or three findings that would bite first. The `file:line` findings
are what makes the upgrade go smoothly and they stay in either case.

**This changes what you deliver, never how you deliver it.** If they asked for steps in the chat, answer
in the chat and write no file. **If you write a file, it is the report** — `coreclr-fit.md`, the full
skeleton, the card, the sign-off. There is no short version of the file: a `coreclr-fit.md` that is
missing the card or the sign-off is indistinguishable from one where the analysis quietly fell short,
and the reader has no way to tell which they are holding.

And the skill still changes nothing. Handing over the steps is not the same as running them.

### 5 · Write the report

One file, `coreclr-fit.md`, at the solution root.

**What you print in the conversation is the summary and then the card, in that order** — the same order
the report itself opens with, so the person who read it in the chat and the person who opens the file
later are reading the same document. The summary first, because it is the part that answers them without
a glossary; the card second, because it is what they will screenshot. Then one line saying where the
file is. Nothing else: the sections below are what the file is for.

It lands inside their repository, so when you hand it over you may add one line saying it is there and
that they may prefer not to commit it. Offer it, do not insist, and leave it out of the file itself — it
is a courtesy to the reader, not a section of the report.

**Copy the skeleton below and fill the slots.** Do not design a structure of your own, do not reorder it,
and do not decide section by section whether something is worth including. The parts written out in full
— the baseline paragraph, the severity legend, the version line, the sign-off — are already correct;
reproduce them, do not rewrite them. Two reports have to be readable side by side, and that only works
if they are the same document with different values in it.

````markdown
# CoreCLR fit: {app} ({their TFM} → .NET 11)

## Summary
{four short paragraphs, no technical vocabulary · rules below}

{score card — geometry below}

{headline: one sentence, the decision they came for}

## What breaks the build
{only if something does · structure below · drop the whole section if nothing breaks}

## What was analyzed
{csproj · TFM · the SDK that evaluated it and whether its major matches · file counts ·
 evaluated or read · every assumption taken when nobody answered step 3, under a heading
 that says they were taken automatically}

## Blockers
{only if a gate fired · drop the whole section if none did}

## Readiness by dimension
{one block per dimension, each deduction as `rule → value → file:line`, totals recomputed
 from the rows · state what passed too · name the rules that could not fire and their points}

## Upside by signal
{the four rows with the points earned and the count behind each · name the absent axes
 that apply to this repo}

## Reported, never scored

> `✖` breaks the build · `◐` blocks the gain · `○` runs as-is

{the ◐ and ○ findings with file:line and scope · the Full trimming note when there are
 reflection findings}

## Fix list
{tiers · every item carries all four requirements · state the projected total per tier}

## Projected gains

> Measured on .NET 11 preview 5 with both runtimes on the same code. Your baseline is
> **`{their TFM}`** with Mono, which we did not measure — we assume the Mono side behaves
> comparably for this kind of work. Treat these as approximations until you measure your own.

{the gains table, weighted by what the search found, groups kept, rows that do not apply dropped}

## What happens if you do nothing
{only when staying put is a real option · the LTS alternative, what they give up,
 what they keep · drop the section when .NET 11 is the only sensible destination}

## Upgrade steps
{only the snippets that apply to this repo, with their names in them}

## Preview notes
{what to re-check at GA · the LTS runway so nothing reads as urgent}

*Produced by {agent} running {model}, self-reported.*
{the mid-tier warning, if it applies}

> *These figures come from the `coreclr-fit v1` evidence set, measured on .NET 11 preview 5. They are
> re-measured when the runtime changes; the current set is always at skills.horus.com.uy/coreclr-fit.*

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
````

**The section set is closed.** Three sections are conditional — *What breaks the build* and *Blockers*
appear only when something fires, and *What happens if you do nothing* only when staying put is a real
option; every other section appears in every report. Nothing may be added, nothing else removed, nothing
reordered — the reader's decision is made in the first two sections, and a report that opens somewhere
else costs them that. A section with little to say gets one honest line, not deletion: *"No Xamarin-era
packages and no native bindings — nothing to report here"* is information. Silence is not.

**If the report does not end with the sign-off block, the report is not finished.** It is the last thing
in the file, after everything else, inside a fenced block, preceded by a `---` rule. Do not translate it,
do not re-space it, do not resize the mark, and never move it into the score card — the card is the part
people screenshot and it stays about their project, not about who wrote the tool. This is the only
decorative element in the report, and the contact line is an offer, not a call to action: in a document
that has just said their UI thread will not get faster and their APK will grow, a line that does not push
is the more credible one.


##### The score card

**Geometry, so two cards line up.** The box is **72 characters wide**, borders included: `┌`/`└` plus 70
`─`, and every content line is `│` + two spaces + 68 characters of content padded with spaces + `│`.
Count the padding; do not eyeball it. A card that does not align is the first thing a reader notices and
the last thing they trust.

**Bars come in two lengths, and mixing them breaks the width.** The two axis bars at the top are **20
characters**, filled = `round(score / 5)`. The three dimension bars below are **10 characters**, filled =
`round(10 × points / maximum)` — so `43/50` is 9 filled, `17/20` is 9, `24/30` is 8. `█` is filled, `░` is
empty, and the Upside bar uses its 0–100 points even though the card prints the band, so the two axis
bars stay comparable.

```
┌──────────────────────────────────────────────────────────────────────┐
│  CoreCLR fit · Fieldpath                                             │
│                                                                      │
│  Readiness   ██████████████░░░░░░  71/100  config changes only       │
│  Upside      ██████████████░░░░░░  HIGH    up to 44% less time on    │
│                                            list work                 │
│                                                                      │
│  → You can move once the binding form is fixed. The JSON path is     │
│    worth more and needs work first; the body has both steps.         │
│                                                                      │
│  Build & TFM            █████████░  43/50   net9.0                   │
│  XAML toolchain         █████░░░░░  10/20   MAUIX2002 · 9 of 24 bare │
│  Dependencies           ██████░░░░  18/30   2 Xamarin-era · Firebase │
│                                                                      │
│  ✖ Breaks the build: 1             After the fix list: 71 → 88       │
│  ── 6 of 42 candidates verified · SDK 9.0.304 · net9.0-android35.0   │
│     34 xaml (24 bind) · 212 cs · coreclr-fit v1                      │
│     skills.horus.com.uy/coreclr-fit                                  │
└──────────────────────────────────────────────────────────────────────┘
```
##### The summary · four paragraphs, no jargon

The report opens with it because the person who decides whether this work happens is usually not the
person who reads the rest. Four short paragraphs, in this order, **each derived from a section further
down and never from fresh analysis**:

1. **Can they do it** — the verdict, plus the one thing standing in the way if there is one.
2. **Is it worth it** — what gets faster and what does not, in the app's own terms: screens, lists,
   searches. Never a multiplier, never a TFM.
3. **What the work is** — the fix list's scope in items and files, and how much of it pays off even if
   they never upgrade.
4. **What happens if they wait** — the LTS date, so the decision reads as planning rather than an alarm.

**No technical vocabulary at all.** No TFM, no package name, no error code, no file extension, no
`file:line`, no severity mark. If a sentence needs one of those to make sense, it belongs further down.

**No percentage without its condition in the same sentence.** This is the block that gets pasted into an
email and quoted on its own, and a number that travels without its scope gets disproved by the first
person with a stopwatch.

**No durations**, the same rule the fix list follows. Items and files, never days.

**Nothing here that is not already below.** If the summary says it, a section further down proves it. A
summary that can say something the report cannot is how the two start to disagree.

**Every number on that card has to be reconstructible from the rules.** Each dimension line names the
deductions behind it and they have to add up: `43/50` is `net9` at −7 and nothing else; `10/20` is the
element-form binding at −7 plus partial `x:DataType` coverage at −3; `18/30` is two Xamarin-era packages
at −10 plus Firebase at −2. **Write the deductions first and let them produce the number** — never pick a
total and justify it afterwards, because a reader who knows the rules will check, and that is the whole
point of publishing them.

Note `34 xaml (24 bind)` in the footer and `9 of 24 bare` on the XAML line. They are different counts on
purpose: the coverage rule's denominator is the XAML that binds, not every `.xaml` in the repo.

The bottom lines matter more than they look. `6 of 42 candidates verified` states the coverage of the
analysis instead of implying it was exhaustive. The footer also versions the skill, so a score can be
traced to the rules that produced it.

**Never print either axis without its caption.** `HIGH` on its own says nothing about what improves; the
right-hand caption is what stops it being read as a blanket speed claim. Same for Readiness: the caption
says what kind of work is left.

On a re-run, read the previous `coreclr-fit.md` and lead with the movement instead of the absolute
number — it turns the report into a plan with a finish line:

```
  58 → 71  (+13)   since 2026-06-04       58 ▁▃▅ 71
  ▲ TFM moved up, 32-bit RIDs dropped       ▼ 3 Xamarin-era packages left
```

##### What each slot must contain

| # | Section | What it must contain |
|---|---|---|
| 0 | **Summary** | Four short paragraphs with no technical vocabulary, each one derived from a section below. Rules above |
| 1 | **Headline** | The technical half of what the summary just said in plain language, so **it must not repeat it**: not *what* the verdict is but *why* it holds — which parts of the app carry the gain, which do not, and the sentence that keeps someone from overselling it internally. When a gate fired, name the gate here |
| 2 | **What breaks the build** | Only if something does. Structure below |
| 3 | **What was analyzed** | csproj, TFM, the SDK that evaluated it and whether its major matches the TFM, file counts, whether values were **evaluated or read**, and every assumption taken when nobody answered step 3 |
| 4 | **Blockers** | Only if a gate fired |
| 5 | **Readiness by dimension** | A block per dimension where **every deduction names `file:line`** and the rows recompute to the total. State what passed too — it is why the number is high. If the project has no Android TFM, say which rules could not fire and how many points that was |
| 6 | **Upside by signal** | The four rows with the points this app earned and the count behind each one — *"LINQ: 54 call sites across 12 files → 34 of 45"*. The card shows a band and one caption; **this is the only place the reasoning behind it is visible**, and without it the band is a verdict with no argument. Name the absent axes here too when they apply |
| 7 | **Reported, never scored** | The `◐` and `○` findings with `file:line` and scope, plus the `Full` trimming note when reflection findings exist. The severity legend is printed here, in the skeleton |
| 8 | **Fix list** | The four requirements below, on every item |
| 9 | **Projected gains** | The baseline paragraph **verbatim, above the table**, then the table weighted by what the search found, with the "available without upgrading" rows kept separate |
| 9b | **What happens if you do nothing** | Only when it is a real option. Almost always it is: .NET 10 is LTS, so moving there instead gets them back into support, keeps Mono and costs a fraction of this fix list. Say what they give up — the runtime gains and, on Android, the MIBC profile — and what they keep, which is every `○` finding, because that work pays on whatever they ship. **This is the section that makes the rest credible.** A report that never names the cheaper option reads as a sales document, and the reader knows it |
| 10 | **Upgrade steps** | Only the snippets that apply to this repo, with their names in them |
| 11 | **Preview notes** | What to re-check at GA, the LTS runway so nothing reads as urgent, then the version line |

**Say that the numbers are versioned, never what is coming next.** A report that promises a measurement
is still in circulation if that measurement never happens, and the backlog is ours, not theirs. Where an
axis genuinely matters to *their* repo and we have no figure, that belongs in the body — the Upside
section names the absent axes that apply, with counts and without numbers.

##### The fix list · four requirements, on every item

A table of one-liners is not a fix list. It is the summary of one, and it leaves the reader with nothing
to do.

1. **A severity mark**, in the canonical wording — `✖ breaks the build` / `◐ blocks the gain` / `○ runs as-is`. Never reword it: two reports have to be readable side by side.
2. **The mechanism, in one or two sentences.** *Why* this costs them something, never "best practice". If you cannot say what goes wrong, the item does not belong on the list.
3. **A snippet using their names** — their packages, their types, their file. A generic snippet makes the reader do the translation, which is the step where fix lists die.
4. **A scope line counted from their repo** — *"28 files · every DTO on the API path"*. Never a duration.

Order by score gained ÷ scope, split into tiers, and **state the projected total explicitly** — *"Tier 1
takes Readiness from 81 to 98"*. If something is deliberately not on the list, say so and say why; an
unexplained ceiling below 100 reads as an oversight.

**Size every fix by scope, never by time.** Count what has to change and let the reader price it:
*"18 files · 34 call sites · 11 DTOs · 6 with custom converters"*. Tier headers sum scope the same way.
It is the skill's own rule turned on itself: **everything else in this report is backed and a duration is
not.** The subject of a gain projection is code, which this skill can read; the subject of a day estimate
is people, which it cannot see. If a size band is unavoidable, say **hours / days / weeks** and never a
decimal — `0.25d` claims two-hour accuracy on work nobody has scoped.

##### Voice

**Write to them, not about them.** The report is for the people whose repo this is, so address them:
*you*, *your app*, *your TFM*. Third person turns a report written for someone into a report written
about them, which reads as a document that reached them by accident.

What stays as it is:

- **First person for what we did.** *"We did not measure this"* · *"we assumed you don't need
  armeabi-v7a"*. It is what makes clear who is answerable for each claim: we assume, you decide.
- **Third person for actual third parties.** The trimmer, the OS, the compiler, QA, Microsoft. Those
  are not the reader.
- **Section headings**, which are fixed by the output contract and do not bend to voice.
- **`the user`, when it means the people who use *their* app.** This one inverts the meaning if you
  get it wrong: *"11 call sites on paths the user waits for"* becomes *"paths your users wait for"*,
  never *"paths you wait for"* — nobody on the team is waiting for that screen, their customers are.

Prefer active voice and contractions, and keep sentences short. *"It is assumed they do not need
armeabi-v7a"* has no owner; *"we assumed you don't need armeabi-v7a"* says who assumed and who is
affected, which is the whole relationship this report describes.

##### Language

**Write the report in the language the person wrote to you in.** Spanish in, Spanish out; Portuguese,
German, French, Japanese, the same. English is the fallback for when you genuinely cannot tell, which is
rare — one message is usually enough. Infer it, don't ask.

Two things override that, and neither is a guess: **they asked for another language** (*"this goes to our
US client"* is such a request even if it does not sound like one), or **they want the chat in one
language and the file in another**. A repo written in English is a hint worth one line — *"the repo is in
English, want this in English too?"* — never a silent override. State the choice in one line when you
hand the report over, because this file gets committed and forwarded, and the person running the skill is
often not its last reader.

**What never translates:** identifiers and code (property names, type names, MSBuild properties, file
paths, `file:line`, error codes like `MAUIX2002`, every snippet) and **the sign-off block**.

**Everything else translates, including the vocabulary.** A reader who has to decode `Upside` before
reading their own report is one the report lost.

- `Readiness` → *Preparación* · `Upside` → *Ganancia*. **Gloss the English once** on first use — *"Preparación (Readiness)"* — then use the translation.
- `LOW` / `MEDIUM` / `HIGH` → *BAJA* / *MEDIA* / *ALTA*.
- `✖ breaks the build` / `◐ blocks the gain` / `○ runs as-is` — translate the words, **never the symbols**. `✖ ◐ ○` are what make the list scan.
- `measured` / `inferred` / `not measured` / `observed, cause not isolated` — translate, and keep the four distinct. The distinction is the point, not the English.

Avoid decorating beyond the severity marks and the card. Heavy box-drawing and emoji headings break in
narrow terminals and read as noise in a report someone forwards to a client.

##### Say what produced this report

**Add one line to Preview notes naming the agent and model that produced it**, as self-reported:

> *Produced by {agent} running {model}, self-reported.*

A report that comes back thin is otherwise indistinguishable from a repo that had little to report, and
six months later nobody can tell which one they are holding. This line costs nothing and settles it.

**If you are not the most capable model your provider offers, say so, right under that line:**

> ⚠ *Produced on a mid-tier model. The analysis is real, but this skill asks for a long chain of work on
> a whole repository — evaluating the build, verifying every candidate before scoring it, and
> reproducing a fixed report structure. A stronger model verifies more candidates and is less likely to
> leave a section thin or a deduction estimated. If anything here reads shallow, re-run it on the
> strongest model available to you and compare.*

Judge that honestly rather than generously. The question is not whether you can do the task — you can —
but whether this particular task, which is long and has an exact output contract, is one where the gap
shows. It is.

**And say it in the report when the run itself was degraded, whatever model you are.** This is the part
that does not depend on knowing your own tier, and it catches the same damage:

- Step 3 went unanswered and defaults were assumed → already required under *What was analyzed*.
- A search could not run, or a rule could not be checked → name the rule and say it did not fire.
- Candidates were scored without opening the file → say how many, in the card footer, where
  `6 of 42 candidates verified` already lives.
- The build could not be evaluated and values were read from the csproj instead → already required.

Two or more of those in one run, and the headline says so in a sentence before anything else: *"this run
was partial, and here is what was missing"*. A reader who knows the analysis was thin can act on it. A
reader who does not know will act on it anyway, which is the failure this whole section exists to stop.

#### "Breaks the build" — structure

This is the section that makes the report circulate, so it earns real estate. These are the findings that
stop the upgrade on day one — **at compile time, not in the field**. That distinction matters and the
report should make it: the compiler names them, you fix them, nobody ships them. They cost a morning, not
a postmortem.

Lead with the count and the shared cause in one short paragraph, then one block per finding, ordered by
how much of the app each one blocks. Show the five widest in full; if there are more, list the rest in a
compact table (file · `file:line` · error code) and say how many there are.

**The `Blocks` row is where that breadth gets counted, so put a number in it** — *"the whole build"* when
nothing compiles until it is fixed, *"this view is used on 4 pages"* when the damage is local. It is
what justifies the order, and it is the same discipline as the fix list's scope line: counted from
their repo, never a duration.

**A `✖` is earned by naming the error, and the `Error` row is the one that cannot be skipped.** If you
cannot say which code the compiler emits, you do not know that the build fails — you are predicting it,
and the mark that means *stop everything on Monday* is not the place to guess. Score it `◐` instead, say
plainly that it is expected to break and has not been confirmed, and it costs the same points either way.
A runtime crash is the same: it belongs here only with a reproduction attached. The day someone compiles
it and brings back the code, the finding earns its `✖` and the rules below take over on their own.

Per block, the fields that actually differ; skip a row when it adds nothing:

```
### 1 · Order list · element-form RelativeSource
`Pages/Orders/OrderListPage.xaml:42`

| | |
|---|---|
| Form | `<Binding RelativeSource="{RelativeSource AncestorType=...}" Path="Total"/>` |
| Error | MAUIX2002 — not a bindable property in element syntax |
| Blocks | the whole build; nothing downstream compiles until it is fixed |
| Fix | `{Binding Total, Source={RelativeSource AncestorType=...}}` — same semantics |
```

Close the section with one line stating that nothing else in the report stops the build — everything else
either runs as-is or costs upside. Readers need to know where the cliff ends.

**If there are no build breaks, drop the section entirely.** Do not pad it with warnings; an empty
`✖` is good news and the card already says `Breaks the build: none`.

> **`✖` means the build fails** — not that a screen crashes in the field. A runtime break belongs here
> only with a reproduction attached.

**One number, three places.** The count on the card is the number of `✖` items in the fix list, and each
of those has its block in this section. If the card says `none`, there is no `✖` anywhere in the report.
The three are written at different moments and nothing reconciles them afterwards, which is exactly how
a report ends up saying *nothing breaks* at the top and *this breaks* further down — and the reader who
spots it stops trusting the rest of the file, which is the correct response.

#### Projected gains — the evidence, and how to present it

Setup: .NET 11 preview 5, Release arm64, two physical phones — a Xiaomi Mi 9T Pro and a moto g54 5G
(Android 15). Mono = AOT + `Full` trimming; CoreCLR = R2R + `SdkOnly`. Noise floor ≈ 5%.

**Two series, and the rows say which.** The published figures below come from the Xiaomi: one cold run
plus five warm, averaged. The ceilings they point at — `5.0×` steady state, `9.9×` on JSON — come from a
longer session on the moto, fifteen runs per launch with the ramp discarded and medians reported. **Mono
lands in the same place on both**, which is what says the distance between the two series is the tier-1
ramp and not the hardware.

**State the gap between this baseline and theirs, every time. It is not the same comparison.** Both
sides of these measurements ran on **.NET 11 preview 5** — the last build where both runtimes existed,
which is what made isolating the runtime possible at all. Their baseline is the TFM they are on today
with Mono, a different SDK and a different MAUI. That difference was not measured.

Put this in the report above the table, **with their TFM in it, not ours**:

> Measured on .NET 11 preview 5 with both runtimes on the same code. Your baseline is
> **`<their TFM>`** with Mono, which we did not measure — we assume the Mono side behaves comparably
> for this kind of work. Treat these as approximations until you measure your own.

A project two or three versions back is a step further from the measured baseline than one on .NET 10,
and the report should say so in the same breath rather than leaving the reader to work it out: *"yours
is .NET 8, two versions further back again"*. Overstating how close the comparison is costs more than
the sentence it saves.

The assumption is strongest exactly where the headline lives, which is what makes it defensible rather
than a disclaimer:

| Rows | Does it carry from .NET 11 Mono back to an older Mono? |
|---|---|
| CPU-bound, JSON, LINQ, dates | **Yes, well.** The BCL does not change much per version for this kind of work, though the gap widens the further back they are |
| Raw cold start | More SDK-sensitive — but that row promises almost nothing, so little is riding on it |
| MIBC profile, app code | **Does not apply.** CoreCLR-only measurements with no Mono side |

Say the direction of travel too, because it is the honest version of "we will know more later": once
their own baseline is measured against `net11` + CoreCLR, nothing is assumed — that measures the
transition they are actually making. These rows explain *where* the gain comes from and stay true
either way.

**Below 2×, quote the percentage of time saved; above 2×, quote the multiplier.** Both come from the
same pair — `38 → 21 ms` is `1.8×` and `−44%` — but they do not read the same. Someone who sees `1.8×`
hears "70% better", which is nearly double the truth: the time drops *to* `18/30`, it does not drop *by*
`0.7` of it. Small multipliers invite that mistake and percentages do not, while above 2× the multiplier
is the clearer of the two. Captions are the exception and always take the percentage, even above 2×,
because they travel without the table. **Never write "1.8× faster", and never convert a multiplier
yourself** —
both forms are in the table below, already derived from the measurement.

**Quote cold start as a percentage, never in milliseconds.** The absolute on the reference app says far
more about how that app was built — its home inflated 24 eager cards with real controls, more than half
of the start — than about either runtime, and it transfers to nobody else's project. The delta does.
This applies to the cold-start rows only: the workload rows keep their figures, because *2 → 1 ms on the
UI thread* is the finding.

**Four groups, and the order answers the reader's questions in the order they ask them:** what arrives
for free, where it does not, what costs me work, and what I already have without this upgrade. Drop any
group that is empty, and keep the grouping — a flat list makes the reader sort fifteen rows into those
four buckets themselves, which is the work this table exists to do for them.

##### What the runtime gives you — nothing to do, it comes with the upgrade

The same data load, measured at three points of the same curve. Which one applies depends on how often
the user opens that screen, which is what the session-length row of Upside scores.

| Source | Measured | Applies when | Confidence |
|---|---|---|---|
| **Runtime** (Mono → CoreCLR, same code) — a full data load, **first time in the process** | 363 → 272 ms, **1.33×** · **−25%** time | Screens opened once per process. On mobile, where the OS kills background processes, this is what most users feel. | measured |
| Runtime — **the same load a few more times** (average of 5) | 252 → 166 ms, **1.52×** · **−34%** time | **The honest everyday figure, and the one we published.** Someone who opens the screen a handful of times and closes the app. It includes the tier-1 ramp and a GC spike, because real use does. | measured |
| Runtime — warm steady state | 238 → 90 ms, **2.6×** · **−62%** time | Once tier-1 has promoted. Over a longer session it keeps improving: a 15-run session reached **5.0×** on a second device. Quote 2.6× and name 5.0× as the ceiling with that condition attached — tier-1 state dies with the process, so most apps never get there. | measured |

And the phases inside it, so the report can cite the one this repo actually earns:

| Source | Measured | Applies when | Confidence |
|---|---|---|---|
| Runtime — steady state, **JSON deserialization** (STJ source-gen) | 176 → 60 ms, **2.9×** · **−66%** time | The heaviest phase: ~70% of the load on both runtimes. The same ceiling applies — the longer session reached **9.9×** on this phase. No figure exists for reflection serializers. | measured |
| Runtime — steady state, **LINQ** (`Select` + `OrderByDescending` + `ToList`) | 38 → 21 ms, **1.8×** · **−44%** time | View models shaping collections on a path the user waits for. Cite this one, not the JSON row, when the repo sorts and filters more than it parses. The run was noisy on the Mono side; the per-run detail is in the evidence. | measured |
| Runtime — steady state, **date parse + format** | 17 → 8 ms, **2.1×** · **−53%** time | Parsing or formatting dates in bulk, typically while building a list. | measured |

##### Where it does not win, or loses

**Lead with this group when the repo's screens are light.** It is the half of the evidence nobody else
publishes, and it is what makes the other half believable.

| Source | Measured | Applies when | Confidence |
|---|---|---|---|
| Runtime — **first load, small phases** | dates 25 → **61 ms** · bind 8 → **14 ms** | **CoreCLR loses here, and the report should say so.** On the very first load it is still running unoptimised code and paying JIT warm-up; the big phases carry enough work to win anyway, the small ones do not. An app whose screens do little work and open once may gain nothing, or lose a little. | measured |
| Runtime — UI-thread work | 2 → 1 ms, **≈ 1×** | Layout, measure, handlers: no gain to promise. | measured |
| Runtime — raw cold start | **−4%** | Small, and at the noise floor on any single run — but CoreCLR won every run across two phones, so the direction is solid even though the size is not worth promising. CoreCLR alone does not meaningfully start faster. | measured |
| Runtime — GC | spikes proportionally larger (3.6× median vs 1.6×), smaller in absolute terms | Report medians and spike counts separately. | measured |
| Runtime — APK size | ~29 → ~30 MB arm64 | Slight regression; Microsoft reports larger ones on big apps. | measured |

##### What build config gives you — their work, Android only, measurable in an afternoon

| Source | Measured | Applies when | Confidence |
|---|---|---|---|
| **Build config** — custom startup MIBC profile for R2R | **−21% cold start** vs CoreCLR without it | The largest build-config lever by far. Android only; ~15 min to regenerate when the startup path changes; official `maui profile` tooling broken in preview 5, workaround exists. | measured, needs work |
| **Runtime + build config, end to end** — Mono+AOT → CoreCLR R2R with a custom MIBC profile | **−25% cold start** | The comparison a client asks for first: *against what I ship today, what does the upgrade plus the profile work give me?* **Not the sum of the two rows above** — a direct measurement of one end against the other, from the same session. Android only. | measured |
| `MauiEnableFullReadyToRun` | **not measured** | Release does partial ReadyToRun by default; the full pass is opt-in. Trades APK size for startup, in that direction. Offer it as a lever to try with the A/B, never as a figure. | not measured |

##### What their own code gives you — no upgrade required

| Source | Measured | Applies when | Confidence |
|---|---|---|---|
| **App code** — lazy first page, Debug-only logging, lazy flyout | **−52%** on Mono · **−53%** on CoreCLR | Lands on whatever they ship today, no upgrade involved. The largest single win in the whole investigation, and it pays almost identically on both runtimes — so the CoreCLR advantage is not Mono suffering more from a slow startup path. | measured |
| **And the runtime still pays on top of it** — both sides optimised | 1,850 → 1,380 ms, **−25%** | **The answer to the obvious objection.** A reader who sees the row above will ask why they should upgrade at all instead of just fixing their startup path. Because after that work, on both runtimes, CoreCLR is still a quarter faster. Measured with the same optimisations applied to both. | measured |

Say the app-code row anyway, and say it early when the startup findings are large — the honest version
of this report sometimes reads *"your biggest available win does not require this upgrade at all"*. The
report lists exactly these problems as `○` findings, so the reader is owed the figure. They are simply
not owed it inside the upgrade's payoff, where it would be summed. **The row below it is what keeps that
honesty from reading as an argument against the upgrade.**

##### Null results

| Source | Measured | Applies when | Confidence |
|---|---|---|---|
| SkSL shader rendering (GPU) | **no difference** between runtimes | Measured, null result. The managed side is three lines per frame and vsync hides whatever margin exists — while both reach 60fps, both read 60. Do not promise graphics gains, and do not accept FPS as evidence either way. | measured |
| Scroll FPS, navigation timing | — | **No data.** Say "not measured". | — |


**The rule that predicts where the gain is.** The per-phase spread is not noise, it tracks one property:
JSON 2.9× (interfaces, generics, virtual converters, spans) · dates 2.1× · LINQ 1.8× (delegates,
iterators) · UI bind ≈1× (framework interop) · shaders, no difference (the work is on the GPU). **The
same ordering came out of a second session on a different phone**, where the spread was wider still
(JSON `9.9×`, LINQ `1.7×`) — two independent runs agreeing on the order is what makes this a law rather
than one dataset's shape. **The more layers of abstraction the code has, the larger CoreCLR's
advantage**: that is tier-1 with dynamic PGO inlining and devirtualising, decisions that depend on
observing at runtime which implementation actually arrives, which Mono's AOT cannot make at compile time.
The corollary is worth saying out loud to a team that expects a speed-up everywhere: **a tight float-math
loop will likely give 1.5× and little more.** Moving work to the CPU is not enough; the work has to be
*abstract*.

Weight each line by what the search found, and **cite the phase rows rather than the aggregate when the
repo tells you which one applies** — the JSON row for an app whose screens deserialize, the LINQ row for
one that mostly sorts and filters, the UI row for one that is layout and handlers. Take each figure from
the table in the form the rule above requires: **a percentage below 2×, the multiplier above it**. Quoting
the 2.6× aggregate to a project that does no JSON is the most common way to overpromise with true
numbers — almost the whole gap is that one phase. Write only the caveats that apply, and close with:
"These are projections from a reference benchmark, not measurements of your app — the upgrade steps
include the side-by-side TFM setup so you can measure before committing."

##### Before you hand it over

Check these against the file you just wrote, not against your intention to write it. Each one is a thing
that has actually gone missing from a real run.

```
[ ] Summary present, four paragraphs, nothing in it that is not proved below
[ ] Summary carries no TFM, package name, error code, file extension or duration
[ ] Score card present, 72 characters wide, every line padded, both bars 20 chars
[ ] Both axes carry their caption
[ ] Every section of the skeleton is there, in order, none added
[ ] The three conditional sections are present or genuinely did not apply
[ ] Severity legend printed above the first section that uses a mark
[ ] Every deduction shows its rule, its value and its file:line
[ ] Each dimension recomputed from the rows printed under it — if they disagree, the rows are right
[ ] No deduction value invented: every one appears in a table in step 4
[ ] Every fix list item carries all four requirements
[ ] Baseline paragraph verbatim above the gains table, with their TFM in it
[ ] Version line at the close of Preview notes
[ ] Preview notes names the agent and model, and carries the mid-tier warning if it applies
[ ] Anything that made this run partial is stated — unanswered questions, rules that could not
    be checked, candidates scored without opening the file
[ ] The card's build-break count equals the number of ✖ items, each with its block
[ ] Every ✖ names the compiler error it produces, or it is not a ✖
[ ] The last thing in the file is the sign-off block, verbatim
```

If one of these fails, fix it before handing the file over. A report that is missing the card or the
sign-off is not a shorter version of this report — it is a different document, and the person reading it
has no way to tell what else was left out.
