# coreclr-fit · PDF export

Read this only when someone has the report and asks for a PDF. It is not part of the analysis, and
nothing here changes what the report says.

**The PDF is the same report, typeset.** Same sections, same numbers, same wording. You are not
rewriting, shortening or re-ordering anything. If you find yourself editing the content while you
build the page, stop: the markdown file is the source and it has already been handed over.

There is one output, the whole report. The summary alone is four paragraphs and does not make a
document.

## 1 · Build the HTML

Write a single self-contained `.html` file next to the report. Everything inline, no external
requests: the person is going to open this on a plane or forward it to someone behind a firewall.

```html
<!doctype html><html lang="{the report's language}"><meta charset="utf-8">
<title>CoreCLR fit · {app}</title>
<style>{the whole stylesheet from § 4, inline}</style>
<body>
  <div class="head">
    <p class="eyebrow">CoreCLR fit</p>
    <h1>{app}<span class="tfm">{their TFMs} &nbsp;→&nbsp; .NET 11</span></h1>
  </div>
  <div class="summary">{the four paragraphs of § Summary, each in its own <p>}</div>
  {the card, § 2}
  {the rest of the report, from the headline down, as HTML}
  <div class="foot">
    <img class="logo" alt="Horus Studio" src="data:image/png;base64,{the logo}">
    <div class="txt">
      <p class="lead">CoreCLR fit, numbers from an upgrade we ran ourselves on .NET 11,
      measured on two phones.</p>
      <p><a href="https://skills.horus.com.uy/coreclr-fit">skills.horus.com.uy/coreclr-fit</a></p>
      <p>Need a hand with this? <a href="https://horus.com.uy/contact">horus.com.uy/contact</a></p>
    </div>
  </div>
</body></html>
```

**Three things come out of the body and do not get repeated.** The `# CoreCLR fit: …` title, because
the header carries it. The `## Summary` heading and its paragraphs, because they are already in
`.summary`. And the ASCII sign-off block at the end, because the footer replaces it — a logo drawn in
characters makes sense in a terminal and nowhere else.

**The logo lives next to this file**, as `logo.png`, if the skill came in as a plugin. If it is not
there, fetch it from
`https://raw.githubusercontent.com/HorusSoftwareUY/agent-skills/main/skills/coreclr-fit/logo.png`.
Either way, inline it as base64 — and that is not only so the file works offline. A remote `<img>` means
our server learns every time someone opens this document, and from where. A report about a client's code
has no business telling us when they read it. If you cannot get the logo at all, drop the `<img>` and
keep the footer text: a missing logo is better than a broken one.

**Nothing else in the page reaches the network either.** No fonts, no stylesheets, no analytics. The
HTML and the PDF stay on their machine.

## 2 · The card becomes real bars

The ASCII card does not survive being typeset — it is a monospace drawing that exists because a
terminal has nothing else. Rebuild it as markup, reading the values off the card you already wrote:

```html
<div class="card">
  <div class="axis"><span class="lbl">Readiness</span>
    <span class="bar"><i style="width:73%"></i></span>
    <span class="val">73/100</span><span class="cap">upgrade on your schedule</span></div>
  <div class="axis"><span class="lbl">Upside</span>
    <span class="bar"><i style="width:34%"></i></span>
    <span class="val">LOW</span><span class="cap">up to 44% less time on catalog lists</span></div>

  <div class="verdict">{the → line from the card, as one sentence}</div>

  <div class="dims">
    <div class="axis"><span class="lbl">Build &amp; TFM</span>
      <span class="bar"><i style="width:72%"></i></span>
      <span class="val">36/50</span><span class="cap">net9.0 · 32-bit RIDs</span></div>
    {one per dimension}
  </div>

  <div class="meta"><b>Breaks the build: {n}</b> &nbsp;·&nbsp; After the fix list: {a} → {b}<br>
  {the provenance line: candidates verified · SDK · TFM · file counts}</div>
</div>
```

**The bar width is a percentage of the maximum, not of 100.** Readiness and Upside use the score
itself; a dimension uses `points / maximum × 100`, so `36/50` is `72%` and `20/30` is `67%`. Getting
this wrong is the one error a reader spots instantly, because the bar and the number sit side by side.

