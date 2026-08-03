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
  <header class="hero">      ← eyebrow, h1, middot row, countdown/map slot, COMPLETE stamp
    div.m-rail                 ← the floating menu, sized to the hero (§14)
  <nav class="nav">          ← sticky day pills (populated by JS)
  <main class="wrap">        ← everything else (populated by JS)
  back-to-top button
  <dialog class="sheet">     ← the phrasebook (§14)
  <script>                   ← 741 lines
  footer
```

The `<script>` runs top to bottom in this order: **icons → artwork → day data → render functions → nav/scroll → floating menu → clock/map → `renderAll()`**. Nothing executes until `renderAll()` at the very bottom.

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
| `meta` | quiet middot line. On an excursion day it is exactly two parts, **guide then hours**, and is held to one line (§15). Other days say whatever that day needs and may wrap. |
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

Every clock read goes through **`NOW()`**, never `Date.now()` directly. That indirection is what let the dev panel simulate dates; with the panel gone it simply returns real time, and it stays because restoring the panel means restoring blocks, not rewiring the clock.

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

**The Today ember edge is 2px, not 1px.** A hairline stroke on an 18px radius has almost nothing to antialias with around the arc, so at fractional card widths or browser zoom the corners render lighter than the straight runs and the ring looks like it stops short of wrapping. Two integer pixels carry the curve. The pale halo came down from 4px to 3px to keep the pair balanced. Do not take the stroke back to 1px, and do not make it fractional.

**The Totality block wears the Today chip too**, on the 12th, alongside the Ísafjörður card above it. That forced `ecHTML` from a const string into a **function**, since it has to re-read `TODAY` on every render. Two consequences: call it as `ecHTML()` in `renderAll()`, and patch it in `refreshToday()` as well — that path runs on date rollover *without* rebuilding `main`, so a render-time-only chip would go stale at midnight.

**The hero stop's wash is a background on the row, not a pseudo-element.** It has to bleed out to both card edges — held inside the row's own box it stops short on each side and reads as a floating rectangle — so `.stop.hero` widens itself by `-1 × var(--gut)` on each side and pays it straight back as padding.

It used to do that with a `z-index: -1` `::after` pinned to `inset: 0 calc(-1 * var(--gut))`, under an `isolation: isolate` on the row. That worked everywhere except iOS Safari, where it put the bleed on its own negative stacking layer, out of flow, inside a rounded `overflow: hidden` card — and the right-hand strip intermittently failed to raster. What showed was **a bare white bar down the right edge of the row, exactly one gutter wide**, covering however much of the row's height a dropped tile happened to span, so it looked like a partial-height rectangle rather than an obvious full-row miss. **Pinching to zoom made it disappear**, which is the tell: a forced repaint fixed it, so the geometry was never wrong, only the paint. A background on the row itself is in normal flow with nothing to compose.

> Two things move with the negative margin. The dotted spine is positioned from the row's left edge, so `ol.stops > li.stop.hero::before` pushes `left` from `13px` to `calc(13px + var(--gut))`; the icon column follows the padding on its own. Both were verified flush and aligned to the pixel at 375, 430, 620 and 900px, across all five hero rows — the check is worth repeating if `--gut` or the gutter breakpoint changes.

**Today is marked twice, at two scales.** The ember "Today" chip identifies the card once you are looking at it; `.day.now` makes the card findable while scrolling past. That treatment is an ember border, a 4px `--ember-pale` halo holding it off the paper, and a warm ember cast in the drop shadow. Ember is reserved for *now* and for the one `hero` stop per day. Spending it anywhere else costs both.

**The hero is one line per level, and nothing repeats a level.** Mark (whose it is), h1 (what), eyebrow (when), middot row (which ship, which route), countdown. That is the whole header and it ends on the countdown deliberately.

> **The eyebrow sits under the h1, not over it.** It ran above the title for a long while, which meant the first thing on the page was a date range in tracked ember caps and the word "Iceland" arrived second. Underneath, the title lands first and the dates read as its answer. Everything below it is then in descending order of how much you already knew.

**Space the hero by the gap you can see, not the one the box model reports.** In this stack they are a long way apart. `Iceland` has no descender, so about 14px at the foot of the h1's line box is empty; the eyebrow and the middot row both inherited the body's `line-height:1.62`, which parks another 4px of dead leading above their first pixel of ink. The result was a 15px margin drawing a 35px hole and an 18px margin below it drawing a 27px one: **the date read as further from the title than from the ship line, which is backwards.** Margins alone cannot be read off this file to know what it looks like.

Both lines now set their own line-height, and the margins are tuned against measured ink. The ladder is **20 / 28 / 46**, ink to ink: the date bound to the title, the ship line a step down, the countdown its own block.

| | margin | optical |
|---|---|---|
| `Iceland` → date | 4px | 20px |
| date → ship line | 25px | 28px |
| ship line → countdown | 38px | 46px |

> **`margin-top:4px` on `.eyebrow` is not a typo**, and the h1's empty descender space is the reason. Do not "correct" it back up to something that looks normal in the stylesheet without measuring what it does on screen.

> `.eyebrow` gets `line-height:1` because it is one unwrappable line of tracked capitals. `header.hero .meta` keeps 1.35 rather than going to 1 because the ladder above was measured against that line box; move it and the 28 and the 46 move with it.

**The hero's middot row never wraps. It shrinks to fit instead.**

```css
header.hero .meta{ flex-wrap:nowrap; white-space:nowrap;
                   font-size:min(15px, (100vw - 76px) / 19.6); }
