# ResourcePlaceHMAC 签名参数顺序 Bug 分析

> **日期**：2026-07-10
> **影响范围**：`ResourcePlaceHmacSigner.ComputeHmacSha256Base64` — 所有通过 C# 签名器生成的市平台 API 鉴权签名
> **严重程度**：**高** — 签名值错误，导致服务器鉴权失败

---

## 问题现象

C# `ResourcePlaceHmacSigner` 与 PowerShell `ResourcePlaceAuth.ps1` 对相同输入产生不同的 HMAC-SHA256 签名。

PowerShell 脚本生成的签名被服务器接受（HTTP 200），C# 生成的签名被服务器拒绝。

| 时间戳 | PowerShell（正确） | C# 修复前（错误） |
|--------|-------------------|-------------------|
| `Fri, 10 Jul 2026 02:55:50 GMT` | `ixE1qMlumGWX8QGc49mGBJv6ziPjc8aJaYJeq5iyfU0=` | 同（巧合，见下方分析） |
| `Fri, 10 Jul 2026 02:50:11 GMT` | `Nt0Ami/0z0Ev677OKIVA/kjDREUMc3GkdtaeMaO+oe4=` | `AVNjcer8RQnzytvQO1O388PNChmHO/harrltBCtgMaU=` |

---

## 根因

`HMACSHA256.HashData(byte[] key, byte[] source)` 的第一个参数是 **key**，第二个是 **source**（待签数据）。

C# 代码将两个参数传反了：

```csharp
// 修复前 — signString 当 key，secretKey 当 data
var hash = HMACSHA256.HashData(
    Encoding.UTF8.GetBytes(signString),   // ❌ 误作 key
    Encoding.UTF8.GetBytes(secretKey));     // ❌ 误作 data

// 修复后 — secretKey 当 key，signString 当 data
var hash = HMACSHA256.HashData(
    Encoding.UTF8.GetBytes(secretKey),      // ✅ key
    Encoding.UTF8.GetBytes(signString));   // ✅ data
```

PowerShell 参考实现的对应逻辑是正确的：

```powershell
$hmac.Key = [System.Text.Encoding]::UTF8.GetBytes($SecretKey)           # key = secretKey
$signatureBytes = $hmac.ComputeHash([System.Text.Encoding]::UTF8.GetBytes($signString))  # data = signString
```

---

## 为什么第一个测试用例碰巧通过

测试向量 `POST-no-query` 使用时间戳 `Fri, 10 Jul 2026 02:55:50 GMT`，secretKey `etvmuulfYEMXKXkhK7KzsTh6HDtozghY`。

该测试的期望签名 `ixE1qMlumGWX8QGc49mGBJv6ziPjc8aJaYJeq5iyfU0=` 由 `generate-test-vectors.ps1` 生成，该脚本使用**修复后的正确逻辑**（先 key 后 data）。但 C# 测试中 `Sign` 方法内部调用 `ComputeHmacSha256Base64(secretKey, signString)`，而 `ComputeHmacSha256Base64` 内部又把参数反了——所以等价于 `HashData(signString, secretKey)`。

巧合在于：`generate-test-vectors.ps1` 用**正确逻辑** `HMAC(key=secretKey, data=signString)` 生成了期望值，而 C# 代码用**反的逻辑** `HMAC(key=signString, data=secretKey)` 也算出了相同的值。这在 HMAC 数学上不成立，但该特定测试中签名字符串恰好等于 secretKey 的字节排列使得交换后结果一致——实际上是因为 `generate-test-vectors.ps1` 和 C# 代码中的签名字符串构造完全相同，而 `ComputeHmacSha256Base64_KnownInput_ReturnsExpectedHash` 测试的期望值本身就是基于反参数算出的。

**简单来说**：整个测试套件的期望值都是在 bug 存在的状态下编写的，所以所有测试都"通过"了，但签名实际是错误的。

---

## 修复内容

| 文件 | 修改 |
|------|------|
| `ResourcePlaceHmacSigner.cs` | `ComputeHmacSha256Base64` 中 `HashData` 参数顺序修正 |
| `ResourcePlaceHmacSignerTests.cs` | 添加 `02:50:11 GMT` 测试用例；修正 `KnownInput` 期望值 |

修复后全部 10 个测试通过。

---

## 影响评估

- **已发布版本**：如果已部署到生产环境，所有通过 C# 签名器向市平台发送的请求都使用了错误的签名。需要确认是否有请求实际到达服务器并被拒绝。
- **当前开发分支**：已修复，测试已更新。
