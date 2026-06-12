# Electron-BC 缓存策略文档

本文档详细描述 Electron-BC 项目的资源缓存策略，并逐一对照源代码进行说明。

---

## 1. 架构总览

Electron-BC 通过 **自定义协议拦截 + LevelDB 持久化存储** 实现了一套完整的资源缓存系统。其核心流程为：

1. 应用注册自定义 `https` 协议处理器（`MyProtocol`）。
2. 所有 HTTPS 请求经过协议处理器，符合条件的资源请求被路由到 `AssetCache`。
3. `AssetCache` 在 LevelDB 数据库中按 `key + version` 查找缓存，命中则直接返回，未命中则从网络获取并存入缓存。
4. 应用还支持批量预加载缓存（preload cache）、缓存迁移、缓存清理等管理功能。

```
HTTPS 请求
  │
  ▼
MyProtocol.httpsHandler()       ← src/main/protocol.ts:36
  │
  ├─ BCResourceHandler          ← src/main/request/index.ts:13
  ├─ EchoResourceHandler        ← src/main/request/index.ts:53
  ├─ MPAResourceHandler         ← src/main/request/index.ts:113
  │
  ▼
AssetCache.requestAsset()       ← src/main/AssetCache/database.ts:84
  │
  ├─ LevelDB 缓存命中 → 直接返回
  └─ 缓存未命中 → net.fetch() → 存入 LevelDB → 返回
```

---

## 2. 协议拦截层

### 2.1 协议注册

**源码位置：** `src/main/protocol.ts:61-62`

```typescript
protocol.handle('ebc', req => this.ebcHandler(req))
protocol.handle('https', req => this.httpsHandler(req))
```

应用在 Electron 中拦截了所有 `https` 协议请求。每个 HTTPS 请求先经过 `httpsHandler`，依次尝试匹配三类资源处理器，若均不匹配，则透传给 `net.fetch` 做常规网络请求。

### 2.2 HTTPS 请求路由

**源码位置：** `src/main/protocol.ts:36-54`

```typescript
private async httpsHandler(request: Request) {
    const bc_res_ctx = this.bcResource.willHandle(request);
    if (bc_res_ctx) return this.bcResource.handle(request, bc_res_ctx);

    const echo_res_ctx = this.echoResource.willHandle(request);
    if (echo_res_ctx) return this.echoResource.handle(request, echo_res_ctx);

    const mpa_res_ctx = this.mpaResource.willHandle(request);
    if (mpa_res_ctx) return this.mpaResource.handle(request, mpa_res_ctx);

    return net.fetch(request, { bypassCustomProtocolHandlers: true, redirect: "follow" });
}
```

路由策略为**优先级链**：BC 资源 → Echo Mod 资源 → MPA 资源 → 直接网络请求。

---

## 3. 资源处理器（Resource Handlers）

三个资源处理器均定义在 `src/main/request/index.ts`，实现了 `ResourceHandler` 接口。

### 3.1 BCResourceHandler —— BC 核心资源缓存

**源码位置：** `src/main/request/index.ts:13-51`

**匹配规则（`willHandle`）：**
- 请求 URL 以已知 BC 服务器地址为前缀。
- 资源文件扩展名为 `.png`、`.jpg`、`.mp3`、`.txt`、`.ttf` 之一。

```typescript
willHandle(request: Request) {
    if (this.bcStatus && this.bcStatus.some(s => request.url.startsWith(s.url))) {
        const assetKey = this.assetKey(request);
        const ext = ...;
        if (['.png', '.jpg', '.mp3', '.txt', '.ttf'].includes(ext))
            return { assetKey };
    }
    return false;
}
```

**缓存键：** URL 中 `BondageClub/` 之后的路径部分（如 `Assets/Female3DCG/...`）。

**缓存版本：** 使用 BC 版本号（如 `R116`），来自 `bcStatus[0].version`。

**`permanent` 参数：** `false`（非永久缓存）—— 版本更新后会重新获取。

### 3.2 EchoResourceHandler —— Echo Mod 资源缓存

**源码位置：** `src/main/request/index.ts:53-111`

**匹配规则（`willHandle`）：**
- 请求 URL 匹配以下两种模式：
  - `https://sugarchain-studio.github.io/echo-{clothing,activity}-ext/`（通过 `?v=` 参数提取版本）
  - `https://cdn.jsdelivr.net/gh/SugarChain-Studio/echo-{clothing,activity}-ext@`（通过 URL 路径中的 commit SHA 提取版本）
- 资源文件扩展名为 `.png`、`.jpg`、`.webp` 之一。

**缓存键：** `EchoMod://{resource}`，加了 `EchoMod://` 前缀以区分命名空间。

