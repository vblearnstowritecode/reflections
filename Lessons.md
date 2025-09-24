# Lessons in Simple But Useful Tech

## Episode 1: Taming HTML→PDF for a Resume on Windows - what I tried, what I learned, and what finally worked!

Before I begin: You should know that I am a recovering product manager, a born-again developer, and this post is an attempt to chronicle my journey. You should also know, since it is funny, this resume was for my wife. "30 mins max" - was what I told her. And 3.5 hours later, there I was dropping her an email saying: "I made an art out of this!".

## Why I went down this rabbit hole

Thanks to Claude Artifacts, I had a nicely styled HTML resume that looked great in the browser. I found word -> PDF to be very limiting and not aesthetically pleasing. The trouble. however, started when I tried to turn the glorious HTML into a clean, shareable PDF. Different tools gave different results:

* **Chrome Print** looked one way, **an online HTML→PDF site** looked another, and **local “Save as PDF”** sometimes added a blank page or ate my thin divider lines.
* On Windows, **OneDrive paths** and default header/footer settings also changed margins and pagination.

This post is a field report: symptoms, experiments, the mental model that made things click, and the exact commands/CSS that produced a dependable PDF. 

---

## TL;DR: the working recipe

* Use **Edge headless** (Chromium) to export the HTML with print CSS honored and **no headers/footers**.
* Keep print CSS **simple**: `@page { size: A4; margin: 12mm; }`, avoid sub‑pixel borders, and don’t overuse `break-inside: avoid`.
* For hard page splits, prefer **`break-after: page` on the previous section** (more reliable) rather than `break-before` on the next.
* Use an actual `<hr>` for rules, and a **table** for rigid rows (like Education) to get stable alignment.

**One-liner (PowerShell):**

```powershell
$edge = "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
& $edge `
  --headless=new --disable-gpu `
  --log-level=3 --disable-logging `
  --print-to-pdf="C:\Users\vivek\OneDrive\Documents\resume_10.pdf" `
  --print-to-pdf-no-header --no-pdf-header-footer `
  "file:///C:/Users/vivek/OneDrive/Documents/Test.html" 2>$null
