# Sensors Wave Unity SDK

[English](README.md) | 简体中文

[Sensors Wave](https://www.sensorswave.cn/) Unity 数据采集 SDK。面向 Unity 游戏与跨平台应用，提供埋点采集、用户识别、用户属性、公共属性、A/B 测试与功能开关、UTM 渠道归因、应用生命周期事件、异常与崩溃采集等能力，以 Unity `.unitypackage` 形式分发，不依赖任何特定游戏逻辑。

如果你是第一次接触 Sensors Wave，欢迎前往 [sensorswave.cn](https://www.sensorswave.cn/) 了解产品并创建账号。

> 最低支持 **Unity 2020.3 LTS**

---

## 核心能力

- **事件采集**：自定义事件 `TrackEvent`、全字段高级事件 `Track`，事件名/属性名校验
- **用户身份**：匿名 ID（`anon_id`）自动生成并持久化，登录 ID（`login_id`）绑定与切换
- **用户属性**：`$set` / `$set_once` / `$increment` / `$append` / `$union` / `$unset` / `$delete` 七种操作
- **公共属性**：静态属性与动态属性（每次事件求值），支持按 key 清除或全部清除
- **A/B 测试与功能开关**：`CheckFeatureGate` / `GetFeatureConfig` / `GetExperiment`，异步回调，缓存优先（fastFetch）策略
- **UTM 渠道归因**：自动解析启动参数中的 UTM 字段
- **生命周期事件**：自动采集 `$AppInstall` / `$AppStart` / `$AppEnd`
- **异常与崩溃采集**：自动捕获未捕获托管异常（error）与进程致命崩溃（fatal，下次启动补偿上报）为 `$Exception` 事件；已捕获的异常可用 `TrackException` 手动上报
- **预置属性**：自动附加设备、系统、网络、应用、SDK 版本等环境信息
- **离线可靠上报**：本地持久化队列、批量发送、失败自动重试、指数退避

## 平台支持

- iOS / Android / HarmonyOS / Windows / macOS / 小游戏 / H5（web）
- Unity **2020.3 LTS** 及以上
- 同时支持运行时（Player）与编辑器扩展（Editor）
- 网络：`UnityEngine.Networking.UnityWebRequest`
- JSON：`Newtonsoft.Json`（需声明为包依赖）

---

## 安装

### 1. 下载 `.unitypackage`

前往 [Releases](https://github.com/sensorswave/unity-sdk/releases) 下载目标版本的 `sensorswave-unity-<version>.unitypackage`。

### 2. 导入 Unity

Unity 菜单 `Assets → Import Package → Custom Package…`，选中下载的文件，在导入窗口点 `Import`（默认全选即可）。导入后 SDK 资源位于 `Assets/Sensorswave/`。

### 依赖：Newtonsoft.Json（需自备）

SDK 依赖 `Newtonsoft.Json`，但 `.unitypackage` 不会自动拉取 UPM 包依赖。若你的工程尚未引入，通过 Package Manager 安装：

Unity 菜单 `Window → Package Manager → + → Add package from git URL`，填入 `com.unity.nuget.newtonsoft-json`。已有该依赖的工程跳过此步。

---

## 快速开始

```csharp
using System.Collections.Generic;
using Sensorswave;
using UnityEngine;

public class GameInitializer : MonoBehaviour
{
    void Awake()
    {
        var config = new SensorswaveConfig
        {
            ApiHost           = "https://api.your-domain.com",
            AutoCapture       = true,   // 自动采集生命周期事件
            BatchSend         = false,   // 默认不批量上报
            EnableAB          = true,   // 启用 A/B 测试
            AbRefreshInterval = 10 * 60 * 1000, // 10 分钟刷新一次
            Debug             = false,
        };

        // 1. 初始化（sourceToken 必填，ApiHost 属于 config）
        Sensorswave.Init("your-source-token", config);

        // 2. 设置用户关联 id（SetLoginId / Identify）
        Sensorswave.SetLoginId("user-123");

        // 3. 发送自定义事件
        Sensorswave.TrackEvent("LevelComplete", new Dictionary<string, object>
        {
            { "level", 5 },
            { "score", 100 },
            { "duration_ms", 12345 },
        });
    }
}
```

---

## 配置项

`SensorswaveConfig` 对外暴露的字段（其它实现细节已隐藏）：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `ApiHost` | string | — | 数据上报地址，**必填**，建议使用 HTTPS |
| `AutoCapture` | bool | `true` | 是否自动采集生命周期事件（`$AppInstall` / `$AppStart` / `$AppEnd`） |
| `Debug` | bool | `false` | 是否开启 Debug 日志 |
| `BatchSend` | bool | `false` | 是否启用攒批 + 定时上报 |
| `EnableAB` | bool | `false` | 是否启用 A/B 测试与功能开关 |
| `AbRefreshInterval` | int (ms) | `600000` | AB 配置刷新间隔（10 分钟） |
| `EnableErrorTrack` | bool | `false` | 异常采集：自动捕获未捕获的托管异常，上报 `$Exception`（`$exception_level = "error"`）。全平台生效，独立于 `AutoCapture` |
| `EnableCrashTrack` | bool | `false` | 崩溃采集：检测未捕获异常导致的进程终止，上报 `$Exception`（`$exception_level = "fatal"`）。仅 App 环境生效（iOS / Android / HarmonyOS / PC / Editor）；WebGL / 小游戏无进程概念 |
| `OptOutCapturing` | bool | `false` | 合规：初始化即禁用采集。设为 `true` 后 SDK 不构造/入队/上报任何事件、不发 AB 请求、不读用户标识；用户授权后调用 `OptInCapturing()` 开启 |
| `PersistOptOut` | bool | `false` | 合规：是否将 opt-out 状态持久化到本地、跨会话保留。设为 `true` 时经 `OptOutCapturing()` / `OptInCapturing()` 切换的状态会被持久化，下次启动自动恢复（`OptOutCapturing=true` 仍强制禁用，优先级最高） |

---

## API 参考

所有公共 API 位于 `Sensorswave` 静态入口类（命名空间 `Sensorswave`）。未初始化时调用会被静默忽略。

### 初始化与生命周期

#### Init

```csharp
public static void Init(string sourceToken, SensorswaveConfig config);
```

初始化 SDK。`sourceToken` 由服务端分配；`config.ApiHost` 必填；重复调用会被忽略。

```csharp
var config = new SensorswaveConfig
{
    ApiHost = "https://api.your-domain.com",
    EnableAB = true,
};
Sensorswave.Init("your-source-token", config);
```

#### Destroy

```csharp
public static void Destroy();
```

停止定时器、销毁插件、立即 flush 残留事件并释放宿主 `MonoBehaviour`。内部用协程等待 flush 完成后再清理上下文。建议在应用退出或热重载时调用。

#### IsInitialized

```csharp
public static bool IsInitialized { get; }
```

SDK 是否已完成初始化。

### 事件追踪

#### TrackEvent

```csharp
public static void TrackEvent(string eventName, Dictionary<string, object> properties = null);
```

手动上报一个自定义事件。事件名与属性名会经过校验，非法名称会被拦截并记录日志。

```csharp
Sensorswave.TrackEvent("purchase", new Dictionary<string, object>
{
    { "order_id", "ORD-001" },
    { "amount", 99.0 },
    { "currency", "CNY" },
});
```

#### Track

```csharp
public static void Track(EventData eventData);
```

高级 API，直接构造完整 `EventData`，可自定义 `time`、`trace_id`、`anon_id`、`login_id`、`user_properties` 等全部字段。适合需要完全控制事件结构的场景。

`EventData` 字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `event` | string | 事件名（必填） |
| `time` | long | 毫秒时间戳（构造时自动填充） |
| `trace_id` | string | 请求追踪 ID（自动生成 UUID） |
| `anon_id` | string | 匿名用户 ID |
| `login_id` | string | 登录用户 ID（为 null 时不参与序列化） |
| `properties` | `Dictionary<string, object>` | 事件属性 |
| `user_properties` | `Dictionary<string, object>` | 用户属性 |

```csharp
var evt = new EventData("purchase")
{
    properties = new Dictionary<string, object>
    {
        { "order_id", "ORD-001" },
        { "amount", 99.0 },
    },
    user_properties = new Dictionary<string, object>
    {
        { "plan", "premium" },
    },
};
Sensorswave.Track(evt);
```

### 异常采集（$Exception）

SDK 可自动捕获托管异常与崩溃并上报为 `$Exception` 事件，同时提供手动 API 上报已捕获的异常。

`SensorswaveConfig` 中两个独立开关（均默认 `false`，独立于 `AutoCapture`）：

| 开关 | 级别 | 生效范围 | 采集内容 |
|------|------|------|------|
| `EnableErrorTrack` | `$exception_level = "error"` | 全部宿主（App / PC / Editor / WebGL / 小游戏） | 未捕获托管异常（Unity `LogType.Exception`：被引擎兜底记录但进程未终止） |
| `EnableCrashTrack` | `$exception_level = "fatal"` | 仅 App 宿主（iOS / Android / HarmonyOS / PC / Editor） | 未捕获异常导致进程终止（app crash） |

每条 `$Exception` 事件携带 `$exception_level`、`$exception_type`、`$exception_message`、`$exception_frames`（结构化堆栈帧，最多 30 帧；无法解析出帧时省略）。

fatal 检测机制：捕获到异常时 SDK 写入崩溃标记文件（`sw_crash_marker.json`）；进程活过宽限窗口（10 秒）则删除标记（仅按 error 上报）；进程在窗口内死亡，则下次启动补报 `fatal` 级 `$Exception`，事件 `time` 与用户归属还原为崩溃时刻。正常退出与进入后台都会删除标记，后台被系统回收不会被误报为 crash。

防异常风暴机制内建（无需配置）：同指纹 1 秒防抖、单会话同指纹最多上报 10 条、单会话 128 个不同指纹熔断。

已知边界：纯 C# 层捕获，不覆盖原生崩溃（SIGSEGV、OC/Java 未捕获异常）；两开关都开且异常致命时，会各报一条 `error` 与 `fatal`（两个独立事实）。

#### TrackException

```csharp
public static void TrackException(Exception exception, Dictionary<string, object> properties = null);
```

手动上报一个已捕获的异常（`$Exception` 事件，`$exception_level` 固定为 `"error"`）。适用于游戏自身 catch 了异常但仍希望上报的场景。不受 `EnableErrorTrack` / `EnableCrashTrack` 开关限制；仍受初始化与 opt-out 合规守卫约束（禁用期调用为 no-op）。`exception` 为 null 时记录日志并忽略。`properties` 中同名 `$exception_*` 键以 SDK 生成的值为准。

```csharp
try
{
    EnterLevel();
}
catch (Exception e)
{
    Sensorswave.TrackException(e, new Dictionary<string, object>
    {
        { "level_id", 42 },
    });
}
```

### 用户身份

#### Identify

```csharp
public static void Identify(string loginId);
```

设置当前用户的登录 ID，并发送 `$Identify` 事件，将匿名行为与登录用户关联。

```csharp
Sensorswave.Identify("user-12345");
```

#### SetLoginId

```csharp
public static void SetLoginId(string loginId);
```

设置登录 ID，**不**发送关联事件。如只需识别用户、不追踪关联事件时使用。

#### GetAnonId / GetLoginId

```csharp
public static string GetAnonId();  // 匿名 ID（自动生成，跨会话持久化）
public static string GetLoginId(); // 当前登录 ID，未设置返回空字符串
```

```csharp
Debug.Log($"AnonId={Sensorswave.GetAnonId()}, LoginId={Sensorswave.GetLoginId()}");
```

#### Reset

```csharp
public static void Reset(bool resetAnonId = false);
```

用户登出时调用，解除登录 ID 与设备的绑定。默认保留匿名 ID，登出后事件继续以匿名 ID 作为 distinct_id 上报；传 `true` 时同时重置（重新生成）匿名 ID，如公共设备场景。已入队/已上报的历史事件不受影响。

```csharp
// 用户登出时
Sensorswave.Reset();  // 默认保留匿名 ID

// 如果需要同时重置匿名 ID（如公共设备场景）
Sensorswave.Reset(true);
```

### 用户属性

所有 Profile 方法作用于「当前用户」，最终以用户属性事件上报。

| 方法 | 说明 |
|------|------|
| `ProfileSet(dict)` | 设置属性，已存在则覆盖 |
| `ProfileSetOnce(dict)` | 仅在属性不存在时设置，不覆盖 |
| `ProfileIncrement(dict)` | 数值属性自增（仅支持数值） |
| `ProfileAppend(dict)` | 向列表属性追加值（不去重） |
| `ProfileUnion(dict)` | 向列表属性追加值（去重） |
| `ProfileUnset(string[] keys)` | 将指定属性置为 null（等价于删除） |
| `ProfileDelete()` | 删除当前用户的全部属性，不可恢复 |

```csharp
Sensorswave.ProfileSet(new Dictionary<string, object>
{
    { "nickname", "Alice" },
    { "level", 5 },
});

Sensorswave.ProfileSetOnce(new Dictionary<string, object>
{
    { "register_date", "2026-01-01" },
});

Sensorswave.ProfileIncrement(new Dictionary<string, object>
{
    { "login_count", 1 },
    { "points", 100 },
});

Sensorswave.ProfileAppend(new Dictionary<string, object>
{
    { "tags", new List<string> { "vip", "beta" } },
});

Sensorswave.ProfileUnset(new[] { "temp_campaign" });

Sensorswave.ProfileDelete();
```

### 公共属性

注册后，公共属性会自动附加到所有后续事件。

#### RegisterCommonProperties（静态）

```csharp
public static void RegisterCommonProperties(Dictionary<string, object> properties);
```

注册静态公共属性，值固定不变。

```csharp
Sensorswave.RegisterCommonProperties(new Dictionary<string, object>
{
    { "server", "asia-east-1" },
    { "vip_level", 3 },
});
```

#### RegisterCommonProperties（动态）

```csharp
public static void RegisterCommonProperties(string key, Func<object> dynamicProperty);
```

注册动态公共属性，每次事件上报时重新求值（适合金币余额、在线时长等实时数据）。

```csharp
Sensorswave.RegisterCommonProperties("coin_balance", () => PlayerState.Instance.CoinBalance);
```

#### ClearCommonProperties

```csharp
public static void ClearCommonProperties(string[] keys = null);
```

清除公共属性。传入 `keys` 清除指定项；传 `null`（或不传）清除全部。

```csharp
Sensorswave.ClearCommonProperties(new[] { "vip_level" }); // 清除指定
Sensorswave.ClearCommonProperties();                       // 清除全部
```

### A/B 测试与功能开关

> 需在配置中启用 `EnableAB = true`，否则上述方法会直接回调默认值。

采用异步回调形式（避免在 Unity 主线程上阻塞），策略为缓存优先（fastFetch）：先读本地缓存，缓存无效再请求网络。

#### CheckFeatureGate

```csharp
public static void CheckFeatureGate(string key, Action<bool> callback);
```

查询功能开关是否对当前用户开启。未初始化或插件不存在时回调 `false`。

```csharp
Sensorswave.CheckFeatureGate("new_checkout", enabled =>
{
    if (enabled) ShowNewCheckout();
    else         ShowOldCheckout();
});
```

#### GetFeatureConfig

```csharp
public static void GetFeatureConfig(string key, Action<Dictionary<string, object>> callback);
```

获取远程配置（JSON 对象，自动解析）。未命中或未启用时回调空字典。

```csharp
Sensorswave.GetFeatureConfig("ui_config", config =>
{
    if (config.Count > 0)
    {
        Debug.Log($"theme={config["theme"]}");
    }
});
```

#### GetExperiment

```csharp
public static void GetExperiment(string key, Action<Dictionary<string, object>> callback);
```

获取 AB 实验的变体配置。未命中时回调空 `Dictionary`（与 JS SDK 一致，不返回 `null`）。

```csharp
Sensorswave.GetExperiment("homepage_layout", exp =>
{
    if (exp.TryGetValue("variant", out var v))
        ApplyLayout(v);
});
```

### 合规与采集授权（Opt-out）

SDK 提供采集授权（opt-out）机制，用于满足「用户授权后再采集」的合规要求。禁用期间遵循**不写、不读**原则：不构造/不入队/上报任何事件、不发送 AB `/ab/evalall` 请求、不读取用户标识（`anon_id` / `login_id`）。

- 禁用期间发生的事件会被**丢弃**。
- 已入队的残留事件不会被删除，留待下次开启后正常上报。
- 初始状态由配置 `OptOutCapturing` 决定（详见[配置项](#配置项)）；运行时通过下方 API 切换。

#### OptOutCapturing

```csharp
public static void OptOutCapturing();
```

禁用采集。初始化时已可通过 `OptOutCapturing=true` 进入禁用态；如需运行时再次禁用（例如用户撤回授权）调用本方法。未初始化时调用会被静默忽略。

```csharp
Sensorswave.OptOutCapturing();
Debug.Log($"已禁用采集: {Sensorswave.HasOptedOutCapturing()}");
```

#### OptInCapturing

```csharp
public static void OptInCapturing();
```

开启采集。

```csharp
// 用户同意隐私协议后开启采集
Sensorswave.OptInCapturing();
```

#### HasOptedOutCapturing

```csharp
public static bool HasOptedOutCapturing();
```

查询当前是否已禁用采集。已禁用返回 `true`；未初始化或已开启返回 `false`。

```csharp
if (Sensorswave.HasOptedOutCapturing())
{
    // 尚未授权，跳过依赖采集的逻辑
}
```

---

## 预置事件

生命周期事件在启用 `AutoCapture = true` 后自动触发；`$Exception` 由独立开关控制（见[异常采集](#异常采集exception)）：

| 事件 | 触发时机 |
|------|----------|
| `$AppInstall` | 首次启动应用（基于本地标记） |
| `$AppStart` | 应用进入前台 |
| `$AppEnd` | 应用进入后台，并附带 `$event_duration` |
| `$Exception` | 未捕获托管异常（`error`，`EnableErrorTrack`）或进程致命崩溃（`fatal`，`EnableCrashTrack`） |

自定义事件请使用 `TrackEvent` / `Track`。

## 预置属性

启用 `EnablePresetProperties = true`（默认开启）后由 `PresetPropertiesPlugin` 自动附加到所有事件：

| 属性 | 说明 |
|------|------|
| `$lib` / `$lib_version` | SDK 类型 `unity` 与版本号 |
| `$os` / `$os_version` | 操作系统及版本 |
| `$model` / `$brand` / `$manufacturer` | 设备型号 / 品牌 / 厂商 |
| `$device_id` | 设备唯一标识 |
| `$screen_width` / `$screen_height` | 屏幕分辨率 |
| `$timezone_offset` / `$language` | 时区偏移 / 系统语言 |
| `$network_type` / `$wifi` | 网络类型 / 是否 WiFi |
| `$app_version` / `$app_id` / `$app_name` | 应用版本 / 包名 / 名称 |

## 编辑器集成

SDK 提供一个 `SensorswaveSettings` ScriptableObject（`Create → Sensorswave → Sensorswave Settings`），可在 Inspector 中以可视化方式配置 token、host、批量参数、生命周期、A/B、UTM、异常/崩溃采集等，并通过 `ToRuntimeConfig()` 生成运行时 `SensorswaveConfig`。`Editor/SensorswaveSettingsInspector.cs` 提供自定义 Inspector 与校验。

```csharp
var settings = SensorswaveSettings.CreateInstance<SensorswaveSettings>();
// 在 Inspector 中填好后：
if (settings.Validate(out var error))
{
    Sensorswave.Init(settings.SourceToken, settings.ToRuntimeConfig());
}
```
---

## 许可证

[Apache-2.0](com.sensorswave.unity/LICENSE)
