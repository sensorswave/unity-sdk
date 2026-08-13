# Sensors Wave Unity SDK

English | [简体中文](README.zh-CN.md)

A Unity data-collection SDK for [Sensors Wave](https://www.sensorswave.cn/). Built for Unity games and cross-platform apps, it provides event tracking, user identification, user properties, common properties, A/B testing & feature flags, UTM attribution, and app lifecycle events. Distributed as a Unity `.unitypackage`, it is independent of any specific game logic.

If you are new to Sensors Wave, visit [sensorswave.cn](https://www.sensorswave.cn/) to learn about the product and create an account.

> Requires **Unity 2020.3 LTS** or later

---

## Core Capabilities

- **Event tracking**: custom events via `TrackEvent`, full-field advanced events via `Track`, with event/property name validation
- **User identity**: anonymous ID (`anon_id`) auto-generated and persisted; login ID (`login_id`) binding and switching
- **User properties**: seven operations — `$set` / `$set_once` / `$increment` / `$append` / `$union` / `$unset` / `$delete`
- **Common properties**: static properties and dynamic properties (re-evaluated per event); clear by key or all at once
- **A/B testing & feature flags**: `CheckFeatureGate` / `GetFeatureConfig` / `GetExperiment`, async callbacks, cache-first (fastFetch) strategy
- **UTM attribution**: automatically parses UTM fields from launch parameters
- **Lifecycle events**: auto-captures `$AppInstall` / `$AppStart` / `$AppEnd`
- **Preset properties**: automatically attaches environment info such as device, OS, network, app, and SDK version
- **Offline reliable reporting**: local persistent queue, batch sending, automatic retry on failure with exponential backoff

## Platform Support

- iOS / Android / HarmonyOS / Windows / macOS / Mini Games / H5 (web)
- Unity **2020.3 LTS** and above
- Supports both runtime (Player) and editor extensions (Editor)
- Networking: `UnityEngine.Networking.UnityWebRequest`
- JSON: `Newtonsoft.Json` (must be declared as a package dependency)

---

## Installation

### 1. Download the `.unitypackage`

Go to [Releases](https://github.com/sensorswave/unity-sdk/releases) and download the `sensorswave-unity-<version>.unitypackage` for your target version.

### 2. Import into Unity

In Unity, open `Assets → Import Package → Custom Package…`, select the downloaded file, and click `Import` in the import window (keep all items selected by default). After importing, the SDK assets are located under `Assets/Sensorswave/`.

### Dependency: Newtonsoft.Json (must be provided by you)

The SDK depends on `Newtonsoft.Json`, but a `.unitypackage` does not automatically pull in UPM package dependencies. If your project does not already include it, install it via the Package Manager:

In Unity, open `Window → Package Manager → + → Add package from git URL` and enter `com.unity.nuget.newtonsoft-json`. Skip this step if your project already has this dependency.

---

## Quick Start

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
            AutoCapture       = true,   // auto-capture lifecycle events
            BatchSend         = false,   // batch sending disabled by default
            EnableAB          = true,   // enable A/B testing
            AbRefreshInterval = 10 * 60 * 1000, // refresh every 10 minutes
            Debug             = false,
        };

        // 1. Initialize (sourceToken is required; ApiHost belongs to config)
        Sensorswave.Init("your-source-token", config);

        // 2. Associate the user id (SetLoginId / Identify)
        Sensorswave.SetLoginId("user-123");

        // 3. Send a custom event
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

## Configuration

Public fields exposed by `SensorswaveConfig` (other implementation details are hidden):

| Field | Type | Default | Description |
|------|------|--------|------|
| `ApiHost` | string | — | Data reporting endpoint; **required**; HTTPS recommended |
| `AutoCapture` | bool | `true` | Whether to auto-capture lifecycle events (`$AppInstall` / `$AppStart` / `$AppEnd`) |
| `Debug` | bool | `false` | Enable debug logging |
| `BatchSend` | bool | `false` | Enable batching + scheduled sending |
| `EnableAB` | bool | `false` | Enable A/B testing & feature flags |
| `AbRefreshInterval` | int (ms) | `600000` | AB config refresh interval (10 minutes) |
| `OptOutCapturing` | bool | `false` | Compliance: disable capturing right at init. When `true`, the SDK does not construct/enqueue/report any event, sends no AB requests, and reads no user identity; call `OptInCapturing()` to enable after the user consents |
| `PersistOptOut` | bool | `false` | Compliance: whether to persist the opt-out state locally so it survives across sessions. When `true`, the state toggled via `OptOutCapturing()` / `OptInCapturing()` is persisted and auto-restored on next launch (an explicit `OptOutCapturing=true` still forces disable and takes the highest priority) |

---

## API Reference

All public APIs are on the `Sensorswave` static entry class (namespace `Sensorswave`). Calls made before initialization are silently ignored.

### Initialization & Lifecycle

#### Init

```csharp
public static void Init(string sourceToken, SensorswaveConfig config);
```

Initializes the SDK. `sourceToken` is assigned by the server; `config.ApiHost` is required; repeated calls are ignored.

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

Stops timers, destroys the plugin, immediately flushes any remaining events, and releases the host `MonoBehaviour`. Internally uses a coroutine to wait for the flush to complete before cleaning up the context. Recommended to call on app exit or hot-reload.

#### IsInitialized

```csharp
public static bool IsInitialized { get; }
```

Whether the SDK has finished initializing.

### Event Tracking

#### TrackEvent

```csharp
public static void TrackEvent(string eventName, Dictionary<string, object> properties = null);
```

Manually reports a custom event. The event name and property names are validated; invalid names are rejected and logged.

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

Advanced API. Construct a full `EventData` directly, allowing you to customize `time`, `trace_id`, `anon_id`, `login_id`, `user_properties`, and every other field. Suitable for scenarios requiring full control over the event structure.

`EventData` fields:

| Field | Type | Description |
|------|------|------|
| `event` | string | Event name (required) |
| `time` | long | Millisecond timestamp (auto-filled at construction) |
| `trace_id` | string | Request trace ID (auto-generated UUID) |
| `anon_id` | string | Anonymous user ID |
| `login_id` | string | Login user ID (skipped in serialization when null) |
| `properties` | `Dictionary<string, object>` | Event properties |
| `user_properties` | `Dictionary<string, object>` | User properties |

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

### User Identity

#### Identify

```csharp
public static void Identify(string loginId);
```

Sets the current user's login ID and sends an `$Identify` event, linking anonymous behavior with the logged-in user.

```csharp
Sensorswave.Identify("user-12345");
```

#### SetLoginId

```csharp
public static void SetLoginId(string loginId);
```

Sets the login ID **without** sending an association event. Use this when you only need to identify the user without tracking an association event.

#### GetAnonId / GetLoginId

```csharp
public static string GetAnonId();  // anonymous ID (auto-generated, persisted across sessions)
public static string GetLoginId(); // current login ID; returns empty string if not set
```

```csharp
Debug.Log($"AnonId={Sensorswave.GetAnonId()}, LoginId={Sensorswave.GetLoginId()}");
```

### User Properties

All Profile methods act on the "current user" and are ultimately reported as user-property events.

| Method | Description |
|------|------|
| `ProfileSet(dict)` | Set properties; overwrites existing values |
| `ProfileSetOnce(dict)` | Set properties only if they do not already exist; does not overwrite |
| `ProfileIncrement(dict)` | Increment numeric properties (numbers only) |
| `ProfileAppend(dict)` | Append values to a list property (without deduplication) |
| `ProfileUnion(dict)` | Append values to a list property (with deduplication) |
| `ProfileUnset(string[] keys)` | Set specified properties to null (equivalent to deletion) |
| `ProfileDelete()` | Delete all properties of the current user; irreversible |

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

### Common Properties

Once registered, common properties are automatically attached to all subsequent events.

#### RegisterCommonProperties (static)

```csharp
public static void RegisterCommonProperties(Dictionary<string, object> properties);
```

Registers static common properties with fixed values.

```csharp
Sensorswave.RegisterCommonProperties(new Dictionary<string, object>
{
    { "server", "asia-east-1" },
    { "vip_level", 3 },
});
```

#### RegisterCommonProperties (dynamic)

```csharp
public static void RegisterCommonProperties(string key, Func<object> dynamicProperty);
```

Registers a dynamic common property that is re-evaluated on every event report (suitable for real-time data such as coin balance or session duration).

```csharp
Sensorswave.RegisterCommonProperties("coin_balance", () => PlayerState.Instance.CoinBalance);
```

#### ClearCommonProperties

```csharp
public static void ClearCommonProperties(string[] keys = null);
```

Clears common properties. Pass `keys` to clear specific entries; pass `null` (or omit) to clear all.

```csharp
Sensorswave.ClearCommonProperties(new[] { "vip_level" }); // clear specific
Sensorswave.ClearCommonProperties();                       // clear all
```

### A/B Testing & Feature Flags

> Requires `EnableAB = true` in the config; otherwise the methods above will simply call back with default values.

Uses an async callback form (to avoid blocking the Unity main thread) with a cache-first (fastFetch) strategy: read the local cache first, and only request from the network if the cache is invalid.

#### CheckFeatureGate

```csharp
public static void CheckFeatureGate(string key, Action<bool> callback);
```

Queries whether a feature flag is enabled for the current user. Calls back `false` when uninitialized or the plugin is absent.

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

Gets a remote config (a JSON object, auto-parsed). Calls back an empty dictionary on miss or when not enabled.

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

Gets the variant config for an A/B experiment. Calls back an empty `Dictionary` on miss (consistent with the JS SDK; does not return `null`).

```csharp
Sensorswave.GetExperiment("homepage_layout", exp =>
{
    if (exp.TryGetValue("variant", out var v))
        ApplyLayout(v);
});
```

### Compliance & Capture Consent (Opt-out)

The SDK provides a capture-consent (opt-out) mechanism to satisfy "capture only after user authorization" compliance requirements. While disabled, it follows a **no-write, no-read** principle: it does not construct/enqueue/report any event, does not send the AB `/ab/evalall` request, and does not read user identity (`anon_id` / `login_id`).

- Events that occur while disabled are **dropped**.
- Residual events already enqueued are not deleted; they are reported normally once capturing is re-enabled.
- The initial state is controlled by the `OptOutCapturing` config (see [Configuration](#configuration)); switch it at runtime via the APIs below.

#### OptOutCapturing

```csharp
public static void OptOutCapturing();
```

Disables capturing. You can already enter the disabled state at init via `OptOutCapturing=true`; call this method to disable again at runtime (e.g. when the user revokes consent). Silently ignored when not initialized.

```csharp
Sensorswave.OptOutCapturing();
Debug.Log($"Capturing disabled: {Sensorswave.HasOptedOutCapturing()}");
```

#### OptInCapturing

```csharp
public static void OptInCapturing();
```

Enables capturing.

```csharp
// Enable capturing after the user accepts the privacy policy
Sensorswave.OptInCapturing();
```

#### HasOptedOutCapturing

```csharp
public static bool HasOptedOutCapturing();
```

Queries whether capturing is currently disabled. Returns `true` when disabled; returns `false` when not initialized or already enabled.

```csharp
if (Sensorswave.HasOptedOutCapturing())
{
    // Not yet authorized; skip capture-dependent logic
}
```

---

## Preset Events

Auto-triggered when `AutoCapture = true`:

| Event | Trigger |
|------|----------|
| `$AppInstall` | First app launch (based on a local flag) |
| `$AppStart` | App enters the foreground |
| `$AppEnd` | App enters the background, with `$event_duration` attached |

For custom events, use `TrackEvent` / `Track`.

## Preset Properties

When `EnablePresetProperties = true` (on by default), `PresetPropertiesPlugin` automatically attaches these to all events:

| Property | Description |
|------|------|
| `$lib` / `$lib_version` | SDK type `unity` and version |
| `$os` / `$os_version` | Operating system and version |
| `$model` / `$brand` / `$manufacturer` | Device model / brand / manufacturer |
| `$device_id` | Unique device identifier |
| `$screen_width` / `$screen_height` | Screen resolution |
| `$timezone_offset` / `$language` | Timezone offset / system language |
| `$network_type` / `$wifi` | Network type / whether on Wi-Fi |
| `$app_version` / `$app_id` / `$app_name` | App version / package id / name |

## Editor Integration

The SDK provides a `SensorswaveSettings` ScriptableObject (`Create → Sensorswave → Sensorswave Settings`) that lets you visually configure token, host, batching, lifecycle, A/B, UTM, etc. in the Inspector, and generate a runtime `SensorswaveConfig` via `ToRuntimeConfig()`. `Editor/SensorswaveSettingsInspector.cs` provides a custom Inspector with validation.

```csharp
var settings = SensorswaveSettings.CreateInstance<SensorswaveSettings>();
// After filling it out in the Inspector:
if (settings.Validate(out var error))
{
    Sensorswave.Init(settings.SourceToken, settings.ToRuntimeConfig());
}
```
---

## License

[Apache-2.0](com.sensorswave.unity/LICENSE)