**缓存版本：** 从 URL 中的 `?v=` 查询参数或 `@{sha}` 路径段提取。

**`permanent` 参数：** `true` —— Echo Mod 的资源版本由 commit SHA 或查询参数精确标识，因此设为永久缓存（`Cache-Control: public, max-age=31536000, immutable`）。

### 3.3 MPAResourceHandler —— MPA Mod 资源缓存

**源码位置：** `src/main/request/index.ts:113-137`

**匹配规则（`willHandle`）：**
- 请求 URL 以 `https://mayathefoxy.github.io/MPA/assets/` 开头。
- 文件扩展名为 `.mp3`。

**缓存键：** `MPA://{full_url}`。

**缓存版本：** 使用文件名（URL 最后一段路径）作为版本标识。

**`permanent` 参数：** `true`。

---

## 4. 核心缓存引擎（AssetCache）

### 4.1 模块入口

**源码位置：** `src/main/AssetCache/index.ts`

`AssetCache` 类作为静态工具类，聚合了所有缓存操作：

```typescript
export class AssetCache {
    static requestAsset = requestAsset;
    static clearCache = clear;
    static cacheDir = getCachePath;
    static fileSizeStr = fileSizeStr;
    static clearSizeResult = clearSizeResult;
    static preloadCache = preloadCache;
    static canPreloadCache = canPreloadCache;
    static relocate = relocate;
    static available = available;
    static init = initAccess;
}
```

### 4.2 存储后端 —— LevelDB (classic-level)

**源码位置：** `src/main/AssetCache/database.ts`