```

`flex-wrap` because `.meta` is a flex row and would otherwise break between the spans; `white-space` for the hyphen inside "round-trip". Full 15px down to a 370px screen, shrinking below that. This is scoped to the hero: **the day cards' `.meta` still wraps, and should**, since those lines are longer and genuinely need two.

The `76px` is 36 for `.wrap`'s padding, 32 for the two separator margins (set in px, not em, so they do not scale with the type and have to come off the top), and 8 of slack for a desktop scrollbar inside `100vw`.

> **The `19.6` is the trap, and it is not Figtree's number.** It is the string's width in em taken from the widest font in the stack *at the smallest size the rule can pick*, which is a different measurement from the obvious one.
>
> | | at 15px | at 11px |
> |---|---|---|
> | Figtree | 17.94 | 17.94 |
> | Arial / Helvetica | 18.10 | 18.10 |
> | **-apple-system** | **18.45** | **19.31** |
>
> Two separate things are going on. **Sizing to Figtree is wrong** because the fallbacks are what an offline reader gets (§8). And **-apple-system is optically sized**: San Francisco swaps to its Text cut below 20px and gets *wider* as the type gets smaller, so a constant read off a large sample does not hold at a small one. A first pass used `18.1`, measured from a 100px sample, and San Francisco overflowed by 0.7px at 320px: a bug that would have appeared on iPhones and on nothing else.
>
> Re-measure at 11px, not at 100px, and check every family in the stack. Verified at 280, 320, 360, 375 and 390px across Figtree, San Francisco, Arial and the generic sans; the tightest case has 9.6px to spare.

**The countdown is left aligned on the same rail as everything above it.** The cells do not stretch: `flex:1` across the full measure parked "05" a column's worth of centring in from the margin, which is invisible while the cells have borders and obvious the moment they do not. They size to their own content and the gap carries the width, fluid so four columns still fit a 320px screen. Measured, the h1, the eyebrow, the middot row, the first number and its label all start on the same pixel.

> **Four things were cut from it once the floating menu existed.** The lede naming the guides, the glance line ("five ports, eight days aboard…"), the Windstar provenance link, and the boxes around the countdown cells. The first three were all *secondary* material parked below the countdown, which meant the header did not end anywhere: it trailed off through two greyed-out lines into a link. The guides are named on every day card that has one, and Windstar's itinerary is now one tap away in the menu, so none of it was load-bearing. Do not put a second tier back under the countdown. If something new has to live in the hero it belongs above the countdown or in the menu.

The countdown cells lost their border and fill in the same pass. Four bordered boxes sat directly above the nav's day pills, which are also bordered boxes in a row, and the two read as one repeated motif at two sizes. The numbers now close the hero on their own. The 13px of padding the cells used to carry moved onto `.countdown`'s `margin-top`, so the gap off the middot row did not change.

> `.cd-msg` is **dead CSS**, and was already dead before this. The map phase writes `.mapwrap` plus `.mapcap`, and `.mapcap` never had a box, so it needed no matching change.

The hero's middot row reuses the day card's `.meta`, deliberately: "which ship" up here reads the same way "which guide" does further down.

**Rúnar leads**, so he is named first wherever guides are listed. He used to carry the `<strong>` in the hero lede as well; with that line cut, the day cards' `meta` rows are the only place the guides appear. The 14th lists him ahead of Pétur for the same reason.

**Weight restraint.** Nothing is 800 and almost nothing is 700. Hierarchy comes from size, colour, and the serif/sans contrast. Newsreader 500 for place names, stop titles and headings; Figtree 400 for body, 500 for numbers and the few words that need to pop.

There are exactly **two deliberate 700s**: the "TODAY" nav label, and the day card's date line, which went bold once the cards became photographs and a 500 stopped holding its own against them. Figtree 700 is loaded from Google Fonts, so neither is a synthesized fake bold — check the stylesheet link before adding a third.

**Letter-spacing costs padding.** Tracked uppercase text carries its tracking after the last glyph, so a chip with symmetric padding looks shifted left. The badge uses `padding-right: calc(10px − .08em)`; centred labels in the nav pills use a matching `text-indent`. Apply the same correction to any new tracked label.

> **The correction is for centred labels only, and `.cd-lab` no longer takes it.** It carried one while the countdown cells were centred. Now that they are left aligned there is nothing to re-centre, and the indent would only push DAYS off the left edge its own number sits on. Adding `text-indent` to a left aligned tracked label is the mistake this note causes if you read the first half of it and stop.

**The Itinerary panel is animated by hand.** `<details>` will not transition its own height, so the summary click is intercepted, `.tl` is animated from its measured height with the Web Animations API, and the rows lift in behind it. Three things follow from that and are easy to undo by accident:

- **`open` stays `true` for the whole of a close** and is only set false in `onfinish`. A closed `<details>` does not render its children, so flipping it early would make the panel vanish instead of retract.
- **The listener is delegated on `main`**, because `renderAll()` replaces `main.innerHTML` and would drop anything bound per element.
- **`prefers-reduced-motion` returns early** and flips `open` with no animation. It does not just shorten the animation, because a zero-length animation still runs the intercept and would leave the panel mid-flight.
- **The summary's default action is always prevented**, in both motion modes, because `slidePanel` owns `open` in both. It used to be left in place under reduced motion so the browser could do the toggling — but the reduced-motion branch flips `open` as well, and a summary click runs its listeners *before* the default action, so the panel opened during dispatch and the browser shut it again on the way out. **Two toggles, no change: the itinerary could not be opened at all with reduced motion on.** That is the whole content of the page, gone, for a preference plenty of people have set without thinking about it. Do not make the `preventDefault` conditional again.

Duration scales with panel height (230–380 ms), and a `sliding` flag drops taps that land mid-animation.

**The foot button's ride back up starts with the fold, not after it.** Collapsing from the bottom of a long panel removes everything above the button, so the viewport is left looking at the next card and has to be returned to the top of the one being closed. That scroll used to run from `onfinish`. Sequential, it read as two separate events — the panel shut, a beat passed, then the page moved on its own — and the second half was the jarring one, because nothing the finger did appeared to cause it. Both now start in the same frame and it reads as one gesture.

The scroll target is safe to measure at click time: the button and everything collapsing sit *below* the card's top, so the card's own position in the document is the one thing the close cannot move. Verified landing pixel-exact on all five panels.

> This needed a scroll with **a duration**, which `behavior: 'smooth'` does not take — hence `glideTo`. Three things it must keep doing, all of them learned the hard way:
>
> - **Re-read the reachable maximum every frame.** The document is shrinking underneath the scroll as the panel folds; a target fixed at the start gets clamped short.
> - **Yield the instant the user takes over.** A wheel, touch or key ends it. The native smooth scroll does this for free, and a hand-rolled one that holds the page hostage for half a second is worse than the problem it fixes.
> - **Bind those listeners one frame late.** Enter on a focused button fires its click while that same keydown is still travelling up to `window`, so binding immediately hands the scroll back to a key the user has already finished pressing and kills it before it moves.
>
> Anything that scrolls by another route calls `stopGlide()` first, or a glide left running overwrites it every frame and its listeners outlive it to cancel the *next* one.

Pacing: the fold keeps its 230–380 ms, the scroll runs `1.6 ×` that. Panel height is a fair proxy for the distance back up, so scaling off the fold scales off the distance. The fold lands around 70% of the way through the scroll, which is the intended shape — the page keeps gliding for a moment after the panel is shut and settles rather than stops. The scroll is eased at **both** ends where the fold is eased out only: it starts a screen away rather than under the finger, so it wants the softer entry. Quadratic, not cubic — a cubic entry is lazy enough against an eased-out fold to read as the page setting off *after* the panel, which is the thing being fixed.

**The itinerary is flat, and that is the point.** It used to be a bordered, filled box inside the card, with the timeline as a second box below it. That put three surfaces in front of the reader — page, card, drawer — and the two outer ones carried no information. It now has no fill, no border and no radius: a hairline separates it from the body above, and the timeline sits directly on the card. Two surfaces, one of which is the content.

Two things quietly depend on that panel having had a background, and both are now wired to the card instead:

- **`--tl-bg`** is what the drive icons fill themselves with to mask the dotted spine behind them. It follows whatever surface the timeline sits on, so it is `var(--card)` now. Change the surface, change this, or the spine runs through the car glyphs.
- **The hero wash** bleeds sideways to meet both card edges. It used to bleed by `.tl`'s 16px gutter; it now bleeds by `--gut`, the day body's own padding, which is 17px and becomes 21px at the 520px breakpoint. That is why the padding is a variable rather than a literal.

**There are two ways to close a panel.** The header button, and a second `Collapse` at the **foot of the timeline**. A long day runs well past a phone screen, and having to scroll back up to the header to shut it is the kind of friction nobody reports but everybody feels. The foot button lives inside `.tl`, so a closed `<details>` leaves it in the DOM and never renders it — no hide rule needed.

> Closing from the foot deletes everything above the button, so the viewport would be left looking at the *next* card. It therefore calls `goTo()` on the day it belongs to once the animation finishes, landing you back on that card's header.

**One function drives every open and close.** `slidePanel(det, scrollBackTo)` is called directly by both buttons. The foot button originally worked by synthesising a click on the `<summary>`, which **races the native `<details>` toggle**: when the default action landed first, the handler saw an already-closed panel and re-opened it. Do not reintroduce a synthetic summary click — call `slidePanel()`.

**The control sits beside the label, not opposite it.** `justify-content:flex-start` with a 14px gap. Space-between put roughly 370px of dead air between a word and its own button on a 584px card.

**The summary is two lines, not one.** `.hd` holds the label and the button as a space-between row; the preview sits beneath at full width. All three used to share one line, and once the control grew a word it was crowded — worst on a phone, where the preview wrapped to four lines *beside* the button. Splitting it also parks the button at the same point on every card at every width, so it stops moving as the preview length changes from day to day. No media query: the two-line form is simply better at both ends.

With no box edge to lift, the affordance is an explicit **pill button that names its action**: "Expand ⌄" becoming "Collapse ⌃". A bare rotating chevron asked the reader to infer both that it was pressable and what it would do; the word says it.

**It is ink, not glacier, and that is deliberate.** Glacier is the ports bar and the pickup tile, which are *information*. Ember is "today" and the hero stop. This is a *control*, so it sits in the neutral family and borrows meaning from nothing. Putting it in glacier made it read as another data chip.

Two details that stop it feeling cheap:

- **Both words live in the markup**, one hidden per state, rather than being swapped by `::after { content }`. Generated content is not reliably exposed to assistive tech, and `<summary>` should carry a real label.
- **`min-width:116px`** because "Collapse" is wider than "Expand" and the button would otherwise resize as it toggles, jerking the row.

> The tap target is the **whole summary row**, 72px tall on a phone, not just the pill. The pill is the signal; the row is the hit area.

**`cursor: pointer` on `<summary>`.** The Itinerary panels didn't read as tappable and the cause was the browser default cursor for `<summary>`, not the visual design. The circled chevron, the faint tint and the press-scale are secondary.

**Specificity trap.** `<main>` also carries `class="wrap"`, so `main { padding-top }` loses to `.wrap { padding: 0 18px }` regardless of source order. Top spacing is set on `main.wrap`. Two earlier attempts to add that margin were silent no-ops.

---

## 10. Content accuracy

**Land days** come from the operator's PDF and were reconciled line by line: every stop time, duration label, drive time and distance now closes arithmetically.

Three things in the source PDF are wrong and are corrected on the site:

- **14 August** writes its time blocks differently from every other day — the block *includes* the stop rather than splitting drive and stop onto separate lines. Mjóifjörður is 10:15–10:35 and Klifbrekkufossar 11:05–11:50, not what a naive read gives.
- **Þingvellir** is labelled "2 hours" but 09:45–11:40 is 1 hr 55. **Whale watching** is labelled "2.5 hours" but 14:35–17:00 is 2 hr 25. The site shows the arithmetic.
- **"Bjarnafoss"** is a typo for **Bjarnarfoss**, the waterfall above Búðir.

> Corrections live here, not on the page. The site used to carry "Your sheet spells it Bjarnafoss" in the Bjarnarfoss description and `leg: ['no transfer listed', '']` above the 13 August lunch. Both were notes about the *source document*, addressed to whoever was reconciling it — no use to someone reading their own itinerary, and faintly alarming to be told their paperwork is wrong. The site shows the corrected fact and says nothing about where it came from; the lunch simply has no drive row, because there is no drive. Keep new discrepancies in this section.

**Sagas & Songs is deliberately absent.** The 15 August note used to end "Windstar usually schedules its Sagas & Songs performance during this overnight, so check the daily programme." It was briefly replaced with a researched note on the 14th and then **cut entirely, by request**. Do not put it back.

The research, so nobody repeats it: it is a real and complimentary Windstar Destination Discovery Event in the Blue Church during the Seyðisfjörður overnight — a local duo, about 45 minutes, drinks and canapés, ring dance outside — and it runs late afternoon or early evening, so on this sailing it would fall on the **14th**, after the 5:35 pm return, not on the free morning of the 15th. The **hour is not publicly checkable**: Windstar sets it on board and puts it in the daily programme, no public page carries the per-sailing time, and `windstarcruises.com` returns 403 to anything that is not a browser, so it cannot be scraped either.

That is the shape of anything shipboard. This site covers the **land days** — the five days Rúnar, Kuba and Pétur run — plus port times and what is booked. Windstar's own programme is Windstar's to publish, it is delivered to the suite nightly, and a copy here can only go stale or contradict it. The 15th keeps the Blue Church as a **landmark on a walk**, which is a fact about the town and stays true whatever the ship schedules.

Also worth knowing: the PDF's day headers overstate their own length (11 and 14 August both say "12 Hours" but run 8h30 and 8h05). The site shows actual spans instead.

**Ship port times were reconciled against Windstar's own itinerary page** (linked from the floating menu; it was in the hero until §14) and three were wrong:

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

**The dev panel has been removed again.** It is date simulation (presets plus a date picker), a live state readout, and a jump to the marked day. It was restored on the `itinerary-flat` branch to test the flattened itinerary across dates, then stripped before that branch merged.

> Removing it is **not** as clean as this section once claimed. A bare `devState();` call sits on its own line after `renderAll()`, *outside* every marked block. Deleting only the fenced blocks leaves it behind to throw a `ReferenceError` on load, which is what happened the first time. It now carries its own `DEV PANEL` comment so a grep finds it. If you strip anything else in this file, grep for callers rather than trusting the fences.

All the dev CSS is scoped under `#dev`, which matters now: the expand button also uses a class called `.tog`, and `#dev .tog` is a different thing entirely. Keep both scoped.

