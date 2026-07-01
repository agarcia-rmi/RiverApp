# River Church App — Developer Integration Brief

**Purpose:** Hand-off reference for the developer building the production River Church / RMI member app. The current deliverable is a clickable single-file HTML *prototype* (design + flows only). This brief covers what production needs: the MinistryPlatform (MP) data connections, push notifications, live streaming / picture-in-picture, access control, and the known challenges.

**Important framing:** The phone app must **never** talk to MinistryPlatform directly. A small **backend / middleware service** sits between the app and MP. It holds the MP API credentials, enforces "each member sees only their own data," shapes responses, caches, and triggers push notifications. Most of the real work is in that backend, not the UI.

> Note: Output/preview files from the prototype are a *design reference*, not production code. Field and table names below reflect the church's stated MP configuration and should be re-confirmed with the MP administrator before build.

---

## 1. Push notifications — how they work

MinistryPlatform does **not** send native push notifications itself. You build it with a standard mobile push stack plus a backend trigger.

**The stack (recommended: native or Capacitor wrapper):**
- **Android:** Firebase Cloud Messaging (FCM)
- **iOS:** Apple Push Notification service (APNs)
- **Unified option:** Firebase Cloud Messaging or a service like **OneSignal** / **Expo Push** that handles both platforms with one API. OneSignal is the fastest path for a church app (built-in segmentation, scheduling, no server to maintain for sending).

**The flow:**
1. App asks the user for notification permission (required, especially on iOS).
2. On allow, the OS issues a **device token**.
3. App sends that token to the backend.
4. Backend stores **device token ↔ MP Contact_ID** (so notifications can be targeted to the right person/segment).
5. When something happens — new announcement, event reminder, prayer-team reply, "doors open" status — the backend (on a schedule, on a webhook, or on an admin action) calls FCM/APNs/OneSignal to deliver the push to the matching tokens.

**Targeting / segments** can be driven from MP data: by membership status, by Group, by Congregation/campus, by event registrants, etc.

**Tier-specific (member-only) notifications.** Some prompts must go to a filtered segment, not everyone. Example: the **"new Badge background"** suggestion is a **Members-only** prompt — build the recipient segment from `Member_Status` (members only) so Guests and Attendees never receive it. Implement it as a segment/tag in the push service (e.g., a OneSignal "Members" tag synced from MP, or a backend query that selects only member device tokens before sending). Belt-and-suspenders: the in-app surface is also gated to Members (see Section 3), so even a mis-targeted send can't expose a member-only feature to a non-member.

> iOS caveat: reliable push really wants a native or Capacitor build. Web-only (PWA) push is limited on iOS.

---

## 2. MinistryPlatform data connections

Each app feature maps to MP data through the backend. Indicative mapping (confirm exact tables/license with the MP admin):

| App feature | MP source (indicative) |
|---|---|
| Sign-in / identity | `Contacts`, `Users` |
| My Household | `Contacts`, `Households` |
| Check-in QR / badge | `Contacts.[ID_Card]` (see field mappings) |
| Membership status & "Member since" | `Participant_Record` (see field mappings) |
| Events list & registration | `Events`, `Event_Participants` |
| Requests (prayer, counseling, etc.) | `Forms` / `Form_Responses` or `Feedback_Entries` |
| Groups & group membership | `Groups`, `Group_Participants` |
| Giving records | Pushpay (giving stays in Pushpay; link/deep-link out) |

### Confirmed field mappings
- **Check-in QR code** → `Contacts.[ID_Card]`. Used on **both** the Member badge and the **Guest check-in** badge.
- **Member Status** → `Participant_Record_Table_Member_Status_ID_Table.[Member_Status]`
- **Member Since (Status Date)** → `Participant_Record_Table.[Status_Date]`

> The long `_Table_..._ID_Table` alias names suggest these may come from a **view or stored procedure** rather than base MP tables — confirm the exact source object with the MP admin so the backend queries the right thing.

---

## 3. Membership tiers & access control

The app has three membership types: **Guest**, **Attendee**, **Member**. The backend must enforce what each tier can see and do (the UI hides/shows, but the backend is the source of truth).

