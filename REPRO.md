# Reproducing the bug (before the patch)

Goal: see the previous profile's sessions stay in the sidebar after a profile switch, on an unpatched build.

## Setup

1. Two profiles with **visibly different session counts** (the recording uses 50 rows vs 29 rows). If both
   profiles have 5 sessions you will not see the swap.
2. A desktop build **without** the patch. Either run an upstream checkout at `8531ae7bb0` (or any commit before
   the fix), or apply the patch, rebuild, and confirm the difference:
   ```bash
   grep -rl "profileSwitchTarget" apps/desktop/dist/assets | wc -l   # patched ≈ >0, unpatched = 0
   ```
3. Only **one** Hermes instance may run. Two instances fight over `userData` and the gateway port and both
   show random results.

## Steps

1. Start Hermes, open the desktop window, note which profile is active and how many sessions the sidebar lists.
2. Click the profile dropdown in the bottom-left corner and pick the other profile.
3. **Do not send a message.** Watch the sidebar for 5 seconds.

## Expected on the unpatched build

- The outgoing profile's session rows stay painted for roughly 2–8 s (as long as `hermes gateway ensure`
  needs to dial the other profile's gateway).
- There is no loading state on the list during that window — the stale rows look like real data.
- Sessions from the outgoing profile are still clickable and can be opened.
- In the grouped "all projects" sidebar view the stale rows persist even longer: that view renders
  `$projectTree`, which is only refetched when something else happens to invalidate it — a manual sidebar
  Refresh is the reliable way out.

## Expected on the patched build

- Within one frame of the click the list empties and the loading skeleton appears.
- The stale rows never appear, and are not clickable.
- The target profile's rows land on their own, without a Refresh (4–8 s, the gateway dial time is unchanged —
  this patch fixes what is *shown*, not how fast the backend answers).

## Timing note

`hermes --profile <name> gateway ensure` measured 4–8 s on a warm machine (`hermes --version` alone is ~1.8 s,
of which ~2 s is `import hermes_bootstrap`). That is the window the patch has to cover; it is why the fix is a
synchronous wipe plus a skeleton rather than a spinner-free wait.
