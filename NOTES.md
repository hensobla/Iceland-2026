# Iceland 2026 — site notes

Context and decisions for `index.html`. Read this before editing.

**What it is:** a single-file microsite for a private Iceland trip, 9–17 August 2026. Windstar *Star Pride* cruise plus five private land days. Built to be shared as a link with a travel group. One file, ~78 KB, ~1,115 lines, no build step, no dependencies.

---

## 1. Hard constraints

These are not preferences. Each one was discovered by shipping something that broke.

**No *remote* images.** An early version pulled photographs from Wikimedia Commons via `Special:FilePath` and **not one loaded**; remote requests are blocked in the viewing environment. Do not reintroduce them. The logo and monogram are inline SVG (see §8).

⚠️ **The file is no longer self-contained.** All nine day cards now use painted PNGs from `assets/images/`, referenced by relative path. `index.html` on its own renders those cards empty. **The folder has to travel with the file**, which gives up the one-link property the site was built around. See §13 before sharing.

**No hash navigation.** `<a href="#aug12">` was treated as a document navigation and tried to leave the page. Every in-page jump is a `<button>` plus `window.scrollTo()`. Do not add anchor links.

**Must work offline** after first load. See §8.

**No em-dashes.** Anywhere. En-dashes in time ranges (`09:45 – 11:40`) are fine and intentional.

---

## 2. Layout of the file

```
<head>
  Google Fonts link          ← the only external request
  <style>                    ← all CSS, no framework
<body>
  <header class="hero">      ← eyebrow, h1, lede, countdown/map slot, glance line, COMPLETE stamp
  <nav class="nav">          ← sticky day pills (populated by JS)
  <main class="wrap">        ← everything else (populated by JS)
  back-to-top button
  DEV PANEL markup           ← removable
  <script>                   ← 741 lines
  footer
```

The `<script>` runs top to bottom in this order: **icons → artwork → day data → render functions → nav/scroll → clock/map → dev panel → `renderAll()`**. Nothing executes until `renderAll()` at the very bottom.

---

## 3. Data model

`days[]` is the single source of truth. One object per calendar day, `aug9` through `aug17`.

| Key | Purpose |
|---|---|
| `id`, `pill`, `date` | anchor id, nav pill label `['Sun','9']`, header date |
| `place`, `pron` | display name and pronunciation subtext |
| `badge`, `kind` | corner label, and which SVG scene to draw |
| `ports` | `['In 09:30','Out 18:00']` — ship times, rendered in the blue bar |
| `calls` | only when a day has **more than one** port call. `[{name, t:[in,out], sub}]`, and it supersedes `ports`. The 16th is Heimaey then Surtsey. Single-call days keep using `ports`. |
| `meta` | quiet middot line: guide, span, distance |
| `note` | `{k:'soft'|'warn', b:heading, p:body}` |
| `summary`, `start`, `stops[]`, `end` | the collapsible Itinerary panel |

Each entry in `stops[]` is `{leg:['45 min','45 km'], t:'09:45 – 11:40 · 1 hr 55', ttl, pron, i:iconKey, hero, desc}`.

`pickup` is an optional **day-level** field: a plain address string. The query is built with `encodeURIComponent`, so store the address and never a URL.

> **It renders in the card body, not in the timeline.** It went inside the Itinerary panel first, which buried the one thing the group has to act on before the day starts behind a tap. Anything that is a *prerequisite* for the day belongs above the `<summary>`; the timeline is for what happens once it is underway.

The block pairs `d.start` as its heading with the address beneath, and the whole thing is one link target. When a day has a `pickup`, the `start` rail is **suppressed inside the timeline** so the time is not printed twice on one card. It is styled from the ports bar's glacier family but with a heavier border and a press state, because it is the only block on a card that is a thing to do rather than a thing to read. It is an outbound link and the only part of a day card that does nothing offline.

> `leg` is the **drive that precedes the stop**, not a stop itself. This was a deliberate restructure: rendering drives as list items doubled the row count and made the timelines unreadable. They are now thin connectors between real stops.

The leg row carries the **car glyph and no word for it** — "45 min · 45 km" on its own read as elapsed time of unclear kind. The icon is the label, which is what keeps the row to one line. It is deliberately uncircled and lighter than a stop's icon so the two never read as the same rank. The bare connector above a `d.end` rail stays iconless: it marks distance, not a drive.

