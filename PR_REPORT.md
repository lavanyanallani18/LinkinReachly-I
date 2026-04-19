# PR Write-Up

**Author:** Lakshmi Lavanya Nallani  
**Email:** lnallani@umd.edu  
**Branch:** lavanya-nallani/fix-bridge-fallback-sdui  
**Files changed:** src/main/easy-apply/click-apply.ts, tests/unit/main/easy-apply-guards.test.ts

---

## What I Found

The first thing I did was run the test suite to understand what was already there and what
was already broken. Out of 70 test files, one had a real failure -- not a build issue,
not a skip, but an actual wrong value coming out of production code:

```
FAIL  tests/unit/main/easy-apply-guards.test.ts

AssertionError: expected 'Could not find Easy Apply button.'
             to match /form didn't open/i
```

The test was checking a specific scenario: LinkedIn's SDUI (Server-Driven UI) apply flow
sometimes takes you to the apply URL and then immediately bounces you back to the job
listing page with no form, no modal, nothing. When that happens, the system should tell
the user "The Easy Apply form didn't open on this page" -- an informative message that
makes sense. Instead it was returning "Could not find Easy Apply button." -- which is
technically a different failure and gives the user no useful information about what
actually went wrong.

That mismatch told me something in the apply-button click path was short-circuiting
before it had a chance to detect the SDUI redirect failure.


## Why It Was Broken (Root Cause)

The bug is in `easyApplyClickApplyButton()` in `src/main/easy-apply/click-apply.ts`.

This function is responsible for finding and clicking the Easy Apply button on a LinkedIn
job page. Looking at its structure, I noticed it splits into two paths based on whether
there's an active Chrome tab ID:

```ts
const tabId = getActiveLinkedInTabId()
if (tabId != null) {
  // use CDP (Chrome DevTools Protocol) to locate and click the button
}
// nothing else -- no else branch
```

When there IS a tab ID, the function uses CDP to physically locate the button in the DOM
and simulate a real mouse click. That's the new, smarter path.

But there's no `else`. If `tabId` is null, the function just... does nothing. It skips
the entire locate-and-click step and falls through to a generic check that asks the
extension "is the form already open?" If the form isn't open (which it won't be, because
nothing clicked anything), it returns "Could not find Easy Apply button." and exits.

The reason this matters is that `tabId` is null in two real situations:

1. In tests -- the test mock for `getActiveLinkedInTabId()` returns null by design,
   because tests don't have a real Chrome tab.
2. In production -- when a job starts processing before the Chrome tab has been registered
   with the Electron main process.

Before CDP was introduced, the function used the Chrome extension bridge to do the work:
it called `LOCATE_EASY_APPLY_BUTTON` first, then `CLICK_EASY_APPLY`. Those bridge command
results (especially the SDUI apply URL that comes back from the click) are what feed into
the SDUI navigation handler downstream. When CDP was added as the primary path, the old
bridge path simply wasn't moved into an `else` block -- it was just removed. So any time
`tabId` is null, the SDUI detection logic never runs and the wrong error message comes out.


## How I Approached the Fix

The fix is an `else` branch that restores the bridge-command locate+click path for when
`tabId` is null. I didn't touch the CDP path at all.

```ts
} else {
  // No active CDP tab -- use bridge commands to locate and click the button
  const locateRes = await easyApplyBridgeCommand(
    'LOCATE_EASY_APPLY_BUTTON', {}, 'click_apply', 'bridge_locate'
  )
  if (locateRes.ok) {
    // If the located button is an SDUI anchor, capture its URL
    const locateData = 'data' in locateRes ? locateRes.data : {}
    if (locateData.sduiApplyUrl) locatedSduiApplyUrl = String(locateData.sduiApplyUrl)

    // Ask the extension to click it
    const bridgeClick = await easyApplyBridgeCommand(
      'CLICK_EASY_APPLY', {}, 'click_apply', 'bridge_click'
    )
    clickResult = { ok: bridgeClick.ok, detail: bridgeClick.detail, ... }

    // Also capture SDUI URL from the click response if present
    if (bridgeClickData.sduiApplyUrl) locatedSduiApplyUrl = String(...)
  }
}
```

Once this block runs, `clickResult` and `locatedSduiApplyUrl` are populated the same way
the CDP path populates them. Everything that follows -- the SDUI navigation handler, the
fast-fail diagnostic check, the informative error message -- runs exactly as it was
designed to. No changes needed anywhere else.

### Trade-offs I thought about

I considered a few other ways to solve this before settling on the else branch:

**Could I just change the failing test to expect the generic message?**
No. The test is right. "Could not find Easy Apply button" is a message that belongs to
a different failure (button literally not present on the page). The test was documenting
correct expected behavior, and hiding the defect by changing the assertion would make the
system worse without fixing anything.

**Could I move the bridge locate+click inside `checkFormAlreadyOpen()`?**
No. That function's job is to check whether the Easy Apply form is already on screen --
it's not a locate-and-click mechanism. Putting that logic in there would make the code
confusing and violate the single-responsibility principle.

**Could I always run the bridge path first, regardless of whether tabId is set?**
That would work but it's wasteful. In the common production case where a tab IS active,
you'd be making two extra round-trips to the extension before even attempting the CDP
click. The else branch is cleaner -- CDP when you have a tab, bridge when you don't.

**What I went with:** The else branch is the minimum correct change. It restores
something that existed before in exactly the place it was missing. It's ~30 lines of
straightforward code, easy to read, easy to review, and it doesn't disturb anything else.


## Tests

### The test that was failing before my fix

```
easy-apply guards
  [FAIL] returns a user-facing unavailable message when SDUI force navigate
         lands back on jobs/view
```

This test mocks all four bridge commands involved in the SDUI bounce scenario
(LOCATE_EASY_APPLY_BUTTON, CLICK_EASY_APPLY, FORCE_NAVIGATE, DIAGNOSE_EASY_APPLY) and
checks that when the diagnostic detects a non-apply landing page, the error message
matches "/form didn't open/i". It was failing because the bridge commands were never
being called at all -- the function was exiting before it got that far.

After the fix it passes.

### New tests I added

I added two new tests in a new describe block called
"easy-apply bridge-command fallback (no active CDP tab)":

**Test 1 -- verifies the call order**
When tabId is null and LOCATE_EASY_APPLY_BUTTON returns ok:true, the function should
call LOCATE_EASY_APPLY_BUTTON first and then CLICK_EASY_APPLY. A successful click should
result in earlyExit being null (meaning the apply flow continues normally). I verify both
the call order and the earlyExit value.

**Test 2 -- verifies graceful failure**
When tabId is null and LOCATE_EASY_APPLY_BUTTON returns ok:false (button not found),
the function should fall through to the generic "Could not find Easy Apply button." message.
This makes sure the else branch degrades cleanly when the bridge locate itself fails.

### Full test run results

Before:
```
Test Files  6 failed | 64 passed (70)
     Tests  1 failed | 600 passed | 33 skipped
```

After:
```
Test Files  5 failed | 65 passed (70)
     Tests  603 passed | 33 skipped
```

The 5 remaining failures are all in `tests/unit/extension/content-*.test.ts`. They fail
on the unmodified initial commit too -- they need a compiled `content-bundle.js` which
requires running `npm run build` first. They are not related to this change.