**Tier-specific behavior in the current prototype:**
- **Guest:** no Groups tab — replaced by a **Check-in** tab that opens a full-screen badge (QR from `Contacts.[ID_Card]`). "Get involved" surface (River University, Become a River Member). Optional River Church / RMI brand toggle.
- **Attendee:** "Your next step" surface — Join a Group (→ Groups tab), Membership Class (→ Membership page), Become a River Member (→ Membership page).
- **Member:** full feature set, including badge personalization.

**Access rule to enforce — Badge Backgrounds are Member-only.**
The badge-background personalization (the swatch picker on the ID/Badge screen) must be shown and offered **only to Members**. Do **not** surface it to Guests or Attendees. The prototype already gates this on member status; production should enforce the same rule server-side so the option isn't returned for non-members. **This extends to notifications:** any prompt suggesting a badge background must be sent to the **Members-only** segment (see Section 1) so Guests and Attendees never receive it.

---

## 4. Live streaming & Picture-in-Picture (Stream button)

**Current prototype behavior:** the **Stream** quick-action opens the Revival TV stream (revivaltv.com) inside the app's **in-app browser overlay**, which is a cross-origin **iframe**.

**Core constraint:** a page cannot reach into a cross-origin iframe and control the `<video>` inside it (browser security). So the app **cannot programmatically trigger picture-in-picture** on the Revival TV stream as currently wired. Also note: standard PiP is an **OS-level floating window that floats *outside* the app** — it is not, by definition, "inside" the app. Two distinct things are worth separating:

**Option A — Permit the embedded player's own PiP (small, quick).**
Add `allow="picture-in-picture; autoplay; encrypted-media; fullscreen"` to the in-app browser iframe. This lets the **viewer** trigger PiP from the embedded player's own controls (or the OS) when the source player supports it. The app still can't press the button for them, but it stops the browser from blocking PiP. Result: a floating window managed by the OS.

**Option B — True in-app floating mini-player (bigger; needs a controllable source).**
To keep the stream playing in a small window *inside the app* while the user navigates other tabs (Events, Groups, etc.), the app must **own the video element**. That requires an embeddable/controllable source instead of the website link:
- a direct **HLS** stream URL (`.m3u8`), or
- a **YouTube Live** or **Vimeo** embed driven via their player/IFrame API.

With one of those, build a real player the app controls, then either call `video.requestPictureInPicture()` (OS PiP) and/or implement a **custom persistent mini-player** component that survives tab changes (a floating, draggable video pinned above the tab bar).

**Platform notes:**
- Video PiP (`requestPictureInPicture`) is supported on **iOS Safari** and **Android Chrome**.
- **Document Picture-in-Picture** (popping arbitrary UI out) is **Android / desktop Chrome only — not iOS**.
- An installed iOS PWA has **limited PiP / background-audio** behavior; a **native or Capacitor wrapper** gives the most reliable PiP and background playback.

**Decisions needed before build:**
1. What is the actual stream source? (YouTube Live / Vimeo / direct HLS feed.)
2. Do you want **OS-level PiP** (floats outside the app) or an **in-app persistent mini-player** (stays inside the app across tabs)?

**Recommendation:** ship Option A immediately (one-line permission, zero source change). If the stream can be exposed as a YouTube Live/Vimeo embed or HLS URL, add Option B's in-app mini-player for the "keep watching while you browse" experience.

---

## 5. Sermon Notes / Journal storage

Confirmed decision: store sermon notes / journal entries **on-device**, with a **mandatory user warning** that notes can be lost if the app is deleted or the device is switched (they do not sync). The prototype implements local-device storage (45-day retention) plus a "Save to Notes" hand-off to the phone's native Notes app via the share sheet.

If a synced experience is later desired, store entries in the **app's own backend database** keyed to the member (with on-device caching for offline) — **not** in MinistryPlatform. Personal devotional/journal content is high-volume and personal and doesn't belong in the church CRM unless the church specifically wants it there.

---

## 6. Testimony moderation & publishing

Testimonies shown in the Requests tab go through an **approval gate** so the church controls exactly what appears. Nothing is public until a leader approves it.

**Flow:**
1. A member submits a testimony in the app (Requests → Testimony).
2. It lands in the backend as **pending / unpublished**, stored in MP (a "Testimonies" form / `Form_Responses`, a `Feedback_Entries` record, or a small custom table). Fields: status (Pending / Approved / Rejected), an `Approved` boolean, optional publish-window dates, optional `Featured` flag.
3. Pastors/admins moderate in an MP **Page/View filtered to "pending"** (or a simple admin screen): approve, edit text, set author display (real name vs. anonymous), set a publish window (start/end), set ordering/feature, or reject/remove.
4. The app feed returns **only Approved** testimonies within the publish window, e.g. `WHERE Approved = 1 AND now >= PublishStart AND (PublishEnd IS NULL OR now <= PublishEnd)`, ordered by `Featured DESC, Date DESC`.