`hero:1` marks the one stop per day worth highlighting in ember. Use it sparingly — one per day, or the emphasis stops meaning anything.

---

## 4. Rendering

**Times are stored in 24h and displayed in 12h.** `days[]` keeps the operator's own notation so §10's line-by-line reconciliation against the PDF still works. `h12()` converts on the way to the screen and is the single place the display format is decided; it is applied to `ports`, each `meta` entry, `start`, `end` and each stop's `t`, plus the eclipse paragraph. A range carries one meridiem when both ends share it (`9:45 – 11:40 am`) and both when they differ (`11:35 am – 12:05 pm`).

> **`desc` and `note` prose are not converted.** Only the time-bearing fields go through `h12()`, so any clock time written inside a description must be typed in 12h by hand or it will sit in 24h next to converted times. The 9 August dinner note ("the coach reaches the hotel at 8:45 pm") and the footer's "1:30 pm Ísafjörður departure" are the two places this applies. If you change `h12()`'s style, both drift and must be matched by hand.

Do not run `h12()` over whole rendered blocks. It is applied per field on purpose, so it can never reach an SVG or an attribute.

`renderAll()` is the only entry point. It calls `computeToday()`, rebuilds `main.innerHTML` and `pills.innerHTML`, re-binds the `links` and `sections` arrays, wires the logo, resets `cdMode`, and calls `tick()`.

Anything that changes the displayed date must go through `renderAll()`. The click handler on `pills` is delegated, so it survives the innerHTML replacement.

**Do not call `scrollIntoView()` on a nav pill.** This caused a real bug: the scroll-spy centred the active pill with `scrollIntoView()`, which scrolls *every* scrollable ancestor including the document, which nudged the page, which re-fired the spy, which centred the next pill. Tapping "Sun 9" would walk down the page on its own. Pills are centred with `pills.scrollTo({left})`, which only touches the horizontal strip.

The spy itself is a plain "last section whose top has crossed the line" check on a rAF-throttled scroll handler, not an IntersectionObserver. A 1.1 s `lock` after a tap stops the spy overriding the target mid-scroll.

---

## 5. Time behaviour

Every clock read goes through **`NOW()`**, never `Date.now()` directly. That indirection is what makes the dev panel work.

| Window | Hero shows |
|---|---|
| before 9 Aug | countdown to local midnight on the 9th |
| 9 Aug | map, "Golden Circle today. You board tomorrow." |
| 10–17 Aug | map, "Day N of 8", pulse on that day's port |
| 18 Aug onward | map, date range, plus the COMPLETE stamp |

The switch is a **local ISO date string comparison**, not a UTC timestamp, so it flips exactly when the viewer's own clock rolls over. This matters because the Today marker is also local-date based; using a UTC boundary would let the two disagree for a few hours.

**`refreshToday()` handles date rollover in place.** The page is expected to sit open for days. Every tick recomputes the date, and on a change it patches the pill labels, moves the Today badge **and the card's `.now` class**, and re-renders the map caption — without rebuilding `main`, which would collapse any open Itinerary panel. `visibilitychange` and `pageshow` listeners force a catch-up tick after a phone suspends timers or restores from bfcache.

**Day number comes from the array index**, `days.indexOf(todayDay)`, not elapsed milliseconds. The original elapsed-ms version read "Day 7" on the morning of the 17th because 6 days 22 hours floors to 6.

---

## 6. The map

Coastline is 207 vertices plotted through an equirectangular projection that corrects for longitude convergence at 65°N:

```
x = (lon + 24.6) × 29
y = (66.6 − lat) × 63
```

Any new point (a port, a marker) must use the same two lines or it will land in the wrong place.

Every vertex carries a **quadratic fillet**: back off 2 units along each adjacent edge, curve through the original point. The cut is clamped to half the adjacent edge, so open coasts soften fully while tight fjord walls barely round at all — that clamp is what keeps Hrútafjörður and Reyðarfjörður reading as slots rather than lobes. The filleted path was tested for self-intersections after flattening; it has none. **If you edit coastline points, re-run that test** — a larger radius bulges across the narrowest fjords.

The dashed sailing route uses the same fillet at radius 9, since it has no detail to preserve.

