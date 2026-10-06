# Synthetic example — review handoff

> **Synthetic example inspired by the production workflow.** The task, the file paths, the class names, and the findings are made up. The *shape* of the exchange is faithful to how the real system works.

## Setup

A fictional task: a listing component needs a "last refreshed" indicator in its header. Simple-looking, but it touches three disciplines (front-end for the display, back-end for the timestamp source, docs for the user-facing explanation).

## Phase 1 — Coordinator dispatches

The coordinator reads the 01-Requirements file and the 02-Solutions file. The design file specifies three parallel tracks and the exact scope of each.

```text
> Dispatch: track A (front-end)
> Scope: components/ListingHeaderBadge.tsx — add a "last refreshed" badge
> Requirements: visible in RTL and LTR; translation key via T().
> Review target: L1 Front-end.

> Dispatch: track B (back-end)
> Scope: ListingEndpoint — include a 'lastRefreshedUtc' field on list responses
> Review target: L1 Back-end + L1 DB for the source-of-truth question.

> Dispatch: track C (docs)
> Scope: Docs/{en,ro}/Concepts/Listings.md — one sentence on the new badge
> Review target: L1 Docs.
```

## Phase 2 — L1 Back-end review (one agent, one file)

The back-end L1 reads the proposed diff for the endpoint and reports:

```text
Findings:
1. [back-end/correctness] ListingEndpoint returns lastRefreshedUtc from
   DateTime.UtcNow at response time — this is "when the response was built",
   not "when the data was last refreshed". Users will see the timestamp
   advance on every click even if nothing changed.
   File: ListingEndpoint.cs:214
   Suggested check: pull lastRefreshedUtc from the shared row-data
   cache's RefreshedAt property so the badge reflects actual cache
   rebuilds.

2. [back-end/i18n] lastRefreshedUtc is sent as a raw ISO string; the SPA
   needs the user's timezone and locale to format it. If the SPA formats
   locally, this is fine; if the server does the formatting, the string
   is wrong for most users.
   File: ListingEndpoint.cs:214
   Suggested check: confirm which side formats.
```

Both findings have file-and-line references. Both are *claims*, not conclusions.

## Phase 3 — L2 fan-out verifies

Each L1 finding is handed to L2 verifiers independently.

### L2 verifier A — on finding #1

```text
Attempting to refute "lastRefreshedUtc from DateTime.UtcNow is wrong":

Looked at the cache implementation. The shared row-data cache does
track RefreshedAt: a RebuildCompleted hook sets _refreshedAt =
DateTime.UtcNow at the end of a successful rebuild.

If the endpoint used _refreshedAt, two cases:
- Table is flagged as cached in metadata. Then RefreshedAt is the
  correct source; the finding is CONFIRMED.
- Table is not cached. Then there is no RefreshedAt; the finding would
  propose a value that does not exist. In that case the badge should
  simply not render — not "fall back to Now".

Verdict: CONFIRMED, with a refinement — the fix needs to handle the
non-cached branch explicitly (no fallback to Now).
```

### L2 verifier B — on finding #2

```text
Attempting to refute "the SPA needs timezone to format":

Checked three things, in order:

1. Payload shape: the proposed field is DateTime.UtcNow serialized by
   Newtonsoft, which emits "...Z" (UTC with explicit offset). So the
   wire carries unambiguous instant-in-time, not a floating local time.
   Good: that eliminates the "server-formatted-for-wrong-fuseau" case.

2. SPA parsing: dayjs(iso) treats "...Z" as UTC and converts to the
   browser's local timezone on access. Confirmed by reading the header
   component — it calls .format(T("datetime.short")) on the parsed
   value, no explicit .utc() or .tz(), so the display is in the
   browser's local timezone (which is what the user wants for a
   "last refreshed" badge).

3. Locale vs. timezone separation: locale (en-US / ro-RO) controls
   DIGIT ORDER and SEPARATOR; timezone controls OFFSET. The existing
   header pulls locale from the active session and timezone from the
   browser. Both are correct for this field.

One residual risk: if a user explicitly set a non-local timezone
preference in their profile (feature exists for scheduled reports),
the badge would still show browser-local. Low-impact for a "last
refreshed" indicator; worth a comment in the code so the next
touch does not confuse it with scheduled-report display.

Verdict: REFUTED on the main claim. Low-priority code comment
recommended for the profile-timezone edge case.
```

## Phase 4 — Coordinator consolidates

```text
Verified findings:
1. [CONFIRMED] lastRefreshedUtc should come from the cache's RefreshedAt;
   if the table is not cached, do not render the badge.
   Action: update ListingEndpoint.cs:214 + ListingHeaderBadge.tsx
           (conditional render).
2. [REFUTED] (dropped) ISO payload is already handled correctly SPA-side.

Dispatching fix pass to the back-end worker with the exact action above.
No open loop — one fix, one delta review, then PR.
```

## Why this works

Three things made this review useful:

1. **The reviewer worked in a tiny lane.** The back-end L1 only read the endpoint and the cache; it did not need to know the SPA at all.
2. **The reviewer wrote claims, not conclusions.** The L2 fan-out had material to argue against.
3. **One of the findings was refuted and dropped.** This is as valuable as confirming the other — it keeps the fix cycle short and the engineer's attention on real defects.

## What a bad version of this would look like

A single generalist reviewer, with the whole repo in context, might have produced a finding like:

> *"The timestamp handling looks a bit off, consider refactoring to pull from a shared source and format consistently across the app."*

No file, no line, no scenario, no scope. The engineer cannot act on it; the agent cannot refute it; the finding is noise. That is the shape of unreviewed AI review output. The adversarial pipeline exists to prevent that shape from surviving.