`let DEV_NOW = null, DEV_KEY = '';` and `NOW()` are deliberately still there. `NOW()` is the indirection every clock read goes through, and it now simply always returns `Date.now()`. Leave it: re-adding the panel means restoring the blocks, not rewiring the clock.

To bring it back, `git log` the file and revert the removal commit.

**The scheduling issues section has been deleted.** It held the two ship-versus-coach timing conflicts (12 August departing an hour before the ship docked, 14 August departing the exact minute it docked) and sat above the advisor card, which is now the natural closer as intended. Its `.flags` / `.flag` CSS and the `flagsHTML` reference in `renderAll()` went with it.

> The conflicts themselves were never resolved, only removed from the page. They are recorded in §12 so they are not lost.

The open-day callouts on the 10th, 15th and 16th were *not* part of that block and are still there. The distinction was deliberate: a day with nothing scheduled is useful context for the group; a timing conflict is a to-do for the trip owner.

---

## 12. Known gaps

- **One unresolved scheduling conflict, no longer shown on the site** (see §11). *14 August:* departure from Seyðisfjörður is booked for 9:30 am, the exact minute the ship docks, and gangway clearance normally takes ten to fifteen minutes. The 12 August conflict turned out to be an artefact of a wrong arrival time and is resolved (see §10).
- **A pill tapped while a panel is still folding lands about a screen short.** `goTo` measures its target from the current layout, and a fold in flight is about to take up to a screenful out of the document, so the measurement is stale by that much before the scroll finishes. Reproduced at ~960px off. It predates the simultaneous-scroll work and is not caused by it — the same tap on the older sequential version missed by ~997px, just by a different route, with the deferred collapse scroll overriding the pill outright. Only reachable inside the ~400 ms a panel is closing. The fix is to land any in-flight fold before measuring, which means giving `slidePanel` a settle function that can be called from outside and keeping `goTo` from finishing the very fold that called it — more surgery than the symptom has earned so far.
- No offline map of the ports beyond the hero SVG.
- 17 August arrival time unverified; Windstar does not list the disembarkation morning.
- Photo aspect is **1.60:1 against the card's 1.90:1**, so `slice` trims about 8% off the top and bottom of every one. Rendering the set at 1.90:1 would recover it.
- Fonts and the photos now stand between this and offline self-containment. Fonts were the last external request; the photos are local but still separate files.
- The phrasebook does not lock the page behind it. Its own list uses `overscroll-behavior: contain`, so a touch scroll stops at the ends instead of chaining, but a wheel over the backdrop still moves the page. A real lock costs either a scrollbar-width jump on desktop or the position-fixed dance iOS needs, and neither is worth it for a sheet this short-lived.

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

