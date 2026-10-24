# MYKIT fork

这是 `tauri-plugin-iap` 的自用 fork，**只为修 Windows 上的两个真实缺陷 + 加诊断日志**。
除 `src/windows.rs` 外不碰上游代码，不新增/修改任何 IPC 命令名和参数。

| 项 | 值 |
|---|---|
| 上游 | https://github.com/Choochmeque/tauri-plugin-iap |
| 基线 commit | `275cfceb2bcef90e958a30782fdca7bedb1748b4`（= crates.io `0.10.0-rc.8`） |
| 分支 | `mykit` |
| 标签 | `mykit-v0.10.0-rc.8.1` |
| crate 版本号 | 保持 `0.10.0-rc.8`（**故意不改**，见下） |

## 为什么版本号不改名

crate 名、`links`、版本号全部保持上游原样，这样：

1. JS 侧 `@choochmeque/tauri-plugin-iap-api@0.10.0-rc.8` **可以继续用官方 npm 包**，不需要我们发布 npm 包
   （改动全在 Rust 内部实现，IPC 命令名/参数/返回结构没变）；
2. `permissions/` 与 capability 里的 `iap:default` 不用改；
3. `cargo tree` 里版本号仍显示 `0.10.0-rc.8`，具体来自哪个源由 `Cargo.lock` 的 `source` 字段区分。

唯一需要留意的是：**同一个进程里不能同时出现两个 `tauri-plugin-iap`**（`links = "tauri-plugin-iap"` 限制）。
每个 App 是独立 workspace，天然满足。

## 怎么引用

```toml
tauri-plugin-iap = { git = "https://github.com/<OWNER>/tauri-plugin-iap", tag = "mykit-v0.10.0-rc.8.1" }
```

想锁定到具体 commit（防止 tag 被移动）也可以直接写 `rev = "..."`。

> 首次构建需要能访问 GitHub（cargo 会 clone 到 `~/.cargo/git/`）。之后靠 `Cargo.lock` 固定 rev，
> 同一台机器不再需要网络。

## 版本配套（一个坑）

基线锁定 `windows = "0.61"`。tauri 从 2.12 起会拉 `windows 0.62`，两个版本的 `HWND` 是不同类型，
`IInitializeWithWindow::Initialize` 会报类型不匹配。所以：

- **App 侧 tauri 保持在 2.11.x（解析到 `windows 0.61.3`）时完全正常**；
- 把 tauri 升到解析出 `windows 0.62` 的版本之前，先升级本 fork（上游 main 已经用
  `.get()` → `.join()` 跟进 0.62，可以照抄）。

不要在不同 App 里一边升 tauri 一边留着旧 fork 不升级，那样报错信息（`expected HWND, found HWND`）
会非常难懂。

## 改了什么

### 1. `restore_purchases` 漏掉开发者管理的消耗品（行为修复）

开发者管理的消耗品（如积分包）在 Microsoft Store 里是 **collection item，不是 license**，
`StoreContext::GetAppLicenseAsync()?.AddOnLicenses()` **永远列不出来**。

后果是一条完整的坏链路：

```
积分包买过一次 → 不在 AddOnLicenses → restore 拿不到 itemId
             → 后端无法 ReportConsumableFulfillmentAsync
             → Store 认为「已拥有但未履行」→ 再也买不了第二次
```

修法：`restore_purchases` 在 `product_type == "inapp"` 时追加
`GetUserCollectionAsync(["UnmanagedConsumable", "Consumable"])`，把已购未履行的消耗品补进结果。

### 2. `mint_store_id_key` 可能永久挂起（行为修复）

原本是 `context.GetCustomerPurchaseIdAsync(..).and_then(|op| op.get())`。
`IAsyncOperation::get()` 是**同步阻塞等待**，而 Store 的完成回调依赖有消息泵的线程 ——
Tauri 的 async command 跑在 tokio 工作线程上，**实测出现过永久不返回**：
前端 `invoke` 一直 pending（恢复购买转圈、订阅同步根本发不出去）。

修法：改成 `Status()` 轮询 + 20s 截止 + `Cancel()`，超时返回可识别的错误码
`storeIdentityTimeout` / `storeIdentityCanceled` / `storeIdentityFailed`，交给前端降级，而不是卡死。

