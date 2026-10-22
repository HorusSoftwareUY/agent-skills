# Horus agent skills

Agent skills we built for our own work and publish instead of charging for them.
Each one is a single Markdown file with no scripts, so you can read it before you trust it.

Currently focused on .NET MAUI and mobile.

| Skill | What it does |
|---|---|
| [`coreclr-fit`](skills/coreclr-fit/SKILL.md) | Scores whether .NET 11 with the CoreCLR runtime is a fit for a .NET MAUI project |

---

## coreclr-fit

From .NET 11, MAUI apps run on CoreCLR and Mono is gone. That is not a version bump: Release builds
compile differently, two XAML forms stop compiling, 32-bit Android is gone, and trimming mistakes fail
at runtime instead of never.

This skill turns a repo into two scores, a fix list with `file:line` references, and a projection the
team can defend in a meeting.

**Two axes, because they are different decisions.** *Readiness* scores how much work the upgrade takes.
*Upside* scores how much of the measured gain this particular app can actually capture. An app can be
ready to upgrade and have nothing to gain, and the report says so when that is the answer.

### Use it without installing anything

Point any agent that can read a URL at it:

```
Read https://skills.horus.com.uy/coreclr-fit and score this project
```

Works in Claude Code, Cursor, Codex and GitHub Copilot.

### Or install it

```bash
mkdir -p .claude/skills/coreclr-fit && \
  curl -fsSL https://skills.horus.com.uy/coreclr-fit -o .claude/skills/coreclr-fit/SKILL.md
```

Or as a plugin marketplace:

```bash
claude plugin marketplace add HorusSoftwareUY/agent-skills
```

### What you get

A `coreclr-fit.md` written at the solution root, and a card printed in the conversation:

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
└──────────────────────────────────────────────────────────────────────┘
```

A full sample report is in [`examples/verado.md`](examples/verado.md). The project is fictional; the
measurements behind it are not.

### Where the numbers come from

22 measurements on two physical Android phones, from a real upgrade of a MAUI library and sample app to
.NET 11 preview 5, with both runtimes built from the same code. Cold start, a 10,000-record load broken
down into JSON, LINQ and dates, GC behaviour, APK size.

**Including the three where CoreCLR does not win.** The UI thread measures about the same on both, the
APK grows slightly, and shader rendering shows no difference at all. The skill reports those as findings,
because a report that only contains good news is not a report.

Every figure carries a confidence label, and the ones we did not measure say so. The skill never sums
gains from independent axes into a single number.

### It does not change your project

It reads the repo and writes one file. No source file, no `.csproj`, no package reference, no TFM. The
commands it runs are read-only: `dotnet msbuild -getProperty` evaluates without compiling. If you read
the report and then ask it to apply the fixes, that is a separate step you ask for.

---

## Licence

MIT. See [LICENSE](LICENSE).

Built by [Horus Studio](https://horus.com.uy), a .NET MAUI consultancy in Uruguay.