---

## 14. The floating menu

A hamburger in the hero's top right opening a three-row panel: **Common Icelandic Phrases**, which raises a phrasebook sheet, and outbound links to **Windstar's published itinerary** and the **operator's excursion PDF**. Added on the `floating-menu` branch.

### It sticks to the hero and nowhere else, with no scroll handler

`.m-rail` is `position:absolute; inset:0` on the header, so it is the hero's own bounds expressed as a box. `.m-dock` inside it is `position:sticky; top:14px`, and **a sticky box cannot leave its containing block**. That single fact is the entire feature: the button pins 14px under the top edge for as long as any of the hero is on screen, and then its bottom edge rides the hero's bottom edge out of view. Measured, `button.bottom === hero.bottom` for the whole exit, which is why it never crosses the day pills.

There is no scroll listener doing this and no class being toggled. Do not add one. The two numbers that matter:

- **`margin-top:36px`** on the dock is the button's resting position, and it is not decoration. A 44px button centred against the 31px wordmark opposite it wants its top at 36px, so at the top of the page the two read as one header row. Sticky then lifts it to 14px as you scroll, which is the whole "floating" part.
- **58px** in the scroll handler is `14 + 44`, the exact rect at which the dock un-pins. The panel closes there so it retracts into its button rather than sliding up over the nav.

