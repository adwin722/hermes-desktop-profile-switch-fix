# Reproducing the bug (before the patch)

> 中文速览：两份 profile（一个 50 条会话、一个 29 条，数量差得越大越好）→ 左下角切换 profile → **别发消息**，
> 盯侧栏 5 秒。坏版：旧 profile 的会话条一直挂着（4~8 秒，直到后台拨号完成），分组视图更久、要手动刷新，
> 而且这期间旧会话还能点开。好版：点击瞬间列表清空 + 加载骨架，旧行不再出现，随后自动填入新 profile 的列表。
> 同一时刻只允许开一个 Hermes 实例（两个会抢 userData 和网关端口）。

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
