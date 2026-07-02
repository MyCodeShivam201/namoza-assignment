# Task 01 — GTM Event Schema · OrthoNow

**Prepared for:** Namoza × OrthoNow digital growth engagement
**Scope:** Full event tracking implementation prior to paid campaign launch

---

## 1. Complete GTM Event Schema

| Event Name | Trigger Type | Key Parameters (min. 3) | Feeds Into (GA4 Report / Audience) |
|---|---|---|---|
| `booking_step_complete` | Custom Event (dataLayer push, **not** a native GTM trigger — see §2) | `step_number`, `step_name`, `clinic_location` | Funnel Exploration (booking funnel), Engagement → Events |
| `booking_form_submitted` | Custom Event (dataLayer push on step 3 confirm) | `clinic_location`, `specialty`, `preferred_date` | Conversions report, Audience: "Completed Booking" |
| `call_now_click` | Click — All Elements (CSS class `.call-now-btn`) | `click_location` (homepage/clinic_page/landing_page), `phone_number`, `page_path` | Engagement → Events, Audience: "High-Intent — Call Click" |
| `whatsapp_chat_opened` | Click — Just Links (CSS class `.whatsapp-fab`, matches `wa.me/*`) | `click_location`, `page_path`, `clinic_context` (if on a clinic page) | Engagement → Events, Audience: "WhatsApp Engagers" |
| `patient_guide_form_view` | Element Visibility (gated form modal opens) | `gate_location`, `asset_name` | Engagement → Events |
| `patient_guide_download` | Custom Event (dataLayer push on gate-form submit success, fired *before* PDF serve) | `asset_name`, `gate_location`, `lead_source` | Conversions report (secondary conversion), Audience: "Guide Downloaders" |
| `clinic_page_view` | History Change / Page View (9x, parameterised — **not** 9 separate triggers, see note below) | `clinic_name`, `clinic_city`, `page_path` | Engagement → Pages and screens, Audience: "City-Intent — {city}" |
| `blog_scroll_25` | Scroll Depth (25%, restricted to `/blog/*`) | `article_title`, `article_category`, `scroll_percentage` | Engagement → Events |
| `blog_scroll_50` | Scroll Depth (50%, restricted to `/blog/*`) | `article_title`, `article_category`, `scroll_percentage` | Engagement → Events |
| `blog_scroll_90` | Scroll Depth (90%, restricted to `/blog/*`) — treated as "read" | `article_title`, `article_category`, `time_on_page` | Engagement → Events, Audience: "Engaged Readers" |

