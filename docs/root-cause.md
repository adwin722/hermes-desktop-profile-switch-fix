# 根因：切 profile 后「旧会话先出现几秒」的四个叠加缺陷

> **English summary** — Switching profile in the Hermes desktop sidebar showed the *outgoing* profile's
> session rows for 2–8 s (as long as `hermes gateway ensure` dials), and in the grouped "all projects" view
> until a manual Refresh. It is not one bug but four stacked ones: (1) the pick starts a dial without clearing
> the UI (`gateway-switch.ts` `beginProfileSwitch()`); (2) nothing re-pulls the list when the profile scope
> finally moves (`wiring-effects.ts` `useGatewayScopeRefresh()`); (3) activating a shared primary socket
> announced route `default`, mirroring `$activeGatewayProfile` back to the default profile (`gateway.ts`
> `applyActive(profile, epoch, servedProfile?)`); (4) the grouped sidebar renders `$projectTree`, not
> `$sessions`, so wiping the session list left the outgoing profile's project rows painted
> (`projects.ts` `resetProjectsForProfileSwitch()`). The patch fixes all four; the two new test files fail on
> the unpatched tree. The user-visible cost is not the flicker: in multi-agent use the stale rows route a
> task to the **wrong agent**, so the work and its context end up on the wrong role (responsibility and
> context pollution), and during the dial the stale rows stay clickable, so nothing forces a wait.

本文只写：问题是什么、怎么定出来的、改了什么。面向想理解补丁内部结构或想在上游修同样问题的人。
所有路径相对 `apps/desktop/src`。

---

## 0. 一句话结论

「切换 profile 后侧栏先显示旧 profile 的会话、几秒后才变成新的」**不是一个 bug，是四个独立缺陷叠加**：

| # | 缺陷 | 用户看到的现象 | 修法落点 |
|---|---|---|---|
| 1 | 切换只是拨号，界面不拦 | 4~8 秒里旧列表仍可点、可点进旧会话 | `store/gateway-switch.ts` `beginProfileSwitch()` + `store/profile.ts` `selectProfile()` |
| 2 | 切换成功后没人主动拉新列表 | 切完仍挂旧行，早期必须右键刷新 | `app/contrib/wiring-effects.ts` `useGatewayScopeRefresh()` |
| 3 | 共享 socket 激活时，路由被播报成 primary 的 profile（`default`） | 后端重启后几秒，侧栏按 `default` 拉一次 | `store/gateway.ts` `applyActive()` 的 `servedProfile` 参数 |
| 4 | 侧栏处于「项目分组」视图时，行来自**项目数据**，切换时没清 | 清空后先填上旧 profile 的会话条，几秒后才翻新 | `store/projects.ts` `resetProjectsForProfileSwitch()` |

**第 4 条是最后一步才定出来的，也是让「旧会话条先出现几秒」彻底消失的关键。** 前三条都修完时，用户仍能看到
旧行——因为清空的只是会话列表（`$sessions`），而分组视图画的是项目树。

---

## 1. 取证手段（不用调试器）

- 渲染进程的 `console.error/warn` 会被 Electron 主进程捕获进桌面日志，前缀 `[renderer console:main]`。
  在这个应用里它是最有效的取证通道：主进程侧也有一条 `[gateway-route win=main] active key=… profile=…`
  记录路由归属，两边对照就能判断「数据层错了」还是「画错了」。
- 应用运行的是仓库里的 `dist/`：源码改完必须 `node scripts/build.mjs` + 托盘完全退出再打开。只关窗口无效，
  `npm run dev` 还会与正在运行的实例抢 userData 与 gateway 端口。

---

## 2. 四个缺陷的定位过程

### 缺陷 1：切换不拦界面

- 证据：`hermes --profile <X> gateway ensure --json` 实测 4~8 秒（冷启动里 `hermes --version` 本身约 1.8 s，
  其中 `import hermes_bootstrap` 约 2 s）。这段时间 UI 不清列表、不禁点。