**Both axis bars are the same colour.** Red on a `LOW` bar reads as an error when it is a finding.

## 3 · Blockquotes become callouts

Every `>` block in the report is a note the reader should not skim past, and in a wall of justified
text they disappear. Two kinds:

- **`IMPORTANT`**, red, for anything describing a way things fail. The `Full` trimming note is always
  this one.
- **`NOTE`**, grey, for everything else, including the baseline paragraph above the gains table.

```html
<div class="callout important"><span class="kicker">Important</span>{the block's content}</div>
```

The severity legend line gets `class="legend"` rather than being a paragraph like any other.

## 4 · The stylesheet

Paste it whole. It is tuned for A4 and for the shapes this report actually contains; editing it to
taste is how two reports stop looking like the same document.

```css
:root{
  --ink:#111; --muted:#6b6b6b; --line:#e4e4e4; --accent:#ff3131;
  --bar:#1a1a1a; --bar-bg:#ececec;
}
@page{ size:A4; margin:19mm 17mm 20mm; }
*{ box-sizing:border-box; }
body{
  font:10.5pt/1.62 -apple-system,"Segoe UI",Inter,Helvetica,Arial,
       "Apple Symbols","Segoe UI Symbol",sans-serif;
  color:var(--ink); margin:0; -webkit-print-color-adjust:exact; print-color-adjust:exact;
}
h1{ font-size:21pt; line-height:1.15; margin:0 0 2mm; letter-spacing:-.01em; }
h1 .tfm{ display:block; font-size:10pt; font-weight:400; color:var(--muted);
         margin-top:2mm; letter-spacing:0; }
h2{ font-size:13pt; margin:10mm 0 4mm; padding-bottom:1.5mm;
    border-bottom:1px solid var(--line); letter-spacing:-.005em; break-after:avoid; }
h3{ font-size:11pt; margin:6mm 0 2mm; break-after:avoid; }
p{ margin:0 0 3.5mm; }
strong{ font-weight:650; }
a{ color:var(--ink); }
code{ font:9.5pt/1.4 ui-monospace,"SF Mono",Menlo,Consolas,monospace;
      background:#f5f5f5; padding:.5mm 1.2mm; border-radius:2px; }
pre{ background:#fafafa; border:1px solid var(--line); border-left:2px solid var(--accent);
     padding:3mm 4mm; overflow:hidden; break-inside:avoid; }
pre code{ background:none; padding:0; font-size:8.5pt; line-height:1.45; }

table{ border-collapse:collapse; width:100%; margin:0 0 5mm; font-size:9.5pt;
       line-height:1.6; break-inside:avoid; }
thead th{ text-align:left; background:#f4f4f4; font-weight:650; color:var(--ink);
          font-size:8.5pt; text-transform:uppercase; letter-spacing:.06em;
          padding:2.2mm 3mm; border-bottom:1px solid var(--line); }
td{ text-align:left; padding:2.4mm 3mm; border-bottom:1px solid var(--line);
    vertical-align:top; }

.callout{ margin:0 0 5mm; padding:3.5mm 4.5mm; background:#f7f7f7;
          border-left:3px solid #8a8a8a; border-radius:0 3px 3px 0;
          font-size:9.5pt; line-height:1.6; break-inside:avoid; }
.callout .kicker{ display:block; font-weight:650; font-size:8pt; letter-spacing:.09em;
                  text-transform:uppercase; margin-bottom:1.5mm; color:#8a8a8a; }
.callout p:last-child{ margin:0; }
.callout.important{ background:#fff5f5; border-left-color:var(--accent); }
.callout.important .kicker{ color:var(--accent); }
.legend{ font-size:9pt; color:var(--muted); margin:0 0 5mm; }

ul,ol{ margin:0 0 3mm; padding-left:5mm; }
li{ margin-bottom:1mm; }
hr{ border:0; border-top:1px solid var(--line); margin:7mm 0; }

.head{ border-bottom:2px solid var(--ink); padding-bottom:4mm; margin-bottom:6mm; }
.eyebrow{ font-size:8pt; letter-spacing:.18em; text-transform:uppercase; color:var(--accent);
          font-weight:650; margin:0 0 2.5mm; }

.summary{ font-size:12pt; line-height:1.6; margin-bottom:7mm; }
.summary p{ margin:0 0 3.5mm; }
.summary p:first-child{ font-weight:600; }

.card{ border:1px solid var(--line); border-top:3px solid var(--accent); padding:5mm 5mm 4mm;
       margin:0 0 7mm; break-inside:avoid; }
.axis{ display:grid; grid-template-columns:22mm 42mm 16mm 1fr; align-items:center;
       gap:0 3mm; margin-bottom:2.5mm; }
.axis .lbl{ font-weight:650; font-size:10pt; }
.axis .val{ font-weight:650; font-size:10pt; font-variant-numeric:tabular-nums; }
.axis .cap{ font-size:9pt; color:var(--muted); line-height:1.35; }
.bar{ height:7px; background:var(--bar-bg); border-radius:1px; overflow:hidden; }
.bar > i{ display:block; height:100%; background:var(--bar); }
.verdict{ margin:4mm 0; padding:3mm 0 0; border-top:1px solid var(--line);
          font-size:10.5pt; font-weight:600; }
.dims{ margin-top:4mm; }
.dims .axis{ grid-template-columns:34mm 26mm 14mm 1fr; margin-bottom:2mm; }
.dims .lbl,.dims .val{ font-weight:500; font-size:9.5pt; }
.dims .bar{ height:5px; }
.meta{ margin-top:4mm; padding-top:3mm; border-top:1px solid var(--line);
       font-size:8.5pt; color:var(--muted); line-height:1.5; }
.meta b{ color:var(--ink); font-weight:650; }

.foot{ margin-top:11mm; padding-top:4mm; border-top:1px solid var(--line);
       display:flex; align-items:center; gap:7mm; break-inside:avoid; }
.foot .logo{ width:32mm; height:auto; flex:none; }
.foot .txt{ font-size:8.5pt; line-height:1.65; color:var(--muted); }
.foot .txt p{ margin:0; }
.foot .txt .lead{ color:var(--ink); }
.foot .txt a{ color:var(--muted); text-decoration:none; }
```