**Implementation note on `clinic_page_view`:** this should be **one** GTM trigger firing on a regex match (`^/clinics/.*`) with `clinic_name` and `clinic_city` pulled dynamically from a dataLayer variable set per-page (or scraped via a DOM-element variable if the WordPress template doesn't push it). Building 9 separate triggers is the kind of thing that looks thorough but is actually a maintenance liability — if a 10th clinic opens, the schema breaks instead of just working.

**Implementation note on Call Now buttons:** same logic — one trigger across all three placements (homepage, clinic pages, landing page), with `click_location` as a dataLayer-pushed or DOM-attribute-derived parameter distinguishing where the click happened. The location-page Call Now button and the landing-page Call Now button should *not* be the same conversion in Google Ads even though they share an event name, because intent differs — landing page clicks are campaign-driven, location page clicks are often local/organic.

---

## 2. The 3-Step Booking Form — Funnel Drop-Off Tracking

This is the part of the schema that actually requires engineering, not configuration.

**The core constraint:** GTM has no native trigger type that listens to "user is now on step 2 of a JS-rendered form." Form Submission triggers fire on a `<form>` tag's submit event — they don't know steps exist. Element Visibility triggers can detect a step *container* becoming visible, which is a usable proxy, but it's fragile if the front-end implementation toggles `display:none` vs. actually mounting/unmounting DOM nodes (Element Visibility won't reliably re-fire on a node that's hidden and re-shown depending on how the visibility tracking is configured).

The reliable approach is: **the front-end developer pushes to `dataLayer` explicitly at each step transition**, and GTM listens for those custom events. GTM is the *listener and router* here, not the *detector*. This has to be briefed to the dev team as an explicit implementation requirement, not assumed as something GTM "just handles."

### What fires at each step

**Step 1 — Location + Specialty selected**
Trigger: Custom Event `booking_step_complete` where `step_number == 1`
Front-end fires this the moment the user selects both a clinic location and specialty and the UI allows progression to step 2 (i.e., on the "Next" action, not on every dropdown change).

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**Step 2 — Contact details entered**
Trigger: Custom Event `booking_step_complete` where `step_number == 2`
Fires when the user completes name/phone/preferred date fields and clicks "Next." Phone number itself should **not** be pushed to dataLayer in plaintext (GTM containers and their data can leak into GA4/Ads exports; treat phone as PII and keep it server-side). Push a boolean or a hashed reference if cross-referencing is needed later.

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "preferred_date": "{{preferred date}}"
}
```

**Step 3 — Booking confirmed**
Trigger: Custom Event `booking_form_submitted` (distinct event name, not `booking_step_complete` with `step_number: 3` — this is the actual conversion and deserves its own event name so it's unambiguous in GA4/Ads, rather than relying on someone filtering `step_number == 3` correctly every time).

```json
{
  "event": "booking_form_submitted",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "preferred_date": "{{preferred date}}",
  "booking_id": "{{booking reference id}}"
}
```

### Who writes this push — the brief and the actual answer

**The front-end developer writes the dataLayer push, not GTM, and not me configuring tags.** GTM can only react to events that already exist in the dataLayer; it cannot inspect React/Vue component state, form-step index, or conditional rendering on its own. If the booking form is a single-page JS component (likely, given it's a 3-step flow with no apparent page reloads), then "step 2" is invisible to GTM unless someone explicitly tells the dataLayer "step 2 just happened."

**How I'd brief this to the dev team**, concretely:

1. Identify the exact UI event that constitutes "step complete" — almost always the click handler on the "Next" / "Continue" button, *after* that step's validation passes (not on every keystroke, not on render).
2. Give them the literal JSON shape above, with the actual variable names already matching dataLayer Variable names I'll configure in GTM (so there's no translation layer where "what the dev calls it" differs from "what GTM is listening for" — this mismatch is the #1 reason booking funnels silently break post-launch).
3. Specify that the push must happen *before* any route change or component unmount, since a dataLayer push followed immediately by a hard navigation can race-condition and get dropped before GTM's listener processes it.
4. Ask for a `step_number` that increments, not a boolean per step, so the funnel visualization in GA4 has a clean numeric sequence to step through.
5. Confirm with QA: open the browser console, watch `window.dataLayer`, walk the form, and visually confirm 3 distinct pushes in order before this ships to staging.

### Surfacing this in GA4 Funnel Exploration

In GA4 Explore → Funnel Exploration:

- **Step 1:** Event `booking_step_complete`, condition `step_number = 1`
- **Step 2:** Event `booking_step_complete`, condition `step_number = 2`
- **Step 3:** Event `booking_form_submitted`

Set the funnel to **open** (not strictly sequential closed funnel) initially, to see if people are re-entering at step 2 directly (which would indicate a bookmarking/back-button pattern worth knowing about). Once the baseline is established, switch to a closed funnel for a cleaner drop-off percentage between adjacent steps. The step-level abandonment rate (GA4 shows this natively as a % between each step) is the number the performance marketing team should be watching weekly — if step 1→2 drop-off is unusually high, that's a UX/form-length problem; if 2→3 drop-off is high, that's more often a trust/hesitation problem (people get cold feet right before confirming).

---

## 3. Conversion Action to Import into Google Ads

**Import: `booking_form_submitted`** (the step 3 / final confirmation event), not `consultation_form_submitted` from the landing page, and not `patient_guide_download`.

**Why this one over the others:**

`patient_guide_download` is a softer signal — gated content downloads correlate with research-stage intent, not booking-stage intent. Optimising Google Ads bidding toward this would teach Smart Bidding to chase people who want a PDF, which is a meaningfully different audience from people who want an appointment. It's useful as a *secondary* conversion (marked "secondary" in Ads, included for visibility, excluded from bidding optimisation), not the primary one.

`call_now_click` looks tempting because phone calls in healthcare are high-intent, but it's a noisier signal in this specific setup: it fires identically whether the click is from a campaign visitor on the landing page or from someone browsing a clinic location page organically (different funnel stage, different cost-efficiency expectation), and it doesn't confirm an actual booking happened — someone can click-to-call and not pick up, or call about something unrelated to booking.

`booking_form_submitted` is the cleanest signal because it represents the actual business outcome — a completed appointment booking with location, specialty, and date attached — and it's specific to the booking flow rather than shared across multiple page types. Importing this lets Smart Bidding optimise toward people who demonstrably complete the full 3-step intent sequence, which is the closest proxy available to "this ad spend produced a real patient," and it's the one event in this schema where false-positive risk (the event firing without a genuine booking) is lowest, since it only fires after all three steps validate and submit successfully.
