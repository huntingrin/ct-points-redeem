# 官方商城浏览器流程

在登录、选品、下单、收银台或恢复时读取。使用宿主浏览器能力；Ego 宿主遵循其 TaskSpace、后台操作和交接规则。

## 会话地图（先看这一节）

电信积分链路跨三套登录体系，登录态不通用：

| 域 | 作用 | 是否必需 |
|---|---|---|
| `apps.telefen.com` / `m.telefen.com` | 移动版积分商城、话费专区、收银台入口 | ✅ 必需，登录一次 |
| `xjfw.189.cn` | 星级服务中心（星级、网龄、权益） | ❌ 兑换不需要 |
| `jf.189.cn` / `login.189.cn` | PC 积分商城、PC 账号体系 | ❌ 最短路径不需要 |

**结论：只登录 `apps.telefen.com` 对应的商城 SSO。** 先登星级中心或 PC 账号都会增加一次短信，且不会让商城免登录。

## 入口

- 话费兑换专区：<https://apps.telefen.com/jycalls/?channelCode=HGOKHD>
- 综合商城首页：<https://apps.telefen.com/home/>
- 我可兑换：`apps.telefen.com/home/nex?channel=HGOKHD&promoid=...`
- 商城登录：`m.telefen.com/MobileSSOv2/Login.aspx?appcode=HGOKHD&goBack=...`
- 收银台：`cashier.telefen.com/dopay?payflag=...` → `cashier.telefen.com/ssocallback?...`
- 成功页：`apps.telefen.com/jycalls/Success?...&result=true`
- 积分：<https://jf.189.cn/selfcenter/myPoints.aspx>
- 订单：<https://jf.189.cn/selfcenter/order.aspx>

`channelCode=HGOKHD` 是官方移动商城渠道参数；不要写死 `promoid`、`payflag`、`Token` 或商品 ID。

## 第一步：移动端环境

**必须在打开页面前完成。** 桌面 UA 下，收银台会 302 到 `https://jf.189.cn/pchome/`，表现为“点兑换后回到商城首页”。

Ego 宿主：

```js
await page.cdp("Emulation.setUserAgentOverride", {
  userAgent: "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.0 Mobile/15E148 Safari/604.1",
  acceptLanguage: "zh-CN,zh;q=0.9",
  platform: "iPhone",
});
await page.cdp("Emulation.setDeviceMetricsOverride", {
  width: 390, height: 844, deviceScaleFactor: 3, mobile: true,
});
```

其他宿主：使用设备模拟（iPhone / 390×844 / mobile）或在移动端浏览器中打开。不要在桌面窗口直接操作移动版收银台。

## 第二步：登录

1. 打开话费专区或商城首页。
2. 未登录会跳 `m.telefen.com/MobileSSOv2/Login.aspx`，页面为手机号 + 短信验证码 + 登录。
3. 交接当前页，由本人填手机号、获取并填写短信码。
4. 登录后通常回到 `apps.telefen.com/nstars/iage?...&Token=...` 或商城首页。
5. 核对页头脱敏号码与本次要操作的号码一致。

不要：
- 先登 `xjfw.189.cn` 再指望商城免登录；实测仍会要求商城 SSO。
- 主动打开 `login.189.cn`；它只在桌面 UA 误触时出现。
- 读取或转述验证码输入值。

## 第三步：读账户与目录

### 话费专区

页面直接显示：

- 页头脱敏号码
- `可用积分: N 积分`
- `充值号码：<完整号码>`（仅用于核对，不写入公开证据）
- 面额列表与所需积分
- 兑换规则（纯积分、5–31 日、仅本机、不可退等）

实测样本（2026-10-21，浙江）：10 元/1000、20 元/2000、30 元/3000、50 元/5000、100 元/10000。**面额随账号/省份/渠道变化，必须每次读取。**

### 综合商城

