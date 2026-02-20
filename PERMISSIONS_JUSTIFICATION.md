# Chrome Web Store 权限使用说明文档

本文档详细说明 HTTP Status 扩展所需的各项权限及其使用理由，用于 Chrome Web Store 审核提交。

---

## 🔐 权限清单

HTTP Status 扩展请求以下权限：

```json
{
  "permissions": [
    "webRequest",
    "storage",
    "tabs"
  ],
  "host_permissions": [
    "http://*",
    "https://*"
  ]
}
```

---

## 📋 详细权限说明

### 1️⃣ webRequest（网络请求监听）

#### **为什么需要此权限？**
这是扩展的**核心功能**所必需的权限。HTTP Status 的主要用途是显示网页的 HTTP 状态码，必须通过 `webRequest` API 监听 HTTP 响应才能获取状态码信息。

#### **具体用途**
- **监听 HTTP 响应**: 使用 `chrome.webRequest.onCompleted` 和 `chrome.webRequest.onResponseStarted` 监听网络请求完成事件
- **读取状态码**: 从响应中提取 HTTP 状态码（如 200、301、404、500）
- **读取响应头**: 获取 HTTP 响应头信息（如 Content-Type、Server、Location）
- **追踪重定向**: 使用 `chrome.webRequest.onBeforeRedirect` 监听重定向链路

#### **权限范围限制**
- ✅ **仅读取**响应元数据（状态码、响应头）
- ❌ **不读取**请求或响应的正文内容
- ❌ **不修改**任何网络请求或响应
- ❌ **不拦截**或阻止任何网络请求

#### **代码示例**
```typescript
// 监听请求完成，获取状态码
chrome.webRequest.onCompleted.addListener(
  (details) => {
    const statusCode = details.statusCode; // 仅读取状态码
    const responseHeaders = details.responseHeaders; // 仅读取响应头
    // 不访问 details.requestBody 或响应正文
  },
  { urls: ["<all_urls>"] },
  ["responseHeaders"]
);
```

#### **单一用途说明**
此权限的使用**完全符合扩展的单一用途**：在地址栏显示 HTTP 状态码。没有此权限，扩展无法获取 HTTP 响应信息，也就无法实现核心功能。

---

### 2️⃣ storage（本地存储）

#### **为什么需要此权限？**
用于保存用户的扩展配置和偏好设置，提供个性化的使用体验。

#### **具体用途**
- **保存用户设置**: 存储用户的显示偏好（如是否显示 IP、是否启用调试模式）
- **缓存扩展状态**: 存储扩展的版本信息，用于升级时的数据迁移
- **持久化数据**: 保存用户的配置，避免每次打开浏览器都需要重新设置

#### **存储的数据类型**
- 扩展版本号（用于版本升级检测）
- 用户偏好设置（显示选项、调试模式等）
- 无其他数据

#### **权限范围限制**
- ✅ **仅存储**扩展相关的配置数据
- ❌ **不存储**用户的浏览历史或访问记录
- ❌ **不存储**任何个人身份信息
- ❌ **不同步**数据到任何外部服务器

#### **代码示例**
```typescript
// 保存扩展设置
await chrome.storage.local.set({
  version: '2.0.0',
  debugMode: true
});

// 读取扩展设置
const data = await chrome.storage.local.get(['version', 'debugMode']);
```

#### **隐私说明**
所有存储的数据**完全在用户本地浏览器中**，不会上传到任何服务器。用户卸载扩展后，所有数据会自动删除。

---

### 3️⃣ tabs（标签页访问）

#### **为什么需要此权限？**
需要获取当前标签页的基本信息（如 Tab ID 和 URL），以便将正确的 HTTP 状态码显示在对应的标签页图标上。

#### **具体用途**
- **获取标签页 ID**: 使用 `chrome.tabs.query` 获取当前活动标签页的 ID
- **更新扩展图标**: 使用 `chrome.action.setIcon` 和 `chrome.action.setBadgeText` 在对应标签页的地址栏显示状态码
- **获取标签页 URL**: 用于判断是否为有效的 HTTP/HTTPS 页面（避免在 `chrome://` 等内部页面上显示）

#### **权限范围限制**
- ✅ **仅读取**标签页的基本元数据（ID、URL）
- ❌ **不读取**标签页的内容或 DOM
- ❌ **不修改**标签页的内容
- ❌ **不注入**任何脚本到标签页