The rail is `pointer-events:none` with `auto` restored on the button and the panel, because it covers the entire hero and would otherwise eat the clicks on the Windstar link underneath it.

### Motion

Rules taken from **Emil Kowalski's animation skills** (`github.com/emilkowalski/skills`) and applied by hand. Nothing is downloaded; the page is still one file.

- **`ease-out` on enter and on exit. Never `ease-in` on UI.** `ease-in` delays the first frame, which is the one the eye is on. `--e-morph` is the only exception and is used once, on the bars, because they morph in place rather than arriving or leaving.
- The three curves live in `:root` as `--e-out`, `--e-morph` and `--e-drawer`. They are custom because the CSS built-ins are too weak to read as deliberate at these durations.
- **Under 300ms for everything at this size.** Panel 190ms, items 160ms, bars 240ms, press feedback 160ms. The phone drawer is the one thing allowed longer at 380ms, which is inside the 200 to 500ms a drawer gets.
- **`transform` and `opacity` only**, so nothing touches layout or paint.
- The panel scales from `transform-origin:100% 0`, the button it came out of, **not** from its own centre. The phrasebook is exempt: it is a modal, it arrives centred, and it keeps a centred origin.
- Nothing starts at `scale(0)`. The panel opens from `.96`.
- **Transitions, not keyframes.** A fast open-close-open retargets mid-flight instead of restarting from zero.
- The row stagger is 30ms and lives **in the open state only**, so opening deals the rows out and closing takes the panel back as one piece.
- Hover lift is behind `@media (hover:hover) and (pointer:fine)`, or a tap leaves the row lit.