The current port gets three concentric rings scaling r4 → r36 over 5 s, staggered 1.67 s apart. `vector-effect="non-scaling-stroke"` keeps them hairline as they grow. Collapses to one static halo under `prefers-reduced-motion`.

---

## 7. Artwork

Eight scenes in `art(kind, u, h)`: `gullfoss, reykjavik, kirkjufell, eclipse, whale, fjord, volcano, dusk`. `fjord` is used twice (14th and 15th).

**Every scene needs a hero subject.** The first version was generic layered ridges with a prop in front and looked like clip-art. What works: a specific, recognisable thing rendered with real care — Hallgrímskirkja's stepped wings, a humpback fluke with the centre notch, Eldfell's notched rim, the Blue Church at the head of a perspective rainbow street.

**The Golden Circle scene was rebuilt from a photograph** and two things it learned are worth keeping. **A waterfall needs a dark back wall.** Drawn against the bright sky the curtain reads as a flat white trapezoid, because its own edges are the only thing defining it; the `back` polygon behind it gives the white something to be white *against*. And **soft masses use the `glo` radial fade**, not flat ellipses, which show their outline plainly at card width. Draw order carries the depth: back wall, falls, cliffs over the curtain's outer edges, gravel, then spray last so the mist billows in front of both the cliff feet and the pool.

**The Seyðisfjörður scene is built around the rainbow street**, drawn as one perspective wedge from the bottom edge up to the church door, with the church seated at its head rather than floating above it. Bands run *across* the road the way the real paint does, cool at the top so they hand off to the church's own blue and warm at the foot where the eye starts. The fjord walls and houses are framing only: pulled back, desaturated, off the centre line.

> **Watch the veil when you place a subject.** `art()` lays a darkening gradient over the bottom third for the caption. A first pass put the street down there and the veil turned the rainbow to mud. The scene now sits high, and the three warm bands are mixed lighter than they look in isolation *because* they are read through the veil. Judge those colours on the card, never in isolation.

**Size the caption zone for a phone, not for your screen.** The caption is a fixed pixel height sitting in an art box whose height scales with the card, so on a 375px-wide phone it covers the bottom **~47%** of the 420-unit viewBox against roughly 24% on a wide card. Anything busy or pale above y≈300 will collide with the date line on a phone while looking perfectly clear on a desktop. This is why Heimaey gives the lower half of the frame to plain dark scoria and rides its landscape up on a `translate`, and why the other scenes get away with less: their lower halves are already dark (fjord walls, lava, open sea, gorge). Check a new scene at phone width before calling it done.

**Kirkjufell was rebuilt from a photograph too**, and it added two rules. **A falling water shape must dissolve, not end.** Drawn as a solid white column with a bottom edge it reads as a pillar standing in front of the cliff; `falG` fades it to near-transparent at the base so it breaks into the mist instead of stopping. And **the silhouette is the subject**: Kirkjufell is deliberately asymmetric, a short steep left flank against a long concave sweep off the right with a low tail to the shore. An early pass drew it as a symmetric triangle and it stopped being Kirkjufell and became any mountain.

> That scene is a **waterfall between two mossy headlands, which is Skógafoss on the south coast, not Gullfoss.** The reference photo was Skógafoss. The day's actual waterfall stop is Gullfoss, which is a two-tier staircase turning into a canyon and looks nothing like this. The scene key is still `gullfoss`. Rebuild it against a Gullfoss photograph if the likeness ever matters.

Shared techniques worth preserving: three-stop sky gradients with a warm horizon band, distant ridges desaturated for atmospheric depth, a consistent light direction with lit and shadow faces split, and a bottom veil gradient so the white caption text stays legible.

`church(x, baseY, scale, litColour, shadeColour, mode)` draws Hallgrímskirkja in local coordinates (base y=260, apex y=0) and is reused in daylight on the 10th and in silhouette on the 17th.

> **Gradient IDs must stay unique.** Every `<defs>` id is suffixed with `u`, the day index. Several scenes render on one page; duplicate ids will cross-contaminate fills.

---

## 8. Offline

Everything except the Google Fonts stylesheet is inside the file. With no connection the page renders identically in system fonts, because the stacks fall back deliberately:

```css
--sans:  'Figtree', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, …
--serif: 'Newsreader', 'Iowan Old Style', 'Palatino Linotype', Palatino, Georgia, serif
```

