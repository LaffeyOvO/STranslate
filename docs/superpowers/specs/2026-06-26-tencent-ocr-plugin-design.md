# 设计：腾讯 OCR 内置插件

**日期**: 2026-06-26
**分支**: main
**状态**: 已批准，待实现

## 背景

STranslate 当前为 v2.0 插件化架构。1.0 分支中 `TencentOCR .cs`（`src/STranslate/ViewModels/Preference/OCR/`）是内置 OCR 服务（继承 `OCRBase` 实现 `IOCR`，用 `Newtonsoft.Json` + `HttpUtil.PostAsync`，配置属性 `AppID`/`AppKey`/`TencentOcrAction` 直接挂在类上），但 v2.0 已重构为独立插件项目（实现 `IOcrPlugin` 接口，用 `System.Text.Json` + `IPluginContext.HttpService`，配置存到独立 `Settings` 类）。

需将 1.0 的腾讯 OCR 逻辑适配为 v2.0 内置 OCR 插件，调用腾讯云文字识别 OCR API（`https://ocr.tencentcloudapi.com`），采用 TC3-HMAC-SHA256 签名鉴权，支持**通用印刷体识别**（`GeneralBasicOCR`，多语种）与**通用印刷体识别（高精度版）**（`GeneralAccurateOCR`，中英文），返回**行级文本 + 坐标框**（支持图片翻译）。

## 目标

- 新增内置 OCR 插件 `STranslate.Plugin.Ocr.Tencent`，纳入解决方案。
- 复用 1.0 的 TC3 签名算法、请求构造与响应解析逻辑，适配 v2.0 插件规范。
- 提供独立的 `Settings` 持久化与 WPF 设置 UI、5 套国际化资源。

## 关键事实