Under `prefers-reduced-motion` the page's existing blanket rule already cuts every transition to .01ms, which would turn these into snaps rather than moves. `.m-panel`, `.m-item` and `.sheet` therefore drop their transforms entirely and appear in place. The bars still cross into an X: that one is state, not decoration.

### The phrasebook is a real `<dialog>`

`showModal()` puts it in the **top layer**, which settles the stacking against the sticky nav at `z-index:60`, the back-to-top button at 70 and the menu rail at 95 without any of them needing to know about each other. Focus containment and the Escape key come with it. Three consequences:

- **The `cancel` event is intercepted.** Escape would otherwise close it instantly and skip the animation.
- **Opening needs two frames.** A dialog is `display:none` until it opens, and you cannot transition out of that. The first `requestAnimationFrame` commits the closed styles now that display is live, the second flips the class so there is a from-state.
- **Closing is on a timer, not `transitionend`,** and the timer length is read from the media query because the phone drawer is 380ms against the sheet's 240ms. `pbOpen()` clears a close still in flight, so tapping through does not strand it.

Below 560px it becomes a bottom drawer: full width, `margin:auto auto 0`, sliding `translateY(100%)` on `--e-drawer`. The opacity fade is dropped there because a drawer that also ghosts reads as two effects.

### Content