```

That produces a PDF that matched my print preview and didn’t add a phantom first page or browser footer.

---

## The symptoms

1. **Missing divider line** after Education on some converters.
   Cause: thin borders or pseudo‑elements often get dropped by older engines.
   Fix: switched to a literal `<hr class="rule">` with a standard 1px top border.

2. **Blank first page** in some exports.
   Cause: a `break-before: page` near the top (or a misapplied break element) can force an empty sheet.
   Fix: move the split to **`break-after: page` on the prior section**.

3. **Huge gaps between jobs**.
   Cause: aggressive `break-inside: avoid` on large containers — if the block doesn’t fit, the engine leaves a canyon and pushes the whole thing.
   Fix: relax `break-inside` on big wrappers during print; keep it surgical.

4. **Extra whitespace + time/URL in footer**.
   Cause: Chrome adds headers/footers by default and reserves space even in headless.
   Fix: pass **both** flags or set `displayHeaderFooter: false` in automation.

---

## The mental model (first principles)

* An HTML page is laid out by a **rendering engine** (Chromium/Blink in Chrome/Edge). Printing switches to **paged media** rules: `@media print`, `@page` (paper size/margins), and the `break-*` family.
* **Engines differ**. Online converters may run older WebKit or custom engines that ignore modern CSS (thin borders, flex/grid in print, hyphenation, some page-break rules). Expect variation. Having said that, I really liked this: https://htmlpdfapi.com/live_demo. And as a quick and dirty approach to convert html -> PDF, this website is a nice starting place.
* **Pagination** is an algorithm: the engine decides where to cut, how to handle orphans/widows, and how to obey `break-inside`. Too many constraints = jagged results.

### Polya, in two minutes (because I really like his Solve It framework!)

1. Understand the problem: “Same HTML, different PDF output; blank first page; missing lines.”
2. Devise a plan: reduce CSS to print‑safe primitives, then control the export tool.
3. Carry it out: swap pseudo‑lines for `<hr>`, simplify margins, remove over‑strict `break-inside`, force one controlled split.
4. Look back: confirm reproducibility via a single command (Edge headless).

---

## What I tried (and why each helped or hurt)

### 1) Using pseudo‑elements vs real rules

**Before:**

```css
.company::after { border-bottom: 0.5px solid #bdc3c7; }
```

**After:**

```html
<hr class="rule">
```

```css
.rule { border: 0; border-top: 1px solid #bdc3c7; height: 0; margin: 10px 0 0; }
```

Why: some engines drop hairline borders or pseudo‑elements during pagination. `<hr>` survives everywhere.

### 2) Grid vs Table for Education

Grids sometimes reflow unpredictably in print. A **3‑column table** (`Year | Institution | Degree`) locks alignment:

```html
<table class="edu-table">
  <tr>
    <td class="edu-year">20XX</td>
    <td class="edu-inst">FMS</td>
    <td class="edu-degree">MBA (Finance & Strategy)</td>
  </tr>
</table>
```

```css
.edu-table { width: 100%; border-collapse: collapse; table-layout: fixed; }
.edu-year { width: 60px; font-weight: 600; white-space: nowrap; }
.edu-degree { text-align: right; font-weight: 600; white-space: nowrap; }
```

### 3) Print CSS that’s boring on purpose

```css
@page { size: A4; margin: 12mm; }
@media print {
  html, body { width: 210mm; margin: 0; padding: 0; }
  body { font-size: 11.75px; }
  .section { margin-bottom: 18px; padding: 0 8mm; }
  .experience-item { margin-bottom: 16px; }
  .experience-item.capgemini { break-after: page !important; }
}
```

* Keep `@page` simple; avoid funky margin tricks.
* Use **one** deliberate page split the last section on page 1 and the first section of page 2.
* Avoid global `break-inside: avoid` on large blocks.

### 4) Export paths I evaluated

* **Chrome Print dialog**: works, but easy to forget “Headers and footers: OFF” and “Background graphics: ON”.
* **Online HTML→PDF** sites: convenient, but engines vary; many ignore `@page` or drop thin lines.
* **Headless Chrome** (CLI): great when it honors the footer flags; sometimes noisy logging.
* **Headless Edge** (CLI): same Chromium engine, but consistently respected footer suppression on Windows in my tests.
* **Puppeteer/Playwright**: scripts that drive the browser programmatically. Using the **system Edge/Chrome** (not a downloaded Chromium) reduces layout drift.

---

## The exact commands that worked on my machine

### Edge headless (no window)

```powershell
$edge = "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe"
& $edge `
  --headless=new --disable-gpu `
  --log-level=3 --disable-logging `
  --print-to-pdf="C:\Users\vivek\OneDrive\Documents\resume_10.pdf" `
  --print-to-pdf-no-header --no-pdf-header-footer `
  "file:///C:/Users/vivek/OneDrive/Documents/Test.html" 2>$null
```

### Chrome headless (equivalent)

```powershell
$chrome = "C:\Program Files\Google\Chrome\Application\chrome.exe"
& $chrome `
  --headless=new --disable-gpu `
  --log-level=3 --disable-logging `
  --print-to-pdf="C:\Users\vivek\OneDrive\Documents\resume_10.pdf" `
  --print-to-pdf-no-header --no-pdf-header-footer `
  "file:///C:/Users/vivek/OneDrive/Documents/Test.html" 2>$null
```

### Playwright script pinned to Edge (optional)

```bash
npm i -D playwright
```

```js
// export-edge.js
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch({ channel: 'msedge', headless: true });
  const page = await browser.newPage();
  await page.goto('file:///C:/Users/vivek/OneDrive/Documents/Test.html', { waitUntil: 'networkidle' });
  await page.pdf({
    path: 'C:/Users/vivek/OneDrive/Documents/resume_10.pdf',
    format: 'A4',
    printBackground: true,
    preferCSSPageSize: true,
    displayHeaderFooter: false
  });
  await browser.close();
})();
```

Run:

```bash
node export-edge.js
```
---

## ELI5: what “printing a web page” does

* The browser switches to **print rules** and chops your long page into paper sheets.
* Some CSS becomes more important (`@page`, `@media print`, `break-*`).
* If you ask for “don’t break here” too often, the browser leaves big holes to obey you.
* If you use a converter with an older brain, it may ignore your fancy CSS. Pick a stable brain (Edge headless) and keep your print CSS boring.

## Quick troubleshooting checklist

* **Footer shows time/URL** → you forgot to disable headers/footers (flags or `displayHeaderFooter:false`).
* **Blank first page** → move the split to `break-after` on the previous section; remove stray break elements near the top; keep `@page` margins modest (≈12mm).
* **Missing lines** → use `<hr>` with 1px border; avoid 0.5px hairlines.
* **Columns misalign** → use a simple table for that section; set `table-layout: fixed`.
* **Random big gaps** → don’t blanket-apply `break-inside: avoid` on large containers.
* **Different look between tools** → remember each converter has a different engine. Use one consistent path (Edge headless or Playwright with `channel: 'msedge'`).

---

## What I’d do next time (and recommend to others)

* Develop with **Chrome/Edge Print Preview open**; adjust `@media print` as you go.
* Keep a **one‑click export** (BAT file or Node script) that bakes in the right flags.
* Use print‑friendly building blocks (tables, `<hr>`, 1px borders, modest margins).
* Limit page breaks to **one or two deliberate splits**.
* If you must use an online converter, test with several before trusting the output.

---

## Appendix: compact print CSS I ended up with

```css
/* Section rule */
.rule { border: 0; border-top: 1px solid #bdc3c7; height: 0; margin: 10px 0 0; }

/* Education table */
.edu-table { width: 100%; border-collapse: collapse; table-layout: fixed; }
.edu-year { width: 60px; font-weight: 600; white-space: nowrap; }
.edu-inst { font-weight: 500; padding-right: 16px; }
.edu-degree { text-align: right; font-weight: 600; white-space: nowrap; }

/* Paged media */
@page { size: A4; margin: 12mm; }
@media print {
  html, body { width: 210mm; margin: 0; padding: 0; }
  body { font-size: 11.75px; }
  .section { margin-bottom: 18px; padding: 0 8mm; }
  .experience-item { margin-bottom: 16px; }
  /* single intentional split so Infosys starts on page 2 */
  .experience-item.capgemini { break-after: page !important; page-break-after: always !important; }
}
```

That’s the journey: fewer mysteries, more control, and a PDF that looks the way I intended—every time.

---