1. **v2.0 OCR 插件接口**（`src/STranslate.Plugin/IOcrPlugin.cs`）：`IOcrPlugin : IPlugin`，需实现 `Init(IPluginContext)`、`Control GetSettingUI()`、`Dispose()`、`IEnumerable<LangEnum> SupportedLanguages`、`bool SupportBoxPoints() => false`（可 override）、`Task<OcrResult> RecognizeAsync(OcrRequest, CancellationToken)`。`OcrRequest` = `(byte[] ImageData, LangEnum Language, int PixelWidth, int PixelHeight)`；`OcrResult` 含 `OcrContents`（每项 `OcrContent{Text, BoxPoints}`）、`Regions`、实例方法 `Fail(msg)` 等。`OcrContent` 无字符串构造函数，须用对象初始化器 `{ Text = ... }`。
2. **内置插件规范**（参考 `STranslate.Plugin.Ocr.Baidu`）：扁平目录结构，`.csproj` 用 `ProjectReference` 引用 `STranslate.Plugin.csproj`，输出到 `.artifacts\Debug\Plugins\<PluginName>\`，含 `plugin.json`、`icon.png`、`Languages/*.{xaml,json}`、`Main.cs`、`Settings.cs`、`Extensions.cs`、`ViewModel/SettingsViewModel.cs`、`View/SettingsView.xaml(.cs)`，并在 `src/STranslate.slnx` 的 `/Plugins/` 文件夹登记。
3. **HttpService.PostAsync**（`IHttpService.cs:138`）：`Task<string> PostAsync(string url, object content, Options? options = null, CancellationToken cancellationToken = default)`，`content` 为对象时自动序列化为 JSON（默认 `ContentType = "application/json"`）。但腾讯鉴权需对**签名时的 body 字符串**做 SHA256，故 `content` 传入**已拼好的 JSON 字符串**（PostAsync 对 string 内容原样发送），确保签名 body 与实际 body 一致。
4. **Options**（`IHttpService.cs:8`）：可设 `Headers`（`Dictionary<string,string>`）、`QueryParams`、`Timeout`、`ContentType`。腾讯请求需在 `Headers` 携带 `Host`/`X-TC-Timestamp`/`X-TC-Version`/`X-TC-Action`/`X-TC-Region`/`X-TC-Token`/`X-TC-RequestClient`/`Authorization`，`ContentType` 设为 `application/json; charset=utf-8`。
5. **1.0 鉴权逻辑**（`TencentOCR .cs`）：TC3-HMAC-SHA256 签名。`canonicalRequest = POST\n/\n\n{canonicalHeaders}\n{signedHeaders}\n{sha256(body)}`；`stringToSign = TC3-HMAC-SHA256\n{timestamp}\n{credentialScope}\n{sha256(canonicalRequest)}`；派生密钥 `secretDate=HMAC(TC3+secretKey, date)` → `secretService=HMAC(secretDate, service)` → `secretSigning=HMAC(secretService, "tc3_request")` → `signature=HMAC(secretSigning, stringToSign)`；`Authorization = TC3-HMAC-SHA256 Credential={secretId}/{credentialScope}, SignedHeaders={signedHeaders}, Signature={signature}`。其中 `date` 由 timestamp 转 UTC `yyyy-MM-dd`，`service` = host 第一段（`ocr`），`credentialScope = {date}/{service}/tc3_request`。
6. **1.0 请求构造**：URL `https://ocr.tencentcloudapi.com`；`version=2018-11-19`；`action` = 枚举 `.ToString()`（`GeneralBasicOCR`/`GeneralAccurateOCR`，恰好与腾讯 API Action 名一致）；`region` = `ap-shanghai`（1.0 默认）；Body 为 JSON：`GeneralBasicOCR` 带 `ImageBase64`+`LanguageType`，`GeneralAccurateOCR` 仅 `ImageBase64`（该接口不支持 `LanguageType`，仅中英文高精度）。
7. **1.0 响应结构**：`Root{ Response{ TextDetections[], Error?, Angle, Language, RequestId, PdfPageSize } }`；`TextDetectionsItem{ DetectedText, Confidence, Polygon[{X,Y}], ItemPolygon{X,Y,Width,Height}, AdvancedInfo, WordCoordPoint, Words }`；`Error{ Code, Message }`。
8. **1.0 `LangConverter` 映射**（仅 `GeneralBasicOCR` 用）：`auto→auto`、`zh_cn→zh`、`zh_tw/yue→zh_rare`、`en→auto`、`ja→jap`、`ko→kor`、`fr→fre`、`es→spa`、`ru→rus`、`de→ger`、`it→ita`、`pt_pt/pt_br→por`、`vi→vie`、`th→tha`、`ms→may`、`ar→ara`、`hi→hi`、`sv→swe`、`nb_no/nn_no→nor`、`nl→hol`；其余（`tr`/`id`/`mn_*`/`km`/`fa`/`pl`/`uk`/`uz`）返回 `null`（不支持）。
9. **1.0 枚举**（`src/STranslate.Model/Enums.cs`）：`TencentOCRAction{ [Description("通用印刷体识别")] GeneralBasicOCR, [Description("通用印刷体识别(高精度版)")] GeneralAccurateOCR }`；`TencentRegionEnum{ ap_shanghai, ap_beijing, ... }`（1.0 默认 `ap_shanghai`，`.ToString().Replace("_","-")` → `ap-shanghai`）。本设计区域固定，不引入 `TencentRegionEnum`。
10. **plugin.json 格式**：`PluginID` 为 32 位无横线 GUID（如 Baidu `64f81251...`、Google `2e83ee2f...`），含 `Name`/`Description`/`Author`/`Version`/`Website`/`ExecuteFileName`/`IconPath`。
11. **PrePluginIDs**（`src/STranslate/Core/Constant.cs:56`）：内置插件须在此列表登记 ID，`PluginManager` 据此决定预装路径。新增腾讯插件须追加其 ID。

## 设计决策

- **位置**：内置插件（`src/Plugins/STranslate.Plugin.Ocr.Tencent/`），与 Baidu/OpenAI/Google 一致。
- **鉴权**：直接移植 1.0 的 `GetAuth`/`Sha256Hex`/`HmacSha256`，用 `System.Security.Cryptography`（SHA256/HMACSHA256，框架内置，无额外依赖）。
- **Body 传字符串**：因签名需对 body 字符串求 SHA256，`PostAsync` 的 `content` 传**拼好的 JSON 字符串**（不传匿名对象），保证签名 body 与实际请求 body 完全一致。
- **区域固定**：固定 `ap-shanghai`（1.0 默认），不暴露区域下拉（YAGNI）。
- **URL 固定**：固定 `https://ocr.tencentcloudapi.com`，不暴露 URL 配置（与 1.0 默认一致；腾讯 OCR 仅此一个端点）。
- **配置项**：暴露 `SecretId`、`SecretKey`、`Action`（下拉：通用/高精度）。命名用 v2.0 风格 `SecretId`/`SecretKey`，而非 1.0 的 `AppID`/`AppKey`。
- **支持坐标框**：`SupportBoxPoints()` 返回 `true`，`BoxPoints` 取自 `Polygon`（4 个坐标点）。
- **`SupportedLanguages` 按动作区分**（仿 Baidu 插件按 `Action` 分支）：
  - `GeneralBasicOCR`：返回 `LangConverter` 支持的语种（Auto/简中/繁中/粤语/英/日/韩/法/西/俄/德/意/葡×2/越/泰/马来/阿拉伯/印地/瑞典/挪威×2/荷兰）。
  - `GeneralAccurateOCR`：仅 Auto/简中/繁中/粤语/英（高精度版仅中英文）。
- **JSON 解析**：用 `System.Text.Json`（`JsonSerializer.Deserialize<Root>`），定义精简 DTO（`Root`/`Response`/`TextDetectionsItem`/`PolygonItem`/`Error`），与 Baidu 插件风格一致（Baidu 亦定义 DTO）。
- **不做段落聚合**：腾讯响应只给行级 `TextDetections`，无 paragraphs 概念，不移植 Baidu 的 `AddStructuredLayout`。
- **PluginID**：新生成 GUID `bb65c593ebb04d40bc2c5ad55aecc4e2`。
- **icon.png**：复制 Baidu 的 icon.png 占位，后续可替换。

## 目录结构

```
src/Plugins/STranslate.Plugin.Ocr.Tencent/
├── STranslate.Plugin.Ocr.Tencent.csproj
├── Main.cs
├── Settings.cs
├── Extensions.cs
├── plugin.json
├── icon.png
├── View/
│   ├── SettingsView.xaml
│   └── SettingsView.xaml.cs
├── ViewModel/
│   └── SettingsViewModel.cs
└── Languages/
    ├── en.xaml / en.json
    ├── zh-cn.xaml / zh-cn.json
    ├── zh-tw.xaml / zh-tw.json
    ├── ja.xaml / ja.json
    └── ko.xaml / ko.json
```

并在 `src/STranslate.slnx` 的 `/Plugins/` 文件夹 OCR 区段追加：
```xml
<Project Path="Plugins/STranslate.Plugin.Ocr.Tencent/STranslate.Plugin.Ocr.Tencent.csproj" />
```

## 组件设计

### Main.cs（`IOcrPlugin` 实现）

```csharp
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using CommunityToolkit.Mvvm.ComponentModel;
using STranslate.Plugin.Ocr.Tencent.View;
using STranslate.Plugin.Ocr.Tencent.ViewModel;
using System.Windows.Controls;

namespace STranslate.Plugin.Ocr.Tencent;

public class Main : ObservableObject, IOcrPlugin
{
    private const string Url = "https://ocr.tencentcloudapi.com";
    private const string Version = "2018-11-19";
    private const string Region = "ap-shanghai";

    private Control? _settingUi;
    private SettingsViewModel? _viewModel;
    private Settings Settings { get; set; } = null!;
    private IPluginContext Context { get; set; } = null!;

    public IEnumerable<LangEnum> SupportedLanguages =>
        Settings.Action switch
        {
            TencentOCRAction.GeneralBasicOCR => [/* LangConverter 支持的语种 */],
            TencentOCRAction.GeneralAccurateOCR =>
            [
                LangEnum.Auto, LangEnum.ChineseSimplified, LangEnum.ChineseTraditional,
                LangEnum.Cantonese, LangEnum.English
            ],
            _ => Enum.GetValues<LangEnum>()
        };

    public bool SupportBoxPoints() => true;

    public void Init(IPluginContext context) { Context = context; Settings = context.LoadSettingStorage<Settings>(); }
    public Control GetSettingUI() { _viewModel ??= new(Context, Settings); _settingUi ??= new SettingsView { DataContext = _viewModel }; return _settingUi; }
    public void Dispose() => _viewModel?.Dispose();

    public async Task<OcrResult> RecognizeAsync(OcrRequest request, CancellationToken cancellationToken)
    {
        var ocrResult = new OcrResult();
        var base64Str = Convert.ToBase64String(request.ImageData);

        // 1. 构造 body
        string body;
        if (Settings.Action == TencentOCRAction.GeneralBasicOCR)
        {
            var target = LangConverter(request.Language) ?? throw new Exception($"unsupportted language[{request.Language}]");
            body = "{\"ImageBase64\":\"" + base64Str + "\",\"LanguageType\":\"" + target + "\"}";
        }
        else
        {
            body = "{\"ImageBase64\":\"" + base64Str + "\"}";
        }

        // 2. 构造签名与请求头
        var host = Url.Replace("https://", "");
        var contentType = "application/json; charset=utf-8";
        var timestamp = ((int)DateTime.UtcNow.Subtract(new DateTime(1970, 1, 1)).TotalSeconds).ToString();
        var action = Settings.Action.ToString();
        var auth = GetAuth(Settings.SecretId, Settings.SecretKey, host, contentType, timestamp, body);

        var options = new Options
        {
            ContentType = contentType,
            Headers = new Dictionary<string, string>
            {
                { "Host", host },
                { "X-TC-Timestamp", timestamp },
                { "X-TC-Version", Version },
                { "X-TC-Action", action },
                { "X-TC-Region", Region },
                { "X-TC-Token", "" },
                { "X-TC-RequestClient", "SDK_NET_BAREBONE" },
                { "Authorization", auth }
            }
        };

        // 3. POST（传字符串 body，保证签名一致）
        var resp = await Context.HttpService.PostAsync(Url, body, options, cancellationToken);
        if (string.IsNullOrEmpty(resp)) throw new Exception("请求结果为空");

        // 4. 解析
        var parsedData = JsonSerializer.Deserialize<Root>(resp) ?? throw new Exception($"反序列化失败: {resp}");
        if (parsedData.Response.Error != null) return ocrResult.Fail(parsedData.Response.Error.Message);

        foreach (var item in parsedData.Response.TextDetections)
        {
            var content = new OcrContent { Text = item.DetectedText };
            foreach (var pg in item.Polygon) content.BoxPoints.Add(new BoxPoint(pg.X, pg.Y));
            ocrResult.OcrContents.Add(content);
        }
        return ocrResult;
    }

    // LangConverter：照搬 1.0 映射表
    public string? LangConverter(LangEnum lang) { /* auto→auto, zh_cn→zh, ... */ }

    #region TC3 签名
    private static string GetAuth(string secretId, string secretKey, string host, string contentType, string timestamp, string body) { /* 移植 1.0 */ }
    private static string Sha256Hex(string s) { /* SHA256 → 小写 hex */ }
    private static byte[] HmacSha256(byte[] key, byte[] msg) { /* HMACSHA256 */ }
    #endregion

    #region 响应 DTO
    public class Root { public Response Response { get; set; } = new(); }
    public class Response { public List<TextDetectionsItem> TextDetections { get; set; } = []; public Error? Error { get; set; } }
    public class TextDetectionsItem { public string DetectedText { get; set; } = ""; public List<PolygonItem> Polygon { get; set; } = []; }
    public class PolygonItem { public int X { get; set; } public int Y { get; set; } }
    public class Error { public string Code { get; set; } = ""; public string Message { get; set; } = ""; }
    #endregion
}
```

> `GetAuth`/`Sha256Hex`/`HmacSha256` 完整移植 1.0（见 `关键事实` 5），仅命名空间/访问修饰符适配。`LangConverter` 完整移植 1.0 映射表（见 `关键事实` 8），用 v2.0 `LangEnum` 成员名（`ChineseSimplified` 等）替换 1.0 的 `zh_cn` 等。

### Settings.cs

```csharp
using System.ComponentModel;