Phrase transcriptions use **the same system as the place names on the day cards**: capitals carry the stress, `TH` is the soft one, and `ll` comes out `tl` the way it does in `KIRK-yu-fetl`. Keep it consistent if you add rows; the group will be reading both on the same page.

Group labels are `--ink-3`, and the sounding-out line is `--glacier`. **Neither is ember, deliberately** (see §9). Ember is *now* and the hero stop, and a phrasebook is neither.

**There is no rule under the sheet header, and `.ph-g:first-child` carries a 38px top margin instead.** With the rule in place there were four horizontal lines inside about 60px of the sheet's top corner: the standfirst, the rule, the group label's own row, and the first phrase's `border-top`. It read as ruled paper. Whitespace separates the header now, which is why that first margin is nearly four times the one between groups. Do not put the rule back without taking the margin out with it.

### Two things that will trip you up

- **The menu is now the only route to Windstar's itinerary.** It was in the hero as well when this was written, and the two copies of the URL had to be changed together; the hero link has since been cut (§9), so `WINDSTAR` in the menu block is the single copy. It is a literal rather than something read back out of the DOM, which is what made cutting that hero line safe.
- **The COMPLETE stamp moved to `top:112px`** because the button took its corner. Not 80px, the button's bottom edge: the stamp is rotated 13 degrees so its box reaches about 19px above whatever top it is given, and its type is clamped against `vw` so that overhang is not the same at every width. It now lands across the title block, which is where a stamp belongs anyway.

### Removing it

The whole feature is four pieces and nothing outside them refers in: the `FLOATING MENU` and `PHRASEBOOK` CSS sections, the `.m-rail` block inside the header, the `<dialog class="sheet">` before the script, and the `FLOATING MENU` script block. The script block is deliberately self-contained and registers its own scroll listener rather than joining the one above it, so it lifts out in one cut. Also drop the three `--e-*` curves from `:root` and the `header.hero.done .stamp` and reduced-motion lines, which are the only edits it made to existing rules. Unlike §11's dev panel, there is no stray caller.

---

## 15. The itinerary as a sheet (experiment, branch `details-sheet`)

**Unmerged.** An alternative to §9's expand/collapse: the card's `<details>` panel is replaced by a **Details ＋** button that raises the day's timeline in the phrasebook's sheet. Written to be looked at and then either merged or deleted whole.

### What changed

- `dayHTML` emits a `<button class="det">` instead of `<details>`. The row keeps its shape exactly: two lines, control beside the label, preview beneath, ink not glacier. Only the mechanism moved.
- **The whole row is still the hit area**, which §9 says explicitly and which nearly got lost. `.det` is the `<button>` and `.tog` is now a `<span>` inside it, mirroring the old `<summary>` wrapping a `.tog` span. A nested `<button>` would have been invalid, and pill-only would have shrunk a 72px target to about 30.
- `tlHTML(d)` renders the timeline. The old CSS is untouched: `--tl-bg` still resolves to `var(--card)` because the sheet is the same surface.
- One `<dialog>` serves both callers, filled per open. `openSheet({title, sub, html, from})`.

### Two things that had to be got right