The fonts can't be base64-embedded from this environment because it can't reach `fonts.gstatic.com`. If you can download the woff2 files, inlining them would make the page fully self-contained.

**The VoyageWell logo is now inline SVG** and the page makes no image requests at all. It was previously an `<img>` pointing at `voyagewell.com/brand/logo-white.svg`, with the "JD" monogram as a fallback and a `haslogo` class added only on `onload`. `logo-white.svg` was later dropped into the folder, so the wordmark is now pasted into `agentHTML` directly and the `<img>`, the `haslogo` CSS and the `onload` wiring are gone.

**The monogram** at the top of the hero came from `monogram.svg` and got the same treatment. Two extra things were done to it: its `cls-1` black was mapped to `--ink` so it sits in the page's palette rather than next to it, and **its viewBox was cropped to the glyph's own bounds** (`14.9 205.1 970.2 589.7`). The source pads the mark inside a 1000 square, so an uncropped `height:31px` rendered a mark only about 18px tall. If you swap the file, re-measure with `getBBox()` rather than assuming the viewBox is tight.

The paste is derived from `logo-white.svg`, not identical to it: the `<defs><style>` block was resolved into `fill` attributes on each path, because an inline `<style>` inside SVG applies to the whole document and `.cls-1` … `.cls-4` are exactly the kind of generic names that collide later. The invisible `class="cls-1"` hairlines and the `id="Layer_1"` were dropped for the same reason. **If the logo is ever replaced, run the source through the same treatment** — do not paste a raw export.

---

## 9. Design decisions

**Today is marked twice, at two scales.** The ember "Today" chip identifies the card once you are looking at it; `.day.now` makes the card findable while scrolling past. That treatment is an ember border, a 4px `--ember-pale` halo holding it off the paper, and a warm ember cast in the drop shadow. Ember is reserved for *now* and for the one `hero` stop per day. Spending it anywhere else costs both.

**The hero is one line per level, and nothing repeats a level.** Mark (whose it is), eyebrow (when), h1 plus its italic (what), middot row (which ship, which route), lede (who runs the land days). Anything secondary goes *below* the countdown: the glance line and the Windstar link. An earlier version put the source link between the lede and the countdown as a bordered card, which gave a reference the same weight as the headline and broke the run from title to countdown.

The hero's middot row reuses the day card's `.meta`, deliberately: "which ship" up here reads the same way "which guide" does further down.

**Rúnar leads**, so he is named first wherever guides are listed and carries the `<strong>` in the hero lede. The 14th lists him ahead of Pétur for the same reason.

**Weight restraint.** Nothing is 800 and almost nothing is 700. Hierarchy comes from size, colour, and the serif/sans contrast. Newsreader 500 for place names, stop titles and headings; Figtree 400 for body, 500 for numbers and the few words that need to pop.

There are exactly **two deliberate 700s**: the "TODAY" nav label, and the day card's date line, which went bold once the cards became photographs and a 500 stopped holding its own against them. Figtree 700 is loaded from Google Fonts, so neither is a synthesized fake bold — check the stylesheet link before adding a third.

**Letter-spacing costs padding.** Tracked uppercase text carries its tracking after the last glyph, so a chip with symmetric padding looks shifted left. The badge uses `padding-right: calc(10px − .08em)`; centred labels in the nav pills and countdown cells use a matching `text-indent`. Apply the same correction to any new tracked label.

**The Itinerary panel is animated by hand.** `<details>` will not transition its own height, so the summary click is intercepted, `.tl` is animated from its measured height with the Web Animations API, and the rows lift in behind it. Three things follow from that and are easy to undo by accident:

- **`open` stays `true` for the whole of a close** and is only set false in `onfinish`. A closed `<details>` does not render its children, so flipping it early would make the panel vanish instead of retract.
- **The listener is delegated on `main`**, because `renderAll()` replaces `main.innerHTML` and would drop anything bound per element.
- **`prefers-reduced-motion` returns early** and lets the browser toggle natively. It does not just shorten the animation, because a zero-length animation still runs the intercept and would leave the panel mid-flight.

Duration scales with panel height (230–380 ms), and a `sliding` flag drops taps that land mid-animation.

