# Iceland 2026

A single-page microsite for a private Iceland trip, 9–17 August 2026: a Windstar *Star Pride* round-trip from Reykjavík wrapped around five private land days, plus the total solar eclipse on 12 August.

Everything lives in **`iceland-2026.html`** — markup, styles and script in one file, no build step and no dependencies. The only other runtime assets are the nine painted day images in `assets/images/`.

## Running it

Any static server from this directory:

```bash
python3 -m http.server 4321
```

Then open `http://localhost:4321/iceland-2026.html`.

Opening the file directly with `file://` works too, though some browsers block the images that way. It cannot be moved on its own — the `assets` folder has to travel with it.

## Read this before editing

**[`NOTES.md`](NOTES.md)** is the working documentation: the data model, the constraints, and the reasoning behind decisions that look arbitrary until you know why. Most of it was written after something broke. Worth reading first.

A few things that catch people out:

- **No hash links.** In-page jumps are buttons plus `scrollTo()`; `href="#id"` tries to leave the page.
- **Times are stored in 24h and displayed in 12h.** `h12()` converts at render, so the data still reconciles against the operator's paperwork. Prose inside `desc` and `note` is not converted and must be typed in 12h by hand.
- **Check new artwork at phone width.** The caption covers the bottom ~47% of a card's image on a 375px screen against ~24% on a desktop.
- **Gradient ids are suffixed with the day index.** Several scenes render at once and duplicate ids cross-contaminate.

## Assets

The nine day images are WebP at 1200px wide, which is a little over 2× the 584px they ever display at. The full-resolution PNG masters are **not in this repo** (see `.gitignore`) — they live outside it and need their own backup. To re-encode one:

```bash
cwebp -q 82 -resize 1200 0 archive/name.png -o name.webp
```

## Before sharing

`NOTES.md` §11 lists the removable blocks. The **dev panel** — the date simulator in the bottom-left corner — is still in the file and should come out before this goes to the group.

## Accuracy

Ship port times match [Windstar's published itinerary](https://www.windstarcruises.com/tour-details/REYREY7D9/n-europe/reykjavik-to-reykjavik/7-day-around-iceland-a-total-solar-eclipse/?pkgid=1013304), linked from the page. The 17 August arrival is the one time not listed there and remains unconfirmed. Land-day times come from the operator's sheet and were reconciled line by line; three errors in that source are corrected on the site and documented in `NOTES.md` §10.
