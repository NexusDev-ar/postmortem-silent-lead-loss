# Postmortem: a contact form that looked like it worked

## Impact
An unknown number of leads submitted through the contact form on
nexusdev.com.ar never reached anyone. The form reported success every
time. Nothing was logged, because no code path existed that could fail.
The bug lived for as long as the form did.

## What the code actually did
The submit handler built a WhatsApp deep link and called
`window.open(waLink(...))`. That was the entire delivery mechanism. The
"lead" was a prefilled message sitting in a new tab on the visitor's
machine. It reached me only if the visitor then pressed send.

It failed silently whenever the visitor was not logged into WhatsApp
Web, the popup was blocked, or they simply closed the tab. All three are
common, and none of them are visible from my side.

## Why it went unnoticed
The failure mode is invisible by construction. There is no error to
catch: `window.open` returning null is the only signal, and the original
code ignored it. Zero leads on a given day is indistinguishable from a
quiet day. I was reading the absence of complaints as evidence of
working software.

## How I found it
I compared analytics sessions reaching the contact page against messages
actually received. The ratio was not plausible for any conversion rate.
I then reproduced it directly with popups blocked in the browser, which
made the failure obvious and immediate.

## The fix, and the second bug it caused
The form now POSTs to `/api/lead`, which writes the row to Postgres
(Supabase) and only afterwards fires the WhatsApp notification. The
database write is the product; the notification is a courtesy.

My first version awaited that fetch before calling `window.open`. That
broke the feature in a new way: once an await resolves, the browser no
longer treats the subsequent `window.open` as user-activated, and blocks
it as a popup. The fix was to fire the request without awaiting it and
pass `keepalive: true`, so it survives the page losing focus.

## What I changed in how I work
Persist before notifying. Any design where the record of an event exists
only in a message is a design where the event can vanish. If I cannot
name where the row lands, the feature is not finished.

## The notification path, one bug later
The notification no longer depends on the visitor at all. It is sent
server-side from `/api/lead` after the row is written, and a failure to
notify can no longer be silent: every failed send is logged at error
level with the full lead in the same line, so the record exists twice.

That path had one more lesson in it. The provider answers `200` even
when it rejects the request, so `r.ok` is not a success signal; the
response body has to be read and checked. A success code that isn't one
is the same class of bug as the original — something reporting that it
worked when it didn't.

## Still open
- Whether a notification actually arrives on my phone is still confirmed
  by noticing it, not by anything automated. The database remains the
  source I trust.
- No deduplication: a visitor who submits twice creates two rows.
- No alerting on an empty day, which is the exact condition that hid the
  original bug.