## 5 · Print it

**Say what you are about to do before you do it.** This skill promised to read and change nothing, and
its commands are read-only by construction. Printing a PDF is not: you write an HTML file and you launch
a browser. It is their own browser, headless, on a local file, and it installs nothing — but it is still
a program starting, and they should hear it from you rather than from a permission prompt. One sentence,
and let them say no.

Any Chromium browser prints a PDF from the command line, and one of these is on almost every machine.
Try them in order and use the first that exists:

```
macOS      /Applications/Google Chrome.app/Contents/MacOS/Google Chrome
           /Applications/Microsoft Edge.app/Contents/MacOS/Microsoft Edge
           /Applications/Brave Browser.app/Contents/MacOS/Brave Browser

Windows    C:\Program Files\Google\Chrome\Application\chrome.exe
           C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe

Linux      google-chrome · chromium · microsoft-edge   (on PATH)
```

```
"<browser>" --headless --disable-gpu --no-pdf-header-footer \
  --print-to-pdf="<output>" "file://<absolute path to the html>"
```

`--no-pdf-header-footer` matters: without it every page gets the file path and a date stamp in the
margins, which is exactly what you do not want on a document someone forwards.

**Where it goes.** The user's Downloads folder, `~/Downloads` or `%USERPROFILE%\Downloads`, named
`coreclr-fit-{app}.pdf`. Not the solution root: the report is already in their repo and a second file
there is one more thing to gitignore.

**If no browser is there, say so and stop.** Leave the `.html` where you built it, tell them the path,
and tell them to open it and print to PDF from the browser — the stylesheet already carries the print
rules, so the result is the same. Do not install anything, do not reach for pandoc or LaTeX, and do not
quietly produce something worse. A person who asked for a PDF and got a silent half-answer has to
check your work, which costs them more than the PDF was worth.

## 6 · Before you hand it over

```
[ ] The PDF opens, and page 1 is the summary and the card
[ ] Bar widths match their numbers: 36/50 is 72% of the bar, not 36%
[ ] No page carries a file path or a date stamp in the margin
[ ] The ASCII card and the ASCII sign-off are gone, replaced by the card markup and the footer
[ ] Nothing in the PDF says anything the markdown report does not
```