**Important — the app must NOT display moderation state.** Approval status, "Featured," and any other admin metadata are **backend-only** and must never be shown to end users (no badge, label, or indicator). "Featured" affects ordering only, with no visible tag. The app simply shows approved testimonies as normal content.

**Two separate layers to honor:**
- **Submitter share level** (their privacy choice — "Leaders only" / anonymous, already in the Requests screen). If they chose anonymous, anonymize the displayed name even after approval.
- **Church approval gate** (editorial control). Both must pass before a testimony is shown publicly.

Optional add-ons: profanity/safety auto-flagging for review, expiration, a "report" button, and submission rate limiting.

---

## 7. Visual theming & liquid-glass rendering (per platform)

The app ships light / dark / and a **liquid-glass** theme. The glass look is built from two layers:
1. **Standard `backdrop-filter`** (blur + saturate + brightness), a diagonal sheen gradient, a bright inset rim, and inner depth shadows. This is the bulk of the effect.
2. **An SVG edge-refraction filter** (`feTurbulence` + `feDisplacementMap` applied via `backdrop-filter: url(#glassEdge)` on a masked edge ring) that bends the content at the border for a true refraction cue.

**The key constraint is the rendering engine, not the browser brand:**
- Layer 1 (`backdrop-filter: blur()/saturate()/brightness()`) works on **both WebKit and Chromium** — so it renders everywhere, including iPhone.
- Layer 2 (the SVG `url(#…)` displacement) is **Chromium/Blink only**; **WebKit does not support it**.

**What that means for the production app:**
- **iOS (any web-based build):** Apple requires **all** web content on iOS to use **WebKit** — Safari, a PWA added to the Home Screen, and the `WKWebView` inside any Capacitor/Cordova wrapper. Wrapping the web app as a native shell does **not** change this. So on iPhone the **edge refraction will not render**; the graceful fallback (blur, sheen, rim, glassy buttons) still does. Plan for the fallback look on iOS.
- **Android (PWA or Capacitor):** the system WebView is **Chromium**, so the **full effect including refraction renders**.
- **Fully native build (SwiftUI / React Native):** implement glass with the **native iOS material** (`UIBlurEffect` / SwiftUI `.ultraThinMaterial`, or Apple's newer system "Liquid Glass" material), which does real-time blur and refraction more convincingly than any web technique. In this path the CSS prototype is the **design target**, and the native iOS result can match or exceed it.

**Guidance:** treat the SVG refraction as **progressive enhancement** — a bonus where the engine supports it (Chromium/Android), never a dependency for the iOS look. For a premium glass look on iPhone, go native and use Apple's material. The glassy **Register buttons** use standard `backdrop-filter` only, so they render on all engines.

**Performance:** `backdrop-filter` is GPU-intensive. Limit the number of simultaneous glass surfaces on screen (especially on lower-end Android), and avoid stacking glass-on-glass where possible.

---

## 8. Known challenges / risks

- **Per-user data isolation** is the hardest part: every backend response must be scoped to the authenticated member's own records.
- **Exact MP table/field names depend on the church's license & configuration** — confirm with the MP admin / Think Ministry (especially the `Participant_Record` aliases, which may be view/stored-procedure-derived).
- **iOS push & PiP** are more constrained than Android — plan for a native/Capacitor build for reliable behavior.
- **Cross-origin embeds** (stream, Pushpay, external sites) can't be controlled by the app and some sites block being framed entirely — provide an "open externally" fallback.
- **Giving stays in Pushpay** — deep-link out rather than re-implementing payments.

---

## 9. Suggested next step

A short scoping call with the **MP administrator / Think Ministry** to confirm: REST API access, OAuth client setup, the Check-In token format (`Contacts.[ID_Card]`), the source object behind the `Participant_Record` membership fields, and which existing tables/forms map to prayer requests, membership status, and applications. In parallel, confirm the **stream source** (Section 4) so the Stream/PiP path can be locked in. That unblocks almost everything above.
