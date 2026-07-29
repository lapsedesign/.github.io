# 工程管理模块 — 并入 lapse-quotation 的实作规格

`engineering.html` 是这个模块的**可点击原型**。配色、字体、排版原语、报价单号格式、
收款节点百分比、状态色，全部对齐 `lapsedesign/lapse-quotation` 的现有实作，
因此可以直接当成视觉与逻辑的验收标准。

本文件说明如何把它变成团队实际使用的系统。

---

## 1. 结论:做成报价系统的一个模块,不另开专案

新增路由 `apps/web/app/projects`,与 `/quote/[id]` 并列。理由:

| | 并入报价系统 | 另开独立 App |
|---|---|---|
| 团队登入 | 沿用 `middleware.ts` + next-auth,零工作 | 要重做一套 |
| 设计系统 | 沿用 `globals.css` + tailwind tokens | 要复制维护两份 |
| 资料串接 | 同一个 `sheets-client`,直接读报价单 | 要另接一次 Sheets |
| 部署 | 同一个 Vercel 专案 | 多一个专案、多一组环境变数 |
| 团队记忆 | 一个网址 | 两个网址 |

**关于「做成 App」**:报价系统 `apps/web/public/manifest.json` 已经设定
`"display": "standalone"`,本来就是 PWA。手机浏览器开启后「加到主画面」即得到
一个无网址列、有图示的 App。工程管理并进同一个 App 后自动继承,不需要另外打包。

---

## 2. 资料从哪里来 — 报价单即案场

报价单状态走到 `Confirmed` 或 `Processing`,就代表这个案子已成交、进入施工。
这些单子**自动**出现在工程管理,不需要重新输入客户名、地址、合约金额。

### 沿用报价系统既有栏位(`QUOTE_DATABASE`,A–S 栏)

| 工程管理显示 | 来源 |
|---|---|
| 案场名称 | `Quote.client` |
| 地址 | `Quote.address` |
| 联络电话 | `Quote.contact` |
| 报价单号 | `Quote.id`(如 `QT260112`) |
| 合约总额 | `calcGrandTotal(items).grandTotal` — **即时算,不另存** |
| 收款节点 | `Quote.paymentMode` → 10/40/40/10 或 50/30/20 |
| 单据状态 | `Quote.status` |

> **收款节点金额不进资料库。** 由 `paymentMode` × 合约总额即时算出,与
> `packages/pdf-template/src/QuotePDF.tsx` 的 `terms-schedule` 共用同一组百分比常数。
> 这样报价单 PDF 与工程管理页面的金额**不可能对不上**。
> 唯一需要记录的是「已收到第几期」与「收款日期」。

### 新增 `PROJECT_DATABASE`(一个案场一列)

| 栏 | 栏位 | 说明 |
|---|---|---|
| A | `quoteId` | 主键,对应 `Quote.id` |
| B | `stage` | 目前阶段 `0–6`(0=拆除进行中,6=全部完工) |
| C | `stageDates` | 六个阶段完成日,逗号分隔 |
| D | `pic` | 现场负责人 |
| E | `startDate` | 实际开工日 |
| F | `dueDate` | 合约完工日 |
| G | `paidUpTo` | 已收到第几期 |
| H | `paidDates` | 各期收款日,逗号分隔 |
| I | `note` | 现场备注 |
| J | `updatedAt` | ISO 8601 |

**日期栏刻意用逗号分隔字串塞进单一栏位**,不展开成多栏 —— 报价系统栏位现在跑到 S 栏,
每加一个栏位就要同步改 `upsertQuote` 的 row array、`getQuote` 的读取 index、以及
`A2:S10000` 这类 range 字串(见 skill 的三处同步陷阱)。工程管理独立一张表、
栏位固定在 A–J,就完全避开这个雷区。

---

## 3. 需要写的档案

```
packages/shared-types/src/index.ts     ← 加 Project 型别 + STAGES 常数 + 阶段/收款计算函式
packages/sheets-client/src/index.ts    ← 加 getProject / listProjects / upsertProject
apps/web/app/projects/page.tsx         ← 案场列表(原型的 grid)
apps/web/app/projects/[id]/page.tsx    ← 案场详情(原型的抽屉,改成独立页更好用)
apps/web/app/api/project/[id]/route.ts ← GET / PUT
```

`STAGES`、`SCHEDULES` 两组常数放 `shared-types`,与 `QUOTE_STATUSES` 并列 ——
改一处全栈跟着改,这是这个 repo 既有的约定。

### 双向链接
- 报价单页 `/quote/[id]`:状态为 `Confirmed`/`Processing` 时,显示「查看工程进度 →」
- 案场页 `/projects/[id]`:显示「查看报价单 →」(原型底部那个连结)

---

## 4. 施工阶段

```
拆除 → 水电 → 泥作 → 木作 → 油漆 → 收尾
```

整体进度 = `stage / 6`。这是**刻意的粗颗粒设计**:现场负责人用手机推进度,
点一下就好,不需要填百分比。要更细可以之后在阶段底下加勾选项,但先不要。

## 5. 逾期判定

- **工期逾期**:今天 > `dueDate` 且 `stage < 6`
- **款项逾期**:今天 > `dueDate` 且仍有未收期数 → 计入首页「已逾期应收」

两者都是**算出来的,不是手动标记的**,所以不会有人忘记更新。

---

## 6. 上线前必须注意(来自既有踩雷纪录)

1. **Sheets 写入必须用 append + `INSERT_ROWS` 再删旧列**,不要改用指定列号 `update` ——
   之前造成过栏位错位的资料遗失。
2. **`PROJECT_DATABASE` 要预留足够空白列**给 append 成长。
3. **新增 Vercel 环境变数后必须重新部署**才会生效,既有部署带的是旧的环境变数快照。
4. **中文不要经由终端机参数写入**(PowerShell cp1252 会乱码),一律用档案写 UTF-8。
5. 无测试套件,验证方式:各 package 跑 `npx tsc --noEmit`,部署后手动点过一轮。

---

## 7. 建议的推进顺序

1. `shared-types` 加 `Project` 型别与常数 → `npx tsc --noEmit`
2. Google Sheets 手动开 `PROJECT_DATABASE` 分页,填 A–J 表头 + 预留空白列
3. `sheets-client` 加三个函式,先用 script 读一笔验证
4. `/projects` 列表页(唯读)→ 部署 → 团队先看得到
5. `/projects/[id]` 加上「推进阶段」「标记收款」两个写入动作
6. 报价单页加双向连结

第 4 步就已经对团队有价值了 —— 先让大家看得到进度,写入功能可以后面再补。