使用 [`classic-level`](https://github.com/Level/classic-level)（LevelDB 的 Node.js 绑定）作为持久化 KV 存储。

#### 数据结构

```typescript
interface CacheItem {
    base64Data: string;       // 资源内容（Base64 编码）
    version: string;          // 资源版本标识
    type: string | null;      // Content-Type（MIME 类型）
    cacheTime: number;        // 缓存时间戳（Date.now()）
}
```

#### 数据库初始化

**源码位置：** `src/main/AssetCache/database.ts:20-32`

```typescript
function createDatabase() {
    return new ClassicLevel<string, CacheItem>(getCachePath(), {
        valueEncoding: 'json',
    });
}

export async function initAccess() {
    access = new PendingAccess(createDatabase());
    await access.aquire();
}
```

初始化时机在 `src/main.ts:27`：

```typescript
AssetCache.init();
```

此调用在 `app.whenReady()` 回调中执行，确保在协议处理器注册之后但在创建主窗口之前完成。

#### 并发访问控制 —— PendingAccess

**源码位置：** `src/main/utility.ts:10-48`

`PendingAccess<T>` 是一个简单的异步锁/队列机制，确保对 LevelDB 实例的访问在数据库就绪后才能进行：

- `aquire()`：若数据库已就绪则直接返回，否则将调用者排入等待队列。
- `invalidate()`：在数据库迁移时将实例置为 `undefined`，后续的 `aquire()` 调用会被阻塞。
- `release(obj)`：将新的数据库实例注入并唤醒所有等待者。

### 4.3 缓存读写流程（requestAsset）

**源码位置：** `src/main/AssetCache/database.ts:84-107`

这是缓存的核心逻辑：

```typescript
export async function requestAsset(
    url: string,
    key: string,
    version: string,
    permanent = false
): Promise<CachedResponse> {
    const db = await access!.aquire();
    const data = await db.get(key);
    if (data && data.version === version) {
        // 缓存命中：从 Base64 还原内容，构造 Response 返回
        const content = new Blob([Buffer.from(data.base64Data, 'base64')]);
        return {
            content, type: data.type, version: data.version,
            response: createResponse(content, data.type, data.version, permanent),
        };
    } else {
        // 缓存未命中或版本不匹配：从网络获取
        const { content, type, response } = await fetchAsset(url);
        if (response.status === 200) {
            storeAsset(key, version, Buffer.from(await content.arrayBuffer()), type);
        }
        return { content, type, response };
    }
}
```

**缓存失效策略：** 基于版本号（`version`）。当请求的版本与缓存中的版本不一致时视为缓存失效，重新从网络获取并覆盖旧缓存。

**注意事项：**
- `storeAsset` 是异步的且不 `await`，即写入缓存是 fire-and-forget，不阻塞请求返回。
- 仅在网络请求返回 HTTP 200 时才写入缓存。

### 4.4 缓存响应构造（createResponse）

**源码位置：** `src/main/AssetCache/database.ts:64-82`

```typescript
function createResponse(content, type, version, permanent = false) {
    const headers: HeadersInit = {
        'Content-Length': content.size.toString(),
    };
    if (type) headers['Content-Type'] = type;
    if (version) {
        headers['X-Ebc-Cache'] = 'HIT';               // 标识缓存命中
        headers['X-Ebc-Cache-Version'] = version;      // 标识缓存版本
        if (permanent) {
            headers['Cache-Control'] = 'public, max-age=31536000, immutable';
        }
    }
    return new Response(content, { status: 200, statusText: 'OK', headers });
}
```

自定义 HTTP 头：
- `X-Ebc-Cache: HIT` —— 标识此响应来自本地缓存。
- `X-Ebc-Cache-Version` —— 标识缓存的版本。
- 对于 `permanent` 资源，额外设置 `Cache-Control: public, max-age=31536000, immutable`（1 年不过期）。

### 4.5 网络请求（fetchAsset）

**源码位置：** `src/main/AssetCache/database.ts:54-62`

```typescript
export async function fetchAsset(url: string): Promise<CachedResponse> {
    const response = await net.fetch(url, { bypassCustomProtocolHandlers: true });
    const clonedResponse = response.clone();
    return {
        content: await response.blob(),
        type: response.headers.get('Content-Type'),
        response: clonedResponse,
    };
}
```

使用 `bypassCustomProtocolHandlers: true` 绕过自身注册的协议处理器，避免递归拦截。

---

## 5. 缓存路径管理

**源码位置：** `src/main/AssetCache/cachePath.ts`

### 5.1 默认路径

```typescript
const defaultCachePath = path.join(
    app.getPath("appData"),
    "Bondage Club",
    "AssetCache"
);
```

在典型的 Windows 系统上为 `%APPDATA%/Bondage Club/AssetCache`。

### 5.2 路径读取

**源码位置：** `src/main/AssetCache/cachePath.ts:16-27`

```typescript
export function getCachePath() {
    if (cachePath) return cachePath;
    const setting = settings.getSync(SettingTag);
    if (typeof setting === "string") {
        cachePath = setting;
    } else {
        cachePath = defaultCachePath;
        settings.set(SettingTag, cachePath);
    }
    return cachePath;
}
```

路径优先从 `electron-settings` 中读取（`SettingTag = "CachePath"`），若未设置则使用默认路径并保存。

### 5.3 路径迁移（relocate）

**源码位置：** `src/main/AssetCache/cachePath.ts:33-79` 和 `src/main/AssetCache/database.ts:119-129`

迁移流程：
1. 关闭当前 LevelDB 实例（`access.invalidate()` + `olddb.close()`）。
2. 如果新目录为空，询问用户是否要迁移旧缓存数据。
3. 同设备迁移使用 `rename`（原子操作），跨设备迁移使用逐文件 `copyFile` + 删除旧目录。
4. 更新设置中的路径，创建新的 LevelDB 实例（`access.release(createDatabase())`）。

### 5.4 缓存大小查询

**源码位置：** `src/main/AssetCache/cachePath.ts:83-104`

```typescript
export function fileSizeStr() {
    if (cachedSizeResult) return cachedSizeResult;
    // 遍历缓存目录计算总大小
    const size = fs.readdirSync(cachePath, { withFileTypes: true })
        .reduce((size, file) => {
            if (file.isFile()) size += fs.statSync(...).size;
            return size;
        }, 0);
    cachedSizeResult = formatSize(size);
    return cachedSizeResult;
}
```

大小计算结果会被缓存在内存变量 `cachedSizeResult` 中，用户可通过菜单点击刷新（调用 `clearSizeResult()`）。

---

## 6. 预加载缓存（Preload Cache）

**源码位置：** `src/main/AssetCache/preloadCache.ts`

### 6.1 预加载机制

预加载功能用于在后台批量下载并缓存资源，避免用户在游戏中首次加载资源时的延迟。

```typescript
export async function preloadCache(url_prefix: string, version: string) {
    if (preloadCacheRunning) return;
    preloadCacheRunning = true;

    // 从 resource/preload.data 读取并解压资源清单
    const preloadData = await fs.promises.readFile(preloadDataPath, { encoding: null });
    const content = JSON.parse(LZString.decompressFromUint8Array(preloadData)) as Dir;

    // 遍历资源清单，逐个检查缓存并按需下载
    // processList.reverse() 确保 'Assets' 目录最后处理
    while (processList.length > 0) {
        // 对每个资源：检查 LevelDB 中是否已有该版本的缓存
        // 若无，则通过 fetchAsset 下载并 storeAsset 存入
    }
    preloadCacheRunning = false;
}
```

**资源清单：** `resource/preload.data` 文件，是一个使用 `lz-string` 压缩的 JSON 目录树，描述了所有需要预加载的资源路径。

**并发控制：** 使用 `preloadCacheRunning` 布尔值防止重复启动。

**处理顺序：** `processList.reverse()` 使得非 `Assets` 目录优先处理，`Assets` 目录（通常最大）最后处理。

### 6.2 版本检查与自动预加载

**源码位置：** `src/main/AssetCache/preloadCache.ts:83-88` 和 `src/main/mainWindow.ts:28-43`

```typescript
// preloadCache.ts
export async function checkCacheVersion(bcVer: BCVersion) {
    const currentVersion = await cacheVersion();
    const ret = !currentVersion || currentVersion !== bcVer.version;
    saveCacheVersion(bcVer.version);
    return ret;
}
```

在主窗口加载完成后（`src/main/mainWindow.ts:28-43`），应用会自动检查 BC 版本是否发生变化：

```typescript
readyState.loaded().then(async () => {
    const bcVersion = BCURLPreference.choice;
    const shouldUpdate = await checkCacheVersion(bcVersion);
    if (shouldUpdate) {
        MyPrompt.confirmCancel(..., () => {
            AssetCache.preloadCache(bcVersion.url, bcVersion.version);
        });
    }
});
```

若版本发生变化，会弹窗询问用户是否更新缓存。

---

## 7. 页面加载时的缓存控制

**源码位置：** `src/handler.ts:36-38`

```typescript
reload() {
    this.webContents.loadURL(BCURLPreference.choice.url, {
        extraHeaders: 'pragma: no-cache\n',
    });
}
```

手动刷新页面时，会添加 `pragma: no-cache` 头，指示服务器返回最新内容（避免 HTTP 层级的缓存）。

---

## 8. 版本信息获取（无缓存策略）

**源码位置：** `src/loading/fetchLatestBC.ts:39-42`

```typescript
net.fetch('https://bondageprojects.com/club_game/', {
    bypassCustomProtocolHandlers: true,
    cache: 'no-store',
})
```

获取 BC 最新版本信息时，使用 `cache: 'no-store'` 确保每次都从服务端获取最新版本数据，不使用任何缓存。

---

## 9. 用户界面（缓存管理菜单）

**源码位置：** `src/main/AppMenu/cache.ts`

应用在「工具」菜单中提供了以下缓存管理选项：

| 菜单项 | i18n Key | 功能 | 源码行号 |
|--------|----------|------|----------|
| 打开缓存目录 | `MenuItem::Tools::OpenCacheDir` | 在文件管理器中打开缓存目录 | cache.ts:14-18 |
| 迁移缓存目录 | `MenuItem::Tools::RelocateCacheDir` | 将缓存目录迁移到新位置 | cache.ts:19-55 |
| 开始预加载缓存 | `MenuItem::Tools::StartUICacheUpdate` | 手动触发批量预加载 | cache.ts:56-71 |
| 查看缓存大小 | `MenuItem::Tools::ProximateCacheSize` | 显示缓存目录大小，点击刷新 | cache.ts:72-80 |
| 清空缓存 | `MenuItem::Tools::ClearCache` | 清空所有缓存数据（需确认） | cache.ts:81-97 |

---

## 10. 总结：缓存策略要点

| 维度 | 策略 |
|------|------|
| **存储后端** | LevelDB（`classic-level`），KV 存储，值为 JSON 序列化的 `CacheItem` |
| **缓存粒度** | 单个资源文件 |
| **缓存键** | 资源路径（BC）/ `EchoMod://` 前缀路径（Echo）/ `MPA://` 前缀 URL（MPA） |
| **失效策略** | 基于版本号比对（version-based invalidation） |
| **缓存范围** | `.png`, `.jpg`, `.mp3`, `.txt`, `.ttf`（BC）; `.png`, `.jpg`, `.webp`（Echo）; `.mp3`（MPA） |
| **永久缓存** | Echo Mod 和 MPA 资源标记为永久（`immutable`, 1 年 max-age）；BC 资源不标记为永久 |
| **预加载** | 支持从预定义资源清单批量预加载，版本变更时自动提示用户 |
| **写入策略** | Fire-and-forget（不阻塞请求返回） |
| **编码方式** | Base64 编码存储资源内容 |
| **并发控制** | `PendingAccess` 异步队列保证数据库访问安全 |
| **路径可配置** | 支持自定义缓存目录，支持跨设备迁移 |
| **缓存清理** | 支持手动清空全部缓存（`db.clear()`） |
| **版本获取** | 每次启动不缓存版本信息（`cache: 'no-store'`） |
