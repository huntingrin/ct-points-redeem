---
name: ct-points-redeem
description: "中国电信积分查询与兑换：话费、流量、影音组合；移动端商城最短路径，登录与支付验证由本人完成，含单次下单保护与到账核对。"
---

# 中国电信积分兑换

先让本人完成一次商城登录，再由 Agent 读取积分、比较商品、准备结算、按本笔授权提交并核对结果。登录、图形码和支付短信验证由本人在官方页面完成。

默认只办理**本机、现金补款 0 元、无自动续订、规则明确**的权益。商品、积分价格、面额、有效期和号码资格一律以本次官网页面为准。

## 先判断本次目的

- **查积分、看能换什么**：只读。读取脱敏号码、可用积分、本年到期积分、网龄和当前目录；不点兑换、不建订单。
- **明确兑换话费、流量或影音**：在用户选定的商品、数量、积分和现金条件内办理。
- **继续上次兑换 / 已付款 / 已到账**：先恢复任务空间并只读查订单与余额，不从选品或支付步骤重来。
- **只问方法**：说明入口和所需验证次数，不擅自下单。

## 最短路径（默认按此执行）

目标：**2 次短信**完成一笔兑换——商城登录 1 次，支付验证 1 次。

1. **先建立移动端环境，再打开页面。** 使用 iPhone UA + 390×844 移动视口。Ego 宿主可用：
   ```js
   await page.cdp("Emulation.setUserAgentOverride", {
     userAgent: "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1",
     acceptLanguage: "zh-CN,zh;q=0.9", platform: "iPhone",
   });
   await page.cdp("Emulation.setDeviceMetricsOverride", {
     width: 390, height: 844, deviceScaleFactor: 3, mobile: true,
   });
   ```
   其他宿主使用对应的设备模拟或移动端浏览器。**不要用桌面 UA 打开移动版话费/收银台**，否则收银台会 302 回 `jf.189.cn/pchome`，看起来像登录失效。

2. **只登录一次商城。** 打开 <https://apps.telefen.com/jycalls/?channelCode=HGOKHD>（话费）或 <https://apps.telefen.com/home/>（综合商城）。未登录会跳到 `m.telefen.com/MobileSSOv2/Login.aspx`，交接给本人填手机号和短信码。登录后回商城。
   - 不要先登录星级中心 `xjfw.189.cn`，兑换不需要它。
   - 不要主动登录 PC 账号 `login.189.cn`。它只在桌面 UA 误触时出现。
   - 不索取聊天中的验证码，不导出 Cookie，不读取输入框值。

3. **读账户。** 等业务数据完整加载后读取：姓名、脱敏号码、可用积分、本年到期积分、网龄。页面初始的 `0` 或空目录可能是占位，等身份和商品都出现再判断；合法余额确实为 0 时如实报告。

4. **选品。** 按 [类别规则](references/redemption-types.md) 核对完整商品、积分、现金、有效期、到账方式和限制。用编号表格给用户比较；不同账号目录可能不同，不套用旧样本的商品 ID、面额或价格。

5. **取得本笔授权。** 用户明确选定商品、数量、积分和现金条件后，才进入下单。仅说“看看”仍是只读。

6. **下单，只点一次。** 选中面额/商品，点「兑换」一次。读取并记录订单号、商品、积分和现金。不要因为页面跳转、加载慢或看起来“没反应”而重复点击。
   - 正常应进入 `cashier.telefen.com/dopay` 收银台。
   - 若跳回 `jf.189.cn/pchome`，先判断为 UA/会话问题：停止点击，切到移动 UA，再查订单；不要继续创建新订单。
   - 若已产生多笔待支付订单，只保留一笔继续支付，其余在订单页「关闭订单」。

7. **收银台核对。** 核对订单号、收款方、商品、`本单支付` 积分、`可用积分` 和现金 0。信息不一致先停下，改方案或对账。

8. **支付验证由本人完成。** 点一次「确认支付」后通常弹出短信验证码。交接同一页面，让本人点「获取短信」、填码并确认。Agent 不读取、不转述、不保存验证码。
   - 本人确认后先判断页面阶段：成功页/处理中只读对账，不再支付。
   - 页面仍在原确认页且条件未变，才允许本人完成的那一次提交；不要由 Agent 重复点。

9. **核对结果。** 分别记录：是否提交、扣分、现金、订单号/状态、到账。扣分不等于到账；「订单已完成」与「运营商/第三方权益已到账」是不同证据。
   - 积分页：<https://jf.189.cn/selfcenter/myPoints.aspx>
   - 订单页：<https://jf.189.cn/selfcenter/order.aspx>
   - 话费成功页：`apps.telefen.com/jycalls/Success?...&result=true`

## 硬性约束

- **移动 UA 是前置条件，不是优化项。** 桌面 UA 下收银台必定回跳，容易诱发重复下单。
- **一笔兑换只允许一次下单动作。** 重复点击会产生多笔待支付订单；待支付不代表扣分，但会污染账户。
- **短信最少 2 次。** 多于 2 次说明走了星级中心或 PC 登录的绕路；下一次应回到最短路径。
- **不把“实付款空白”当成已验证现金 0 元。** 余额不足或页面未加载完时，结算金额可能为空。
- **不承诺全国统一商品。** 全国入口不等于所有号码同目录、同面额、同资格。
- **话费分三种，不能混说：** 直充话费、分期返还话费、业务抵用金。分期返还和抵用金不能当成立即到账的通用余额。
- **影音组合要拆开说：** 例如“20 元话费 + 腾讯季卡”同时含话费与季卡，有效期和到账方式不同。
- **语音包不默认存在。** 两个实测账号均未搜到语音商品；没有当前页面证据就不写“可兑”。

## 可选：本地提交保护

需要跨轮恢复或防止重复提交时，使用 [状态脚本说明](references/transaction-state.md) 中的 `scripts/transaction_state.py`：

```sh
python3 /absolute/skill/scripts/transaction_state.py --state /absolute/task/transaction.json prepare --plan /absolute/task/plan.json
python3 /absolute/skill/scripts/transaction_state.py --state /absolute/task/transaction.json arm-submit --observed /absolute/task/observed.json
python3 /absolute/skill/scripts/transaction_state.py --state /absolute/task/transaction.json record --result /absolute/task/result.json
```

脚本只管理本地 JSON，不调用网站、不读短信、不点击页面。实际状态文件放在任务私有目录，不放进技能安装目录。

## 验收边界（2026-10-21）

已完成一笔真实闭环：浙江 3 星号码，10 元话费直充，1000 积分，现金 0 元；积分 1836 → 836，订单已完成，用户确认到账。该次因桌面 UA 绕路产生 4 笔待支付重复订单，已全部关闭，未重复扣分。

流量和影音只完成目录/详情/规则读取，尚未完成本机真实兑换；语音未发现当前在售商品；金豆签到、生日礼包、短信直兑不在本技能范围。支付按钮由用户完成短信验证，Agent 未读取验证码。

详细页面线索和失败恢复见 [浏览器流程](references/browser-flow.md)。
