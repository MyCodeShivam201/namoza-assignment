# OrthoNow Digital Growth — Developer Assignment

**Candidate submission for:** Namoza Developer Assignment (Position 1 — Client Web + Martech)
**Client scenario:** OrthoNow, 9-clinic orthopaedic chain across Bengaluru, Hyderabad, Chennai

---

## Repo structure

```
├── task1/
│   └── gtm-event-schema.md       — Full GTM event schema, booking funnel dataLayer spec, Ads conversion choice
├── task2/
│   └── index.html                — Single-file landing page (HTML/CSS/JS, no frameworks)
├── task3/
│   └── integration-architecture.md — HubSpot + WhatsApp + Google Ads integration design (300–400 words)
└── README.md                     — This file
```

## Task 2 — running the page locally

This is a single self-contained HTML file with no build step and no server dependency. Open `task2/index.html` directly in a browser, or serve it statically:

```bash
cd task2
python3 -m http.server 8000
# then visit http://localhost:8000
```

To see the GTM dataLayer push fire live: open DevTools console, fill the 2-field form (name + phone), submit, and watch for the `[OrthoNow] GTM dataLayer push fired:` log line showing the `consultation_form_submitted` event object.

## PageSpeed Insights

**Not yet captured in this submission.** I built the page deliberately to the constraints that drive a high mobile score — zero raster images (every icon is inline SVG), zero render-blocking scripts, CSS inlined in `<head>`, only one external dependency (Google Fonts, loaded with `preconnect` hints) — but I don't have a way to run an actual PageSpeed Insights audit or capture a real screenshot from the environment I built this in. Before final submission, deploy this file anywhere static (GitHub Pages, Netlify drop) and run it through https://pagespeed.web.dev — if the font loading is the one thing keeping it under 90, the fix is self-hosting Sora/Inter as WOFF2 and dropping the Google Fonts request entirely, since that's the only external network call the page makes.

## Notes on what's intentionally not built

- **Task 1** is a written deliverable per the brief, no code.
- **Task 2's HubSpot/Karix/Google Ads server-side calls described in Task 3 are not implemented in `index.html`** — the brief's Task 2 scope is the front-end dataLayer push only; the actual CRM/WhatsApp wiring is correctly scoped to Task 3 as an architecture answer, not a working integration, since standing up live HubSpot and Karix accounts wasn't part of the ask.
- **GTM container itself is not provisioned** — Task 1 is the schema that would be built *into* a GTM container; the brief asks for the schema as a deliverable, not a live GTM workspace export.
