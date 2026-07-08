# Background Work API

## Introduction

Browsers intervene aggressively on hidden pages to save battery, CPU, and
memory: timers are throttled, JavaScript can be paused entirely by [Page
Lifecycle freezing](https://wicg.github.io/page-lifecycle/), and tabs may be
discarded under memory pressure. This is the right default, but it silently
breaks legitimate, user-initiated tasks.

The canonical example is a large photo upload: a user selects 500 photos to
back up, switches tabs, and returns to find the upload stalled or lost. From
the user's perspective, they asked the site to do something and the browser
quietly stopped it. Sites work around this today with silent audio, dummy
WebRTC connections, or "don't switch tabs!" banners — hacks that are opaque to
users and worse for battery than an honest mechanism.

The **Background Work API** proposes a user-granted **permission**, analogous
to camera or microphone access, that lets an application run its background
code without fear of it being stopped, and without resorting to hacks to keep
the tab alive. While the permission is granted, the JavaScript the application
needs to execute keeps running even if the browser freezes the tab: the page
itself may still be frozen to reclaim resources, but the work carries on. The
user is prompted in context, and can revoke the grant at any time from site
settings. In exchange, the browser gets an explicit, attributable signal about
which sites do background work, instead of guessing or being gamed.

## Goals

- Let web applications **reliably complete user-initiated work** while
  backgrounded (uploads, syncs, long-running workflows).
- Keep the **user in control**: explicit, revocable consent, requested with
  context.
- Replace today's workarounds with an **honest signal** the browser can
  attribute and surface in UI.
- Support **long-lived applications** (e.g. a call-center console) whose need
  for background execution is persistent, not tied to one short task.

## Non-goals

- **Running code after the tab is closed** — that's the domain of
  service-worker APIs like Background Fetch and Background Sync.
- **Keeping the screen or device awake** (Screen Wake Lock's job).
- **Unthrottled rendering** — hidden pages still don't paint;
  `requestAnimationFrame` stays paused.
- **An absolute guarantee against every intervention.** The exemption is a
  strong, signaled commitment, not immunity — see
  [Exactly what is exempted?](#exactly-what-is-exempted) for the proposed
  positions on discarding and exceptional conditions.

## Motivating use cases

1. **Large media uploads (primary).** Uploading hundreds of photos involves
   live application logic — resizing/transcoding in workers, chunk hashing for
   resumable uploads, retry timers — not just network transfer. Throttling
   crawls it, freezing stops it, discarding loses it.
2. **Long-lived line-of-business apps.** A call-center agent keeps a web
   softphone/CRM open all shift; its queues, heartbeats, and connections must
   stay serviced while the agent works in other windows. A time-boxed exemption
   would be unusable here, which is why the grant is persistent.
3. **Finishing what the user started.** Flushing unsaved edits, completing a
   client-side export or encode, syncing local changes.

## Proposed solution

A new permission, tentatively `"background-work"`, integrated with the
[Permissions API](https://www.w3.org/TR/permissions/) and [Permissions
Policy](https://www.w3.org/TR/permissions-policy/).

### Requesting it

Requested imperatively (like `getUserMedia`) so the site can ask in context,
ideally when the user starts the relevant task. UAs may require transient user
activation.

```js
uploadButton.addEventListener("click", async () => {
  startUpload(files);

  const state = await navigator.backgroundWork.request(); // "granted" | "denied"
  if (state !== "granted") {
    showBanner("Keep this tab open in the foreground so your upload can finish.");
  }
});
```

### Observing it

Standard Permissions API integration, so apps can adapt when the user revokes:

```js
const status = await navigator.permissions.query({ name: "background-work" });
status.onchange = () => {
  if (status.state !== "granted") checkpointUploadState();
};
```

### What a grant means

While granted and the page is hidden:

- **Freezing does not stop the work.** The UA may still freeze the page itself
  — pausing its main thread and reclaiming rendering resources — but the page's
  workers keep executing. Messages posted to the frozen page queue up and are
  delivered when it resumes.
- **Timers are not throttled** while the page is unfrozen (no 1-minute clamping
  or intensive-throttling budgets).
- **Workers keep running** at normal priority, frozen page or not — so
  background work should live in a dedicated or shared worker.

The grant is **persistent and indefinite**, like notifications: it applies to
the origin until revoked, rather than being scoped to a task or time window.

### Permissions Policy and UA UI

A `background-work` policy-controlled feature (default `'self'`) gates use in
cross-origin iframes unless the embedder delegates it via `allow`. UAs are
encouraged to show an indicator on tabs doing exempt background work, a
revocation toggle in site settings, and attribution in task-manager UIs.

## Detailed design discussion

### Why a permission rather than heuristics or a developer-only API?

Browsers already exempt pages from throttling via heuristics (audio, WebRTC,
etc.). Heuristics fail in both directions: legitimate apps that match none are
stuck, and others game them with silent audio. A permission makes the contract
explicit — the site asks, the user decides, the browser attributes the cost —
and matches native platforms, where background activity is a per-app,
user-visible setting.

### Why indefinite?

A time-boxed grant fails the long-lived app: a console open for an eight-hour
shift can't re-request every few minutes, and repeated prompts train users to
click through. Indefinite-until-revoked mirrors notifications and keeps the
model simple: *this site may work in the background*. UAs can still apply
safeguards — surfacing heavy background CPU use, or auto-revoking grants for
long-unvisited sites, as some do for notifications. A future extension could
add an activity signal (e.g. `setActive(true)`) so UAs can throttle granted
sites that are idle; it's left out of the minimal proposal.

### Exactly what is exempted?

Rather than exempting the whole page from freezing, the grant guarantees that
**the work continues even if the page freezes**. Freezing exists to reclaim
what a hidden page holds — main-thread scheduling and the rendering pipeline.
Keeping all of that live just so an upload loop can tick would give up most of
the benefit. Splitting the model lets the UA reclaim everything the work
doesn't need:

- While the page is hidden but **unfrozen**, its main-thread timers and
  scheduling run unthrottled.
- If the UA **freezes** the page, its dedicated and shared workers continue at
  normal priority, with full access to `fetch()`, WebSockets, IndexedDB, and
  the rest of the worker platform. `postMessage` to the frozen page queues
  until resume, and the Page Lifecycle `freeze` event gives the main thread a
  last chance to hand remaining work to a worker.

The developer guidance this implies is the same as for performance in general:
put the heavy lifting in a worker. Pages that keep all their logic on the main
thread still benefit from unthrottled timers, but must be prepared to be
frozen. Rendering is never exempted, and visibility semantics don't change:
`document.hidden` and `visibilitychange` behave as today.

**Discarding.** We propose that a grant **strongly deprioritizes** the page in
tab-discard heuristics rather than making it undiscardable. Discarding is
driven by memory pressure; a hard guarantee could force the UA to sacrifice
foreground responsiveness for a background tab, and cannot be honored against
OS-level process kills on mobile anyway. Deprioritization means that in
practice a granted page doing real work is among the last to go.

**Exceptional conditions.** The grant is a strong commitment, not an absolute
one: UAs **may override it** under critical battery, thermal pressure, or
enterprise policy, and **must signal** the intervention to the page — via the
Page Lifecycle `freeze` event and/or `PermissionStatus.onchange` — so it can
checkpoint state. The intended contract: under normal conditions developers can
rely on the exemption; under exceptional ones they get a signal, never a silent
stop. Apps should still persist state incrementally as good practice.

**Worker scope.** Dedicated workers inherit the exemption from their owning
page. A shared worker is exempt while at least one of its clients holds a
grant. Service workers are **out of scope**: their lifetime is already governed
by their own event-driven model and by APIs like Background Fetch/Sync, and
this proposal doesn't change that.

## Considered alternatives

- **Background Fetch** — right tool for pure transfers that outlive the page,
  but can't run application JS (transcoding, chunk hashing, custom protocols,
  long-lived apps); complementary, not competing.
- **Background Sync / Periodic Sync** — defer work into short,
  UA-controlled service-worker windows; unsuitable for in-progress,
  long-running, or persistent work.
- **Screen Wake Lock** — keeps the screen on; a wasteful proxy that only works
  while the user stays on the page.
- **Web Locks / `beforeunload`** — no effect on throttling or freezing.
- **Status quo workarounds** — silent audio and dummy WebRTC burn more
  resources than honest background work and invite an arms race with browsers.
- **A scoped, promise-based task API** (silent, time-boxed, UA-budgeted) —
  avoids prompts but fails long-lived apps and gives users no visibility or
  control; the permission model plus a future activity signal captures the
  best of both.
- **Declarative fetch metadata** (e.g. extending `keepalive`) — covers only the
  network leg, with size limits, and no application logic.

## Privacy and security considerations

- **Resource abuse (e.g. cryptomining):** mitigated by the prompt, easy
  revocation, background-activity indicators and task-manager attribution, and
  UA discretion to intervene on egregious use (signaled via `onchange`).
- **Background beaconing/tracking:** gated behind the same consent; the API
  grants no new data access, only scheduling fidelity.
- **Fingerprinting:** one more queryable permission bit; standard Permissions
  API mitigations apply.
- **Prompt fatigue / dark patterns:** the countermeasures developed for
  notifications apply (quieter UI, user-activation requirement, blocking
  abusive origins).
- **Embedded contexts:** Permissions Policy default `'self'` prevents
  third-party iframes from acquiring background execution without explicit
  delegation.

## Open questions

We'd particularly like feedback on:

1. **Default-on for trusted contexts:** should UAs be allowed to grant without
   a prompt for installed PWAs or high-engagement origins, reserving prompts
   for everything else?
2. **Activity signal in v1:** is an explicit "I have pending work" signal
   needed at launch to make idle-site throttling workable, or is it safe to
   defer to a follow-up?
3. **Naming:** `background-work` vs. `background-execution`, and the exact
   shape of the request method (`navigator.backgroundWork.request()` is a
   placeholder).

## References

- [Page Lifecycle API](https://wicg.github.io/page-lifecycle/)
- [Background Fetch](https://wicg.github.io/background-fetch/)
- [Background Sync](https://wicg.github.io/background-sync/spec/)
- [Screen Wake Lock](https://www.w3.org/TR/screen-wake-lock/)
- [Permissions API](https://www.w3.org/TR/permissions/)
- [Permissions Policy](https://www.w3.org/TR/permissions-policy/)