#### **代码示例**
```typescript
// 获取当前标签页
const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
const tabId = tab.id;
const tabUrl = tab.url;

// 在对应标签页显示状态码
chrome.action.setBadgeText({
  text: '200',
  tabId: tabId
});
```

#### **单一用途说明**
此权限用于确保 HTTP 状态码显示在**正确的标签页**上。如果没有此权限，扩展无法区分不同标签页，也就无法准确显示每个页面的状态。

---

### 4️⃣ host_permissions（主机权限）

#### **为什么需要 `<all_urls>` 权限？**
由于用户可能访问**任何网站**，扩展需要在**所有网站**上监听 HTTP 响应，才能正确显示状态码。

#### **具体用途**
- **监听所有 HTTP/HTTPS 请求**: 使用 `webRequest` API 监听用户访问的任何网站的 HTTP 响应
- **支持所有域名**: 确保扩展在任何网站上都能正常工作

#### **权限范围限制**
```json
"host_permissions": [
  "http://*",   // 支持 HTTP 网站
  "https://*"   // 支持 HTTPS 网站
]
```

- ✅ **仅监听** HTTP 响应的元数据（状态码、响应头）
- ❌ **不访问**网页内容或 DOM
- ❌ **不修改**网页内容
- ❌ **不注入**任何代码到网页

#### **为什么不能使用更窄的权限？**
- 用户访问的网站是**不可预测的**（可能是 `example.com`、`github.com`、`google.com` 等任何网站）
- 如果只请求特定域名的权限，扩展将**无法在其他网站上工作**
- HTTP Status 是一个**通用开发者工具**，需要在所有网站上提供相同的功能

#### **类比其他类似扩展**
其他同类扩展（如 Redirect Path、HTTP Header Live 等）也都需要 `<all_urls>` 权限，这是此类工具的**行业标准**。

---

## 🎯 扩展的单一用途说明

**HTTP Status 的单一用途是：在浏览器地址栏显示当前网页的 HTTP 状态码。**

所有请求的权限都**直接服务于这一核心功能**：

1. **webRequest** - 获取 HTTP 状态码和响应头（核心功能）
2. **storage** - 保存用户设置（增强用户体验）
3. **tabs** - 将状态码显示在正确的标签页上（功能必需）
4. **host_permissions** - 在所有网站上工作（功能范围）

扩展**不包含**以下功能：
- ❌ 广告拦截
- ❌ 内容过滤
- ❌ 数据收集或分析
- ❌ 第三方服务集成
- ❌ 社交媒体功能
- ❌ 页面内容修改

---

## 🔒 隐私与安全保证

### 数据处理原则
- ✅ **100% 本地处理** - 所有数据仅在用户浏览器中处理
- ✅ **零数据上传** - 不连接任何外部服务器
- ✅ **零追踪代码** - 不使用任何分析或追踪工具
- ✅ **零第三方依赖** - 不使用任何外部库或 SDK

### 开源透明
- ✅ **完整源代码开放** - https://github.com/maicong/HTTP-Status
- ✅ **MIT 开源许可** - 允许任何人审查和验证代码
- ✅ **无混淆代码** - 所有代码清晰可读

### 用户控制
用户可以随时：
- 卸载扩展（所有数据自动删除）
- 撤销权限（在浏览器扩展管理页面）
- 审查源代码（GitHub）

---

## 📄 相关文档

- **隐私政策**: 参见 `privacy-policy.html`
- **商店描述**: 参见 `STORE_DESCRIPTION.md`
- **源代码**: https://github.com/maicong/HTTP-Status

---

## 🆘 审核团队联系方式

如果 Chrome Web Store 审核团队对任何权限有疑问，欢迎通过以下方式联系：

- **GitHub Issues**: https://github.com/maicong/HTTP-Status/issues
- **项目主页**: https://github.com/maicong/HTTP-Status

我们承诺在 24 小时内回复审核团队的所有问题。

---

## ✅ 合规性检查清单

- [x] 所有权限都有明确的使用说明
- [x] 权限使用符合扩展的单一用途
- [x] 不收集任何用户个人信息
- [x] 不使用混淆代码
- [x] 不连接外部服务器
- [x] 提供完整的隐私政策
- [x] 源代码完全开源
- [x] 遵循 Manifest V3 规范
- [x] 遵守 Chrome Web Store 政策

---

**最后更新**: 2026年2月16日
**扩展版本**: 2.0.0
**文档版本**: 1.0
