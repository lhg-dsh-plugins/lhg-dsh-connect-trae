# BUGFIX — v4.4.4

## Bug 标题

同价的会员分支被读成第二重折扣：`badgeOf()` 无条件把 `member` 比例写进徽标，悬停称「会员价另有 X 折」，暗示在现价上再打折；且上游实时判定出的闲时徽标不带时段（上游 issue #53，HEAD v4.4.3 未修）

## 证据链（修复前）

`src/promo.ts` 的 `badgeOf()` 对任何带 `member` 分支的活动无条件计算
`memberDiscountPercent = discountOf(discount.member)` 并写进徽标，客户端
（`TraeUsageCard.tsx`、`TraeModelLabel.tsx`）据此渲染悬停文案
「会员价另有 {zhe} 折」/ "subscription price is a further {percent}% off"。

但 `member` 与产生徽标的分支（`current` / `off_peak`）是**同一个折后价的两个
资格入口**，不是可叠加的两层折扣。issue #53 的实测（2026-10-23）：
`deepseek-v4.1-flash` 的活动 `off_peak_member_discount_18_9` 中，`member`
与 `off_peak` 两分支同为 `0.12 → 0.06, discount 50`——会员全天可享、非会员
只在闲时窗口内可享，是「谁能拿到」的差别。旧实现下悬停读作 0.06 再打五折
= **0.03**，上游从来没有这个价。

离线复现（临时 vitest spec，夹具即仓库自带 `tests/promo.spec.ts` 的实测
文档形状；验毕已删除、未入库）：

- 同价用例（`current: none` + 闲时窗口开启，`member.rate = 0.06` 与徽标
  现价 `0.06` 相等）：基线 `memberDiscountPercent = 50`（应为 `undefined`），
  断言失败 `expected 50 to be undefined`；
- 真便宜用例（`member.rate = 0.1 < 现价 0.2`）：基线无 `memberRate` 字段
  （应为 `0.1`），断言失败；
- 上游 `current` 实时判定为闲时活动的用例：基线徽标 `fromClock = undefined`
  （应为正在运行的窗口 `00:00`），断言失败。

三项在基线全红、修复后全绿。另两条 open issue 已排除：#49（draft-3
`required:true`）已在 HEAD 的 `src/raw-chat.ts` 归一化实现，不重复处理；
#52（模型数量爆炸）是产品语义诊断、无最小修复面。

## 根因

1. `badgeOf()` 把「解析到了 member 分支」直接等同于「会员另有更低价」，
   从不比较两条分支的**折后价**。比例（discount）相同不代表可叠加——同价
   时引用 member 比例就是虚构第二重折扣。
2. `selectTraePromoBadge()` 的实时判定分支（走上游 `current` 裁决）调用
   `badgeOf()` 时不传窗口，于是闲时类活动经此路径出的徽标没有
   `fromClock/untilClock/timeZone`，悬停说不出是哪几个小时；只有本地窗口
   兜底路径才带时段。

## 改动文件

- `src/promo.ts`（核心逻辑）：
  - 新增 `cheaperMemberOf()`：按折后价比较，只有
    `member.rate < 徽标现价 rate` 时才产出会员引用；两边任一缺 rate 则
    不引用（无可比之数不编话）。
  - `TraePromoBadge` 新增 `memberRate`；`memberDiscountPercent` 重新定义为
    「仅当会员价严格更便宜时存在」，`badgeOf()` 改走 `cheaperMemberOf()`。
  - 实时判定分支对 `offPeak` 类活动附加 `runningWindow()` 算出的正在运行
    窗口；无窗口在运行或时钟不可判（未知时区）时不附加。
- `src/client/locales.ts`：`row.promoHintMember` 文案改给绝对数——
  zh `会员价 {rate} 积分/百万 token（{zhe} 折）`、
  en `subscription price is {rate} credits/Mtoken ({percent}% off)`；
  等值时整句随字段缺席而不出现。
- `src/client/TraeUsageCard.tsx` / `src/client/TraeModelLabel.tsx`：会员
  悬停要求 `memberRate` 与 `memberDiscountPercent` 同时在场，并传入
  `rate` 占位符。
- `package.json`：version 4.4.3 → 4.4.4；`repository.url` 改指本 fork
  落点仓（diff 恰为这两行）。上游无包内版本戳常量，pnpm-lock 不记录
  根包版本，无其他待同步项。
- `README.md`：顶部追加中文 lhg fork 头，上游原文一字未删；`LICENSE`
  原样保留；新增 `UPSTREAM.md` / 本文件。

## 复验结果

- 复现 spec：修复前 3 红 → 修复后 3 绿。
- 相关既有测试 `tests/promo.spec.ts` + `tests/card-promo.spec.tsx` +
  `tests/model-label-slot.spec.tsx`：修复前后均 40/40 通过（真便宜的
  subsidy 夹具 `member.rate 0.1 < 现价 0.2`，会员引用按预期保留）。
- 全量 `npx vitest run`（修复后单独运行）：82/82 文件、1017/1017 测试
  全部通过（1014 条上游测试 + 3 条临时复现）。
- `git stash` 对照：`vitest list` 在修复树与基线树均为 1017，测试收集
  零增减；复现 spec 在基线源码上复跑仍为 3 红，钉住缺陷为上游既有。
- `pnpm run typecheck`（双 tsconfig）修复前后均退出 0；`pnpm run build`
  修复后成功。
- 基线说明：修复前全量 `npx vitest run` 为 81/81 文件、1014/1014 测试
  全部通过——本缺陷未被上游既有测试覆盖，失败证据由上述复现 spec 提供
  （基线源码 3 红、修复后 3 绿），修复未触碰任何既有测试的预期。

## 验证边界

- 全程离线：未连真实 Trae 账号/上游接口；夹具用的是仓库自带测试与
  `docs/PROMO_BADGE.md` 记录的真实 CN 响应形状及 issue #53 的实测数字。
- 未在 DSH Desktop 真实宿主里做端到端渲染验证；客户端改动仅限悬停
  句子的字段门槛与占位符，胶囊本体渲染路径未动。
- 同价判定依赖两条分支都发布了折后价（rate）：任一缺失时本版选择
  「不引用会员价」而非按比例猜测——宁可少说，不编价格。
- issue #53 附带的官方客户端渲染取证文档（`docs/PROMO_CLIENT_RENDER.md`）
  未移植，本版只落代码修复；CHANGELOG 未改（上游习惯由维护者合并后
  单独提版本 commit，本 fork 的版本号体现在 package.json）。