### 3. crate 不自包含：`Win32_UI_Shell` feature 没开（构建修复）

`src/windows.rs` 用了 `Win32::UI::Shell::IInitializeWithWindow`（给 Store 购买弹窗指定父窗口），
但上游 `Cargo.toml` 的 `windows` 依赖只开了 `Foundation` / `Foundation_Collections` /
`Services_Store` / `System`，**没开 `Win32_UI_Shell`**。

结果：这个 crate 只能在使用方**恰好也**依赖 `windows` 并开启 `Win32_UI_Shell` 时编译通过，
否则报 `cannot find Win32 in windows`。属于上游的隐性耦合。

修法：在 fork 的 manifest 里补上 `"Win32_UI_Shell"`。只加 feature，不改行为。

### 4. `IAP_DIAG` 诊断日志（无行为变更）

`query_associated_products` / `get_products` / `mint_store_id_key` / `restore_purchases`
打印请求了哪些 id、命中了哪些、每个商品的价格原值（`""` 与 `<err 0x..>` 可区分）、
mint 前后的长度与是否为空。

这几条日志的价值在于把「查不到」拆成三种可区分的情况：
**商店没返回商品** / **返回了但没匹配上** / **匹配上了但价格是空串**。
没有它们，这三种在调用方看起来一模一样。

## 上游 rc.9 / rc.10 / main 到底改了什么（2026-10-24 逐文件核对）

**结论：rc.8 之后没有任何 Windows 逻辑修复。没有升级的必要，升级反而引入新风险。**

| 对比 | `src/windows.rs` | 其它文件 |
|---|---|---|
| rc.8 → rc.9 | 只有 `#[allow(unknown_lints, clippy::unused_async_trait_impl)]` 属性调整，**逻辑零改动** | lint / 注释 |
| rc.9 → rc.10 | **零改动**（`diff` 无任何输出） | 删掉废弃的 `initialize` 命令（`commands.rs` / `lib.rs` / `models.rs`）；`tauri = "2.11.4"` → `"2"`；`base64 0.22` → `0.23`；`rust-version 1.90` → `1.93` |
| rc.10 → `main` | 全部 `.get()` → `.join()`，**这只是 windows crate 0.62 的 API 改名，语义仍是同步阻塞**，我们实测的挂起 bug 原样存在 | 依赖升级、CI |

也就是说，「积分包不进 `AddOnLicenses`」和「mint 可能永久挂起」两条，在 rc.10 和最新 `main` 上都**没修**。

⚠️ **不要升 rc.10**：它把 `tauri = "2.11.4"` 放宽成 `tauri = "2"`，更容易解析到 tauri 2.12 →
`windows 0.62`，而它自己的 `windows` 依赖仍写 `"0.61"`，直接踩 `HWND` 类型不匹配。
rc.8 的 pin 反而是对的。

## 与上游同步

基线固定在上游 commit `275cfceb`（= crates.io rc.8，实测通过的那一版）。

跟进上游的做法（`mykit` 只有 3 个提交，rebase 即可）：

```bash
git fetch upstream
git rebase --onto upstream/main 275cfceb mykit
# 冲突只会落在 src/windows.rs 的两处改动 + Cargo.toml 的 feature 列表上
```

**如果要跟 `main`（windows 0.62），额外要做**：把剩余的 `.get()` 全换成 `.join()`
（`GetAppLicenseAsync` ×2、`GetAssociatedStoreProductsAsync` ×1、`RequestPurchaseAsync` ×1、
`ReportConsumableFulfillmentAsync` ×1；`mint_store_id_key` 里那两处已经被我们的轮询替代，无需改）。

回归自检：

```bash
grep -c "GetUserCollectionAsync"                           src/windows.rs  # 1
grep -c "IAP_DIAG get_products"                            src/windows.rs  # 1
grep -c "IAP_DIAG store product"                           src/windows.rs  # 1
grep -c "IAP_DIAG mint key"                                src/windows.rs  # 2
grep -c "IAP_DIAG restore: collection consumables added"   src/windows.rs  # 1
```

## 上游 PR

这两条都是通用缺陷（任何 Windows + Tauri + 消耗品/订阅的 App 都会踩），值得提 PR。
若上游合入，本 fork 可以退役，依赖行改回 crates.io 版本即可。