**The itinerary is flat, and that is the point.** It used to be a bordered, filled box inside the card, with the timeline as a second box below it. That put three surfaces in front of the reader — page, card, drawer — and the two outer ones carried no information. It now has no fill, no border and no radius: a hairline separates it from the body above, and the timeline sits directly on the card. Two surfaces, one of which is the content.

Two things quietly depend on that panel having had a background, and both are now wired to the card instead:

- **`--tl-bg`** is what the drive icons fill themselves with to mask the dotted spine behind them. It follows whatever surface the timeline sits on, so it is `var(--card)` now. Change the surface, change this, or the spine runs through the car glyphs.
- **The hero wash** bleeds sideways to meet both card edges. It used to bleed by `.tl`'s 16px gutter; it now bleeds by `--gut`, the day body's own padding, which is 17px and becomes 21px at the 520px breakpoint. That is why the padding is a variable rather than a literal.

With no box edge to lift, the affordance moved onto the **chevron and the label**. The chevron is a filled 36px button, not a hairline ring — glacier-pale closed, solid glacier with a white glyph open. It carries the whole "press me" signal now, so the two states differ in **fill as well as rotation** and it reads as a toggle without being read. The label goes glacier on hover and stays glacier while open. Weaken any of that and the row goes back to looking like static text.

> The tap target is the **whole summary row**, 72px tall on a phone, not the 36px circle. The circle is the signal; the row is the hit area.

**`cursor: pointer` on `<summary>`.** The Itinerary panels didn't read as tappable and the cause was the browser default cursor for `<summary>`, not the visual design. The circled chevron, the faint tint and the press-scale are secondary.

**Specificity trap.** `<main>` also carries `class="wrap"`, so `main { padding-top }` loses to `.wrap { padding: 0 18px }` regardless of source order. Top spacing is set on `main.wrap`. Two earlier attempts to add that margin were silent no-ops.

---

## 10. Content accuracy

**Land days** come from the operator's PDF and were reconciled line by line: every stop time, duration label, drive time and distance now closes arithmetically.

Three things in the source PDF are wrong and are corrected on the site:

- **14 August** writes its time blocks differently from every other day — the block *includes* the stop rather than splitting drive and stop onto separate lines. Mjóifjörður is 10:15–10:35 and Klifbrekkufossar 11:05–11:50, not what a naive read gives.
- **Þingvellir** is labelled "2 hours" but 09:45–11:40 is 1 hr 55. **Whale watching** is labelled "2.5 hours" but 14:35–17:00 is 2 hr 25. The site shows the arithmetic.
- **"Bjarnafoss"** is a typo for **Bjarnarfoss**, the waterfall above Búðir.

Also worth knowing: the PDF's day headers overstate their own length (11 and 14 August both say "12 Hours" but run 8h30 and 8h05). The site shows actual spans instead.

**Ship port times were reconciled against Windstar's own itinerary page** (linked in the hero) and three were wrong:

| Day | Was | Windstar says |
|---|---|---|
| 10 Aug Reykjavík | Board 08:00 | **Embark 13:00**, sail 17:00 |
| 12 Aug Ísafjörður | In 09:30 | **In 08:00**, out 13:30 |
| 16 Aug Heimaey | In 07:00\* | **In 10:00**, out 16:30, **then Surtsey 17:00 – 19:00** |

The other four match. The 17 August arrival is not listed on that page at all and is the one ship time still unconfirmed; the footer says so.

> The 12 August correction dissolved a conflict that was on the site for a while: the coach leaves at 08:30, which looked like an hour *before* the ship docked. It docks at 08:00, so the 08:30 departure is fine. The 14 August conflict is real and still stands (see §12).

**The page is client-rendered**, so fetching that URL server-side returns a 404 and the link looks dead. It is not. Open it in a browser.

---

## 11. Removable blocks

**The dev panel has been removed.** It was three blocks marked `DEV PANEL`: CSS near the media queries, markup above `<script>`, and wiring at the bottom of the script. It gave date simulation (presets plus a date picker), a live state readout, and a jump to the marked day. It came out before the site went public.

> Removing it was **not** as clean as this section used to claim. A bare `devState();` call sat on its own line after `renderAll()`, *outside* every marked block, and deleting the blocks left it behind to throw a `ReferenceError` on load. If you restore the panel from git history, restore that call too — and if you strip anything else, grep for callers rather than trusting the fences.