- 修法：`selectProfile()` 在拨号**之前**同步置 `$profileSwitchTarget` 并调用 bridge 的 `begin()`；
  `begin()` 清空会话 / 定时任务 / 消息列表 + 置 loading。`use-session-actions` 的 `resumeSession` 在
  `$profileSwitchTarget` 非空时直接返回（禁止点进旧会话）。
- bridge 方向：`store/gateway-switch.ts` 在**模块作用域**注册 `registerProfileSwitchBridge({begin, recover})`，
  `store/profile.ts` 只持引用——因为 profile.ts 不能被 gateway-switch 反向 import（循环依赖）。

### 缺陷 2：切完没人拉新列表

- 证据：唯一可能的被动触发是后台同步里的一个 effect，它开头是 `if (gatewayState !== 'open') return`；
  切换那一刻 socket 正在拨号，它看一眼就走了，之后没人再管。
- 修法：把「会话列表」接到既有钩子 `useGatewayScopeRefresh()`（它本来负责模型 / 配置 / 当前 profile），
  scope 一变就显式 `refreshSessions()`。
- 顺序安全性：这是 passive effect，一定晚于更新 `profileScopeRef` 的 layout effect，所以不会被
  「这页属于旧 profile」的判定丢掉。

### 缺陷 3：路由被播报成 default（最隐蔽）

- 证据（桌面日志，一次切换，已去掉临时探针标记，只留顺序）：

```
select / begin wipe, rows before = 50
sharedPrimary→primary for code-assistant
set @registry code-assistant      ← 修好后这里才对
ridesPrimary→primary for code-assistant
set @registry default             ← 修好前：4 秒后被拽回 default
refresh enter scope=default  rows=29   ← 侧栏按 default 重画
active-route now code-assistant   ← 100 ms 后又变回来
```

- 机理：本地多 profile 共用**同一个 primary socket**（启动日志 `conn:local::<profile> via=primary-shared
  primary=local/default`）。`ensureGatewayForProfile()` 走 `sharedPrimaryRoute()` 分支时调的是
  `applyActive(g.primaryProfile, …)`，于是把 primary 自己的名字 `default` 播报成"当前路由"；
  `use-gateway-boot.ts` 的 `onActiveRouteChanged` 又把它镜像进 `$activeGatewayProfile`，侧栏就按 `default`
  拉了一次列表。
- 修法：`applyActive(profile, epoch, servedProfile?)` 增加第三参数。共享 socket 激活时播报**被服务的那个
  profile**（`key`），而 `activeRouteProfile()` 保持原样（仍按 socket 派生）——因为「是否落地」的判定和
  请求 scope 键必须继续描述 socket，不能描述用户意图。
- 另一处：`ensureGatewayForAgent()` 的 rides-primary 分支。修法不是让它播报 profile，而是
  **「路由已经指向该 profile 就直接放行、不播报」**：

```ts
if (normKey($activeGatewayRoute.get()) === normKey(profile)) return !signal?.aborted
```

  这样后台拨号（切换完成 4 秒后那次）不会打扰前台路由。
- 影响面：`profile-select-webapp-served.test.ts` 原本断言 `activeGatewayProfileKey() === 'default'`，
  那正是上面这个分裂状态，已改为断言被服务的 profile，并在测试里写清理由。

### 缺陷 4：项目分组视图画的是项目数据（决定性一条）

- 证据（侧栏渲染入口的一次计时探针，一次换 profile）：

```
sidebar view rows=0   scope=code-assistant project=__all_projects__ projects=0
sidebar view rows=0   scope=demand         project=__all_projects__ projects=1
sidebar view rows=50  first=demand scope=demand projects=1
```