namespace STranslate.Plugin.Ocr.Tencent;

public class Settings
{
    public TencentOCRAction Action { get; set; } = TencentOCRAction.GeneralBasicOCR;
    public string SecretId { get; set; } = string.Empty;
    public string SecretKey { get; set; } = string.Empty;
}

public enum TencentOCRAction
{
    [Description("通用印刷体识别")] GeneralBasicOCR,
    [Description("通用印刷体识别(高精度版)")] GeneralAccurateOCR
}
```

### Extensions.cs

与 Baidu 插件完全相同的 `EnumExtensions.GetDescription()`。

### ViewModel/SettingsViewModel.cs

仿 Baidu：`ObservableObject, IDisposable`，持有 `Action`/`SecretId`/`SecretKey` 属性，`Actions` 列表（`Enum.GetValues<TencentOCRAction>()`），`PropertyChanged` 回写 `Settings` 并 `SaveSettingStorage<Settings>()`。

### View/SettingsView.xaml

三张 `ui:SettingsCard`：
1. SecretId —— `PasswordBox` + `plugin:PasswordBoxAssistant`
2. SecretKey —— `PasswordBox` + `plugin:PasswordBoxAssistant`
3. 识别动作（Action）—— `ComboBox`（绑定 `Actions`/`Action`），带描述说明「GeneralBasicOCR 为多语种通用版，GeneralAccurateOCR 为中英文高精度版」
4. 官网 —— `HyperlinkButton` → `https://cloud.tencent.com/product/ocr`