`let DEV_NOW = null, DEV_KEY = '';` and `NOW()` are deliberately still there. `NOW()` is the indirection every clock read goes through, and it now simply always returns `Date.now()`. Leave it: re-adding the panel means restoring the blocks, not rewiring the clock.

To bring it back, `git log` the file and revert the removal commit.

**The scheduling issues section has been deleted.** It held the two ship-versus-coach timing conflicts (12 August departing an hour before the ship docked, 14 August departing the exact minute it docked) and sat above the advisor card, which is now the natural closer as intended. Its `.flags` / `.flag` CSS and the `flagsHTML` reference in `renderAll()` went with it.

> The conflicts themselves were never resolved, only removed from the page. They are recorded in §12 so they are not lost.

The open-day callouts on the 10th, 15th and 16th were *not* part of that block and are still there. The distinction was deliberate: a day with nothing scheduled is useful context for the group; a timing conflict is a to-do for the trip owner.

---

## 12. Known gaps

- **One unresolved scheduling conflict, no longer shown on the site** (see §11). *14 August:* departure from Seyðisfjörður is booked for 9:30 am, the exact minute the ship docks, and gangway clearance normally takes ten to fifteen minutes. The 12 August conflict turned out to be an artefact of a wrong arrival time and is resolved (see §10).
- No offline map of the ports beyond the hero SVG.
- 17 August arrival time unverified; Windstar does not list the disembarkation morning.
- Photo aspect is **1.60:1 against the card's 1.90:1**, so `slice` trims about 8% off the top and bottom of every one. Rendering the set at 1.90:1 would recover it.
- Fonts and the photos now stand between this and offline self-containment. Fonts were the last external request; the photos are local but still separate files.

---

## 13. Photographs

**All nine day cards carry a painted PNG.** The drawn eclipse scene survives on the **Totality block**, the separate `ecHTML` section below the 12th, which calls `art('eclipse','EC',300)` and takes no photo. That is the only drawn scene still rendering, plus `gullfoss` as the fallback when a `kind` does not match.

> The 12 August *day card* and the Totality *block* are two different things. The card is Ísafjörður and takes `isafjordur.png`; the block underneath is the eclipse itself and keeps the artwork. Do not give the block a photo.

`photo` is a day-level field holding a **bare filename**. `art()` prefixes `assets/images/` and runs `encodeURI`. Filenames are plain ASCII, lowercase and hyphenated; the originals carried Icelandic characters and spaces, which worked but made every reference fragile to copy, quote and shell-escape.

The image is drawn **inside the same `<svg>`** as the drawn scenes, at the same 800×420 with `preserveAspectRatio="xMidYMid slice"`. That is deliberate: the caption veil, the badge, the Today chip and the art radius all keep working with no separate code path.

| Day | File |
|---|---|
| 9 Aug | `golden-circle.png` |
| 10 Aug | `reykjavik-start.png` |
| 11 Aug | `grundarfjordur.png` |
| 12 Aug | `isafjordur.png` |
| 13 Aug | `husavik.png` |
| 14 Aug | `seydisfjordur.png` |
| 15 Aug | `iceland-colorful-houses.png` |
| 16 Aug | `heimaey.png` |
| 17 Aug | `reykjavik-end.png` |
| Totality block | *drawn SVG, no photo* |

### Sizing

The card art is capped by `.wrap` at **620px minus 36px of padding, so 584px of display width**. The shipped WebPs are **1200px wide, a touch over 2x**, quality 82. That took the set from **36 MB of PNG to 2.4 MB**, about 270 KB a card, with no visible loss at card size.

`assets/images/archive/` holds the **full-resolution 1586×992 PNG masters** for all nine, plus the two never used (`puffin-cliffs.png`, `reykjavik-aerial-folk-gouache.png`). Nothing at runtime reads from `archive/`; it exists so the set can be re-encoded without regenerating the paintings.

To re-encode after changing a master:

```
cwebp -q 82 -resize 1200 0 archive/name.png -o name.webp
```

> Raise the 1200 only if `.wrap`'s max-width goes up. It is derived from that number, not chosen for its own sake.

> The drawn scenes for the seven replaced days are still in the file and now render nothing. They are roughly 300 lines of dead weight, kept only as a fallback while the photo approach settles. Delete them once it does, but keep `eclipse` and `gullfoss`.