首页显示姓名、脱敏号码、可用积分、本年到期积分、网龄。商品区可能有懒加载和广告浮层；等业务数据出现再读，不把初始 0 或空列表当最终结果。

搜索：
- 首页搜索是覆盖层：点 `.search-wrap` 打开，填 `input[placeholder="搜索商品名称"]`，点 `.close.click`（按钮文字为“搜索”）。
- 搜索会跳到 `home/commoditylist?KeyType=1&KeyWord=...`；首次可能空白，刷新或等待后读取。
- “我可兑换”页的搜索是另一个路由（`KeyType=7`），在实测账号上返回空列表，不能据此判断全站无商品。以首页搜索或分类为准。

### 分类

`全部分类` → `电信产品` 下只有应用类（天翼云盘、爱音乐、天翼云游戏、天翼视讯/超高清、其他）。话费走 `jycalls` 专区，不在该分类里。

## 第四步：下单

1. 在话费专区点选面额（如 `10元`），确认选中态和所需积分。
2. 点「兑换」**一次**。
3. 正常链路：创建订单 → 收银台。
4. 记录订单号、商品名、积分、现金。
5. 若页面没有明显变化，先读当前 URL 和页面阶段；不要连续点击。

### 下单接口（仅用于诊断，不作为默认路径）

页面自己的下单请求：

```text
POST https://apps.telefen.com/jycalls/commodity/order/createOrder
```

返回 `{data: {orderId, payOrderId, status, url}, isSuccess: true}`，`url` 为收银台地址。只有在 UI 已创建订单但丢失收银台地址时，才用页面上下文读取该响应或重新进入订单流程；不要把接口参数或商品 ID 写死到技能里。

## 第五步：收银台

正常页面：

```text
1000 积分
收款方: 电信积分商城
订单编号：...
积分支付
电信积分（可用积分：N）
本单支付 1000
确认支付
```

核对：
- 订单号与刚创建的一致
- 积分 = 所选面额所需积分
- 可用积分 ≥ 本单支付
- 现金 0（页面无现金项或显示 0）

点一次「确认支付」后，弹出：

```text
短信验证码
[输入框]
获取短信    确认支付
```

交接给本人完成。本人确认后：

- 成功：跳到 `jycalls/Success?...&result=true`
- 页面仍在收银台：只读判断，不重复提交
- 页面显示处理中：查订单，不再支付

## 失败签名与恢复

| 现象 | 含义 | 处理 |
|---|---|---|
| 点兑换后到 `jf.189.cn/pchome` | 桌面 UA 下收银台回跳，不是登录失效 | 停止点击；切移动 UA；查订单；只继续一笔 |
| `login.189.cn/web/login` | 触发了 PC 账号体系 | 不要登录；回到 `apps.telefen.com` 移动流程 |
| 订单页出现多笔「待支付」 | 重复点击造成 | 只支付一笔；其余「关闭订单」 |
| 收银台 `payflag` 打开即回首页 | payflag 过期或 UA 不对 | 重新走一次 UI 下单；不要复用旧 payflag |
| 支付后订单仍「待支付」 | 支付未完成或页面未刷新 | 刷新订单页；核对积分；不重复支付 |
| 订单「已完成」但权益未到 | 平台完成 ≠ 到账 | 查运营商/第三方明细，或标记用户确认 |

## 收尾

- 关闭重复待支付订单，保留一笔已完成。
- 记录脱敏号码、商品、积分、现金、订单号、状态、到账证据。
- 结束任务空间时不保留无关页面；需要用户查看结果时只保留成功页或订单页。
- 不把完整手机号、Cookie、Token、验证码写入公开文档或仓库。

## 证据强度

目录/详情可见 → 收银台可见 → 提交/支付结果 → 扣分核对 → 订单完成 → 运营商/第三方到账明细。

后面的证据没有取得时，不用前面的证据替代。分别报告“平台订单完成”和“权益已到账”。