### Languages（5 种语言）

资源键前缀 `STranslate_Plugin_Ocr_Tencent_`：`SecretId`、`SecretKey`、`Action`、`Action_Description`、`Official`、`Official_Description`。`.json` 含 `Name`/`Description`。

### plugin.json

```json
{
  "PluginID": "bb65c593ebb04d40bc2c5ad55aecc4e2",
  "Name": "Tencent OCR",
  "Description": "Tencent Cloud OCR plugin for STranslate",
  "Author": "zggsong",
  "Version": "1.0.0",
  "Website": "https://github.com/STranslate/STranslate",
  "ExecuteFileName": "STranslate.Plugin.Ocr.Tencent.dll",
  "IconPath": "icon.png"
}
```

### .csproj

完全仿 Baidu 的 csproj：`TargetFramework=net10.0-windows`、`UseWPF=true`、`ProjectReference` 引用 `STranslate.Plugin.csproj`、输出路径 `.artifacts\Debug\Plugins\STranslate.Plugin.Ocr.Tencent\`、`Content` 包含 `Languages/*.*`、`icon.png`、`plugin.json`。

### Constant.cs 维护（`src/STranslate/Core/Constant.cs:56`）

作为内置插件，在 `PrePluginIDs` 列表 OCR 区段追加（与 BaiduOCR/OpenAIOCR/WeChatOCRBuiltIn/GoogleOCR 并列）：

```csharp
"bb65c593ebb04d40bc2c5ad55aecc4e2", //TencentOCR
```

> 该 ID 必须与 `plugin.json` 的 `PluginID` 一致，否则内置插件无法被识别为预装插件。

## 错误处理

- 响应为空 → `throw new Exception("请求结果为空")`
- `JsonSerializer.Deserialize<Root>` 失败或返回 null → 抛出含原始响应的异常
- API 返回 `Response.Error != null` → `ocrResult.Fail(Error.Message)`（不抛异常，保持 `IsSuccess=false`）
- `GeneralBasicOCR` 时 `LangConverter` 返回 `null` → `throw new Exception($"unsupportted language[{request.Language}]")`
- `TextDetections` 为空 → 返回空 `OcrResult`（无识别内容）

## 不实现的内容（YAGNI）

- 不暴露区域下拉（固定 `ap-shanghai`）
- 不暴露 URL 配置（固定官方端点）
- 不引入 `Newtonsoft.Json`（用 `System.Text.Json`）
- 不做段落级结构化布局聚合（无 paragraphs 概念）
- 不引入 `TencentRegionEnum`（区域固定）
- 不改动 `STranslate.Plugin` 框架、不改其他插件

## 验证

- `dotnet build src/STranslate.slnx` 编译通过
- 插件 DLL 输出到 `.artifacts\Debug\Plugins\STranslate.Plugin.Ocr.Tencent\`
- 设置 UI 可加载、SecretId/SecretKey/Action 修改后持久化
- （可选，需真实 SecretId/SecretKey）对测试图片调用返回行级文本 + 坐标框