**`--gut` on `.sheet-bd` must equal that box's own horizontal padding.** The stop of the day widens itself by `--gut` and pays it back as padding so its wash reaches the edges of whatever it sits on (§9). Set it wrong and either the wash stops short or the row pushes past the edge, and because `overflow-y:auto` computes `overflow-x` to `auto`, one pixel too wide puts a sideways scrollbar in the sheet. Measured, the hero stop lands at 0px from both sheet edges.

**The start rail is no longer suppressed on pickup days.** Inside the card that rule stopped the departure time being printed twice, once in the pickup tile and once at the head of the timeline (§3). The tile stays on the card and does not come into the sheet, so there is nothing left to duplicate, and leaving the rule in opened every pickup day on a bare drive leg: "45 min · 45 km" as the first line, a journey from nowhere.

> `sheetBody.scrollTop = 0` runs **after** `showModal()` and unconditionally. Before it the dialog is `display:none`, the scroller has no layout and the assignment is dropped; and skipping it when the sheet is already open drops you halfway down another day's timeline.

### What it would cost

Adopting this **deletes the work in `3b4028d`**. `slidePanel`, `glideTo` and `stopGlide` are all still in the file on this branch and none of them is reachable: the foot Collapse button was `glideTo`'s only caller, and there is no `<details>` left for `slidePanel` to drive. That is the fold-and-ride-back-up gesture, the rAF scroll that re-reads the reachable maximum every frame, and the reduced-motion double-toggle fix. They are left in place deliberately so the branch reverts cleanly and so the two can be compared side by side. **If this merges, strip them in the same commit** rather than leaving three unreachable functions and a dead `main` click handler behind.

The sheet also gives up two things the inline panel had: the timeline no longer sits in the reading flow of the day it belongs to, and the card behind it is hidden while it is open, so the ports bar and pickup tile cannot be read at the same time as the stops.

### Second pass on the sheet

**Every sheet is the same height.** `height`, not `max-height`. A `<dialog>` is `height:fit-content`, so the sheet used to be as tall as whatever was poured into it: the phrasebook always overflowed and sat at the cap while a three-stop day came out about half as tall. Same control, same surface, two sizes depending on which one you pressed. Short days now have room to spare at the foot, which is the right trade.

**The gap under the standfirst belongs to the header, not to the scroller's first child.** Both look identical at rest and only one survives a scroll: `.sheet-hd` is a separate flex item above `.sheet-bd`, so its `padding-bottom` is a permanent band that the list is clipped against, while padding on the first row scrolls away with that row and lets the next one arrive hard against the type. The two still add up to the ~38px the phrasebook was tuned to, and `.ph-g:first-child` and `.sheet-bd .tl` are both zero now so there is only one place to change it.

**The excursion middot line is fixed at guide then hours.** It used to be guide, span, distance, and 12 August broke the pattern by printing `4 hours` where every other day printed a range. It is `08:30 – 12:30` now, which is the same four hours stated the same way as its neighbours. **The day total distance is gone from the page** as a result; per-drive distances are still on every leg in the timeline.

`.meta.oneline` holds those five lines to one line and shrinks the type rather than wrapping, by the same method and with the same trap as the hero's middot row (§9): `86px` is 36 for `.wrap`'s padding, 34 for the body's two gutters and 16 for the one separator margin, none of which scale with the type; `21.4` is the longest line measured in em against the widest font in the stack **at the smallest size it can pick**, which is 14 August's two-guide line in San Francisco at 21.02em down at 10px against 20.02em at 14.5px. Figtree is 19.12, so sizing to Figtree would fit online and wrap offline.

> Only excursion days get it. 16 August runs to "6.5 hours ashore · Then two hours off Surtsey", which is a genuinely different shape and is better wrapping than squeezed; sizing every day's line to that one would have cost the other eight about 9% of their type on a phone for nothing.

> **The sheet takes the opening focus itself, and that is deliberate.** `showModal()` otherwise hands focus to the first focusable thing inside, which is the close button, and Safari counts programmatic focus as `:focus-visible`: on iPhone every tap on **Details** opened the sheet with a ring drawn around the X. The dialog carries `tabindex="-1"`, `openSheet` calls `sheet.focus({preventScroll:true})` explicitly rather than trusting the dialog focusing steps (which element those pick has moved between spec revisions and browsers), and `.sheet:focus` clears its own outline because a container is not a control. Tabbing still puts the ring on the close button, which was checked with a real key press rather than a scripted `.focus()` — the two do not behave the same and only the real one proves it.
