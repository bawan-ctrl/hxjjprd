# 风控 PRD 交接文档（PRD-020 ~ PRD-023）

**交接日期：** 2026-06-29
**交接人：** bawan
**适用范围：** HashRace Platform 风控系统四个模块的需求设计

---

## 1. 背景与定位

风控系统是**为从零设计的新平台**（GitHub org `hashrace`，后端 `platform-api`，Go module `rk-api`）配套的需求文档。

设计时三个来源的关系：

- **平台侧代码**（`hashrace/platform-api`）：落地依据。代码是参考，**改动是预期内的**——缺的表/字段当作正常开发量，不能因为现有代码没某能力就砍功能。
- **测试环境后台**（`https://eks-admintesta.ugtest888.com`，账号 `tudou08`/密码 `ujk521`）：UI/交互参考。⚠️ 这套后台**不是** `rk-api` 那份代码，rk-api 里目前**完全没有风控代码**。
- **既有 PRD**：版式参考，绝大多数沿用。

**两条贯穿全程的产品决策：**
1. 设计中**不要会员层级（level）、不要会员标签（tags）**。PRD-022 已据此移除「修改会员层级/标签」处罚项及相关批量项、详情列。
2. PRD 是**干净的需求文档**——只写功能本身，不写「复用 vs 新增」「需新增表」「代码参考 file:line」「(按运营可配置)」这类批注。

---

## 2. 四个模块当前状态

| PRD | 模块 | 优先级 | 测试环境 | 本次会话状态 |
|-----|------|--------|----------|--------------|
| **020** | 黑名单管理 | P0 | ✅ 已上线（风控>黑名单，6 页签一一对应） | 未改动（仅文件头仓库列表待统一） |
| **021** | 对赌监控 | P0 | ❌ 无（无 UI 可参考，纯靠代码设计） | 未改动 |
| **022** | 同关联监控（刷子监控） | P0 | ✅ 已上线（风控>刷子监控 `/risk/brush-monitor`） | **已重写**（去层级/标签，已推 main） |
| **023** | 派奖监控 | P2 | ❌ 无 | **已还原为原版** |

**Git 记录（main 分支）：**
- `9390f9e` docs: 还原 PRD-023 原版，更新 PRD-022 同关联监控 ← 最新有效改动
- `de6c74f` docs: 重写 PRD-023（**已被 9390f9e 还原，作废**）
- `b70d750` refactor: 扩展 PRD-020
- `b7c05dc` feat: 添加风控 PRD-020~023 初版

---

## 3. 各模块要点

### PRD-020 黑名单管理（已上线，对得上测试环境）
六大黑名单类型：会员 / IP / 手机 / 设备 / 充值 / 提现账号黑名单，与测试环境 6 页签一一对应。设计模式干净、无批注，可作为其余模块版式样板。

### PRD-021 对赌监控（待定）
识别同一游戏局内对冲投注的可疑会员组合，防优惠对赌套利。**测试环境无对应界面**，只能纯靠代码从零设计。本次未动。

### PRD-022 同关联监控 / 刷子监控（本次主要工作）
- 7 个关联维度：同 IP / 设备号 / 登录密码 / 提现密码 / 提现账号 / 上级代理 / 姓名。
- 默认自动规则配置表（关联原因 / 处罚触发值 / 处罚范围 / 处罚方式）。
- 处罚方式：正常 / 禁止领取优惠 / 加入黑名单 / 冻结 / 禁止进入游戏 / 恢复正常（**已去掉「修改会员层级/标签」**）。
- 同关联人数详情弹窗、变更规则、批量操作、同关联处罚记录页（处罚记录操作含 [改备注]）。
- 文档版本 v1.1。

### PRD-023 派奖监控（已还原原版）
三类监控：高倍爆奖（500x/5000）、大额中奖（20000）、会员获利比（20%/10000）；四个页签：待处理/已处理/已忽略/全部。本次按你的要求还原回初版,未保留中途重写版。

---

## 4. 落地需要注意的代码现状（rk-api）

> 这些是 PRD 落地时的预期开发量，代码缺这些是正常的。

- **完全没有风控/黑名单/同关联/派奖代码**——风控整体从零建。
- `User` 实体：有 ID/Username/Password/Mobile/Email/Nickname/IP(注册IP)/LoginIP/Device/Channel/Promoter/PromoterCode/Color/Status。**无层级、无标签、无真实姓名**；`Status` 仅 0/1/2，无黑名单表、无细分限制状态。
- 注单：**每款游戏各一张表**（crash/dice/limbo/mine/nine/wingo/hash），无统一注单表。字段：UID、RoundID/PeriodID、BetAmount(本金)、Delivery(有效投注=抽水后)、RewardAmount(派奖含本金)、Status、BetTime；倍数仅 dice/limbo/mine 有 Multiple 字段。
- 日聚合实体 `GamerDailyStats` 已定义但**无填充逻辑**——会员获利比（PRD-023）需新建日聚合任务。
- 工程模式：controller→service→repository→entity；分页 `entities/pagination.go`；cron 用 robfig/cron（`task/listen.go`）；MQ 用 asynq；admin 路由 `/api/admin/*` 挂 `AdminMiddleware`（仅 token 校验，无菜单级 RBAC）；**导出功能暂无**。

---

## 5. 待决事项（接手后需确认）

1. **PRD-020 文件头仓库列表**：目前写了 `platform-interface / platform-admin-api / platform-admin-interface`,这三个仓库**不存在**。应统一为只留 `hashrace/platform-api`（PRD-021、023 也有同样问题；022 已改对）。等确认后可一并修。
2. **PRD-021 对赌监控**：是否要按代码从零重做(测试环境无 UI 参考)。
3. **PRD-023 派奖监控**：已还原原版；若要重新适配 platform-api 落地需再确认方向。

---

## 6. 测试环境 / 自动化探查备忘

- 后台：`https://eks-admintesta.ugtest888.com/login`（UG 后台管理系统 / XGCash Admin，Vue + Element Plus），账号 `tudou08` 密码 `ujk521`，无验证码。
- 导航是**顶部横向菜单 + hover 弹下拉**（不是左侧栏）。风控下拉有：黑名单、刷子监控、账号权限、IP白名单、后台日志、非经营地访问限制。
- 自动化：本机 Playwright 1.60 + Chrome，脚本在 scratchpad，`headless` + `storageState` 复用登录态；hover 顶部 nav（y<70 窄元素）再按坐标点子菜单最稳。
