# 官方话费兑换浏览器流程

在登录、读面额、下单、收银台或恢复时读取。使用宿主浏览器能力；Ego 宿主遵循其 TaskSpace、后台操作和交接规则。

## 会话地图（先看这一节）

电信积分链路跨三套登录体系，登录态不通用：

| 域 | 作用 | 是否必需 |
|---|---|---|
| `apps.telefen.com` / `m.telefen.com` | 移动版话费专区、收银台入口 | ✅ 必需，登录一次 |
| `xjfw.189.cn` | 星级服务中心 | ❌ 兑话费不需要 |
| `jf.189.cn` / `login.189.cn` | PC 积分商城、PC 账号体系 | ❌ 最短路径不需要，仅用于查积分/订单 |

**结论：只登录 `apps.telefen.com` 对应的商城 SSO。** 先登星级中心或 PC 账号都会增加一次短信，且不会让话费页免登录。

## 入口

- 话费兑换专区：<https://apps.telefen.com/jycalls/?channelCode=HGOKHD>
- 商城登录：`m.telefen.com/MobileSSOv2/Login.aspx?appcode=HGOKHD&goBack=...`
- 收银台：`cashier.telefen.com/dopay?payflag=...` → `cashier.telefen.com/ssocallback?...`
- 成功页：`apps.telefen.com/jycalls/Success?...&result=true`
- 积分：<https://jf.189.cn/selfcenter/myPoints.aspx>
- 订单：<https://jf.189.cn/selfcenter/order.aspx>

`channelCode=HGOKHD` 是官方移动商城渠道参数；不要写死 `promoid`、`payflag`、`Token` 或面额商品 ID。

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

其他宿主：使用设备模拟（iPhone / 390×844 / mobile）或在移动端浏览器中打开。不要在桌面窗口直接操作话费收银台。

## 第二步：登录

1. 打开话费专区。
2. 未登录会跳 `m.telefen.com/MobileSSOv2/Login.aspx`，页面为手机号 + 短信验证码 + 登录。
3. 交接当前页，由本人填手机号、获取并填写短信码。
4. 登录后回到 `apps.telefen.com/jycalls` 或商城首页。
5. 核对页头脱敏号码与本次要充值的号码一致。

不要：

- 先登 `xjfw.189.cn` 再指望话费页免登录；实测仍会要求商城 SSO。
- 主动打开 `login.189.cn`；它只在桌面 UA 误触时出现。
- 读取或转述验证码输入值。

## 第三步：读账户和面额

话费专区直接显示：

```text
173****2062
可用积分: 1836 积分
充值号码：173****2062
10元  1000积分
20元  2000积分
...
兑换
```

读取：

- 脱敏号码与充值号码
- 可用积分
- 全部面额及所需积分
- 兑换规则（纯积分、5–31 日、仅本机、不可退等）

页面初始的空值或 0 可能是占位；等身份、积分和面额都加载完再判断。合法余额确实为 0 时如实报告。

## 第四步：下单

1. 点选面额（如 `10元`），确认选中态和所需积分。
2. 点「兑换」**一次**。
3. 正常链路：创建订单 → 收银台。
4. 记录订单号、面额、积分、现金。
5. 若页面没有明显变化，先读当前 URL 和页面阶段；不要连续点击。

### 下单接口（仅用于诊断，不作为默认路径）

页面自己的下单请求：

```text
POST https://apps.telefen.com/jycalls/commodity/order/createOrder
```

返回 `{data: {orderId, payOrderId, status, url}, isSuccess: true}`，`url` 为收银台地址。只有在 UI 已创建订单但丢失收银台地址时，才用页面上下文读取该响应；不要把接口参数或商品 ID 写死到技能里。

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
| 订单「已完成」但话费未到 | 平台完成 ≠ 到账 | 查 189.cn 充值记录/余额，或标记用户确认 |

## 收尾

- 关闭重复待支付订单，保留一笔已完成。
- 记录脱敏号码、面额、积分、现金、订单号、状态、到账证据。
- 结束任务空间时不保留无关页面；需要用户查看结果时只保留成功页或订单页。
- 不把完整手机号、Cookie、Token、验证码写入公开文档或仓库。

## 证据强度

话费专区可见 → 收银台可见 → 提交/支付结果 → 扣分核对 → 订单完成 → 189.cn 充值记录或用户确认。

后面的证据没有取得时，不用前面的证据替代。分别报告“平台订单完成”和“话费已到账”。
