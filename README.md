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

### Use it once, nothing to install

Paste this into any coding agent that has a shell:

```
Download https://skills.horus.com.uy/coreclr-fit with curl, not with your web fetch
tool, because I need the raw file and not a summary of it. Save it as
coreclr-fit-skill.md, read the whole thing, then follow it to score this project.
```

**The "with curl" part is not decoration.** Some agents fetch a page and hand the model a summary of it
rather than the page. This skill does not survive that: the report contains blocks that have to be
reproduced exactly, a 72-column score card, a severity legend, a baseline paragraph and a sign-off, and a
summary keeps the rules and drops all of them. The scores come out right and the report is a different
document. Going through a file on disk removes the question.

### Or install it

```bash
mkdir -p .claude/skills/coreclr-fit && \
  curl -fsSL https://skills.horus.com.uy/coreclr-fit -o .claude/skills/coreclr-fit/SKILL.md
```

Or as a plugin marketplace:

```bash
claude plugin marketplace add HorusSoftwareUY/agent-skills
```

Same file either way. Installing just means the team has it and it triggers on its own when someone asks
whether the upgrade is worth it.

Works in Claude Code, Cursor, Codex and GitHub Copilot.

### Which model to run it on

Run it on the most capable model you have access to. This skill asks for a long chain of work on a whole
repository: evaluating the build, searching 27 patterns, opening each candidate before scoring it, and
then reproducing a fixed report structure. Smaller and faster models do the analysis, but they are more
likely to verify fewer candidates or leave a section thin.

The report says which model produced it, and flags when the run was partial, so you can tell the
difference between a thin report and a repo with little to report. If something reads shallow, re-run it
on a stronger model and compare the two.

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