其中 `projects=1` 意味着侧栏处于「项目 / 主页」分组视图：该视图的行**不来自 `$sessions`**，而来自
`$projectTree`（每个 profile 各自的 `projects.db`）。切换全程 `projects` 一直是 1（旧 profile 的），
所以清空会话列表后，它继续画旧 profile 的项目会话，直到几秒后重新拉回。
- 修法：
  - `store/projects.ts` 新增 `resetProjectsForProfileSwitch()`：`$projects.set([])`、`$projectTree.set([])`、
    `$activeProjectId.set(null)`；
  - `beginProfileSwitch()` 在**同一个同步步骤**里调用它（和清会话列表同理）；
  - 失败回滚 `recoverActiveSourceAfterFailedProfileSwitch()` 里补 `refreshProjects()` /
    `refreshProjectTree()`（失败时 scope 不动，没有别的东西会重拉）；
  - `useGatewayScopeRefresh()` 里补 `refreshProjects()` / `refreshProjectTree()`，保证新 profile 的项目
    概览自己回来。
- 通用教训：**任何"切完还显示旧数据"的问题，动手前先回答"这个视图读的是哪个 atom"**。清空动作只对它自己
  那个 atom 有效；清空 + 四处判据都做足、症状还在，就是一直清错了对象。

---

## 3. 改动文件清单

修改 20 个（合计 +485/−52）：

```
store/gateway.ts                   servedProfile 参数 + rides-primary 放行 + 测试断言更新
store/gateway-switch.ts            beginProfileSwitch() / recoverActiveSourceAfterFailedProfileSwitch() / bridge 注册
store/profile.ts                   $profileSwitchTarget / registerProfileSwitchBridge / selectProfile 进入切换态
store/projects.ts                  resetProjectsForProfileSwitch()
store/gateway.test.ts              新增「共享 socket 播报被选中 profile」
store/profile-select-webapp-served.test.ts   断言改为被服务的 profile
store/reactions-local.test.ts      夹具补 $profileSwitchTarget
app/contrib/wiring-effects.ts      useGatewayScopeRefresh() 补 refreshSessions / refreshProjects / refreshProjectTree
app/contrib/wiring.tsx             传参
app/gateway/hooks/use-gateway-boot.ts          onActiveRouteChanged 镜像 servedProfile
app/session/hooks/use-session-list-actions.ts  live scope 守卫 + 迟到页作废
app/session/hooks/use-session-list-actions.test.tsx
app/session/hooks/use-session-actions/index.ts 切换期间禁止 resume
app/chat/sidebar/index.tsx
app/chat/sidebar/profile-switcher.tsx          pending 转圈（Loader2）
app/chat/sidebar/profile-dropdown-switcher.tsx pending 转圈（行尾）
app/chat/sidebar/profile-*.test.tsx（4 个）     夹具补 $profileSwitchTarget
```

新增 2 个：

```
store/profile-switch-barrier.test.ts                     4 条：同步清空 / 同 profile 不触发 / 失败回滚 / 成功清标记
app/contrib/wiring-effects.gateway-scope-refresh.test.tsx 5 条：scope 移动才拉 / 连接移动 / 重渲染不重拉 / 失败不抛
```

---

## 4. 验证

```bash
cd apps/desktop
npx tsc -p tsconfig.json --noEmit          # 0 错误
npx vitest run src/store/profile-switch-barrier.test.ts \
               src/app/contrib/wiring-effects.gateway-scope-refresh.test.tsx \
               src/app/session/hooks/use-session-list-actions.test.tsx \
               src/store/gateway.test.ts \
               src/store/gateway-background-reconnect.test.ts \
               src/store/profile-select-webapp-served.test.ts
# 2026-11-14 实测：6 文件 / 76 条全过
```

手工验收（分组视图是重点）：

1. 开「所有项目」分组视图，profile A 有 50 行、profile B 有 29 行；
2. A → B：侧栏应在点击瞬间清空（骨架），**不应**出现 A 的会话条，随后出现 B 的行；
3. B → A 同样；
4. 把网关拨号故意弄慢（或切到冷 profile）再看一遍：界面必须停在骨架，而不是停在旧行。

已知 flaky：`gateway-background-reconnect.test.ts` 在机器负载高时会因 15 s 超时而挂（重连次数断言未变），
单独跑与空闲时复跑均通过。
