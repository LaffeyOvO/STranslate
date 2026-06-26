# 腾讯 OCR 插件 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 新增内置 OCR 插件 `STranslate.Plugin.Ocr.Tencent`，移植 1.0 腾讯 OCR 的 TC3-HMAC-SHA256 签名鉴权与识别逻辑到 v2.0 插件架构。

**Architecture:** 以现有 Baidu OCR 插件为结构模板（`Main.cs` 实现 `IOcrPlugin`、`Settings.cs` 配置持久化、`View`/`ViewModel` 设置 UI、`Languages` 5 套国际化、`plugin.json`/`icon.png`/`csproj`）。`Main.cs` 内移植 1.0 的 TC3 签名算法与响应 DTO，用 `System.Text.Json` 解析。`PostAsync` 传字符串 body 以保证签名一致。区域固定 `ap-shanghai`，配置项暴露 SecretId/SecretKey/动作下拉。

**Tech Stack:** .NET 10 (net10.0-windows), WPF, CommunityToolkit.Mvvm, System.Text.Json, System.Security.Cryptography。

**参考源码:** 1.0 分支 `src/STranslate/ViewModels/Preference/OCR/TencentOCR .cs`（已下载到 `/tmp/TencentOCR_ref.cs`）。
**设计文档:** `docs/superpowers/specs/2026-06-26-tencent-ocr-plugin-design.md`。

---

## 文件结构

| 文件 | 职责 |
|---|---|
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/STranslate.Plugin.Ocr.Tencent.csproj` | 项目文件，引用 `STranslate.Plugin.csproj`，输出到 artifacts |
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/plugin.json` | 插件元数据（PluginID `bb65c593ebb04d40bc2c5ad55aecc4e2`） |
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/icon.png` | 插件图标（复制 Baidu 的占位） |
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/Settings.cs` | 配置类 + `TencentOCRAction` 枚举 |
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/Extensions.cs` | `EnumExtensions.GetDescription()` |
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/Main.cs` | `IOcrPlugin` 实现 + TC3 签名 + 响应 DTO + `LangConverter` |
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/ViewModel/SettingsViewModel.cs` | 设置 VM，绑定 SecretId/SecretKey/Action |
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/View/SettingsView.xaml` | 设置面板 UI |
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/View/SettingsView.xaml.cs` | UI code-behind |
| `src/Plugins/STranslate.Plugin.Ocr.Tencent/Languages/{zh-cn,en,ja,ko,zh-tw}.{json,xaml}` | 5 套国际化资源 |
| `src/STranslate.slnx` | 修改：OCR 区段追加项目引用 |
| `src/STranslate/Core/Constant.cs` | 修改：`PrePluginIDs` 追加腾讯 ID |

> **测试说明:** 本项目为 WPF 插件，无单元测试工程。验证依赖编译通过 + 人工 UI 检查。每个任务以「编译验证」替代测试步骤。

---

### Task 1: 创建项目骨架（csproj + plugin.json + icon.png）

**Files:**
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/STranslate.Plugin.Ocr.Tencent.csproj`
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/plugin.json`
- Copy: `src/Plugins/STranslate.Plugin.Ocr.Baidu/icon.png` → `src/Plugins/STranslate.Plugin.Ocr.Tencent/icon.png`

- [ ] **Step 1: 创建 csproj**

文件 `src/Plugins/STranslate.Plugin.Ocr.Tencent/STranslate.Plugin.Ocr.Tencent.csproj`：

```xml
<Project Sdk="Microsoft.NET.Sdk">

    <PropertyGroup>
        <TargetFramework>net10.0-windows</TargetFramework>
        <UseWPF>true</UseWPF>
        <AppendTargetFrameworkToOutputPath>false</AppendTargetFrameworkToOutputPath>
        <AppendRuntimeIdentifierToOutputPath>false</AppendRuntimeIdentifierToOutputPath>
        <!--// 编译后打包为插件 //-->
        <!--<EnableAutoPackage>true</EnableAutoPackage>-->
    </PropertyGroup>

    <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|AnyCPU' ">
        <DebugSymbols>true</DebugSymbols>
        <DebugType>portable</DebugType>
        <Optimize>false</Optimize>
        <OutputPath>..\..\.artifacts\Debug\Plugins\STranslate.Plugin.Ocr.Tencent\</OutputPath>
        <DefineConstants>DEBUG;TRACE</DefineConstants>
        <ErrorReport>prompt</ErrorReport>
        <WarningLevel>4</WarningLevel>
        <Prefer32Bit>false</Prefer32Bit>
    </PropertyGroup>

    <PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Release|AnyCPU' ">
        <DebugType>none</DebugType>
        <Optimize>true</Optimize>
        <OutputPath>..\..\.artifacts\Release\Plugins\STranslate.Plugin.Ocr.Tencent\</OutputPath>
        <DefineConstants>TRACE</DefineConstants>
        <ErrorReport>prompt</ErrorReport>
        <WarningLevel>4</WarningLevel>
        <Prefer32Bit>false</Prefer32Bit>
    </PropertyGroup>

    <ItemGroup>
        <Content Include="Languages\*.*">
            <Generator>MSBuild:Compile</Generator>
            <SubType>Designer</SubType>
            <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
        </Content>
        <Content Include="icon.png">
            <CopyToOutputDirectory>Always</CopyToOutputDirectory>
        </Content>
        <Content Include="plugin.json">
            <CopyToOutputDirectory>Always</CopyToOutputDirectory>
        </Content>
    </ItemGroup>

    <ItemGroup>
        <ProjectReference Include="..\..\STranslate.Plugin\STranslate.Plugin.csproj" />
    </ItemGroup>

</Project>
```

- [ ] **Step 2: 创建 plugin.json**

文件 `src/Plugins/STranslate.Plugin.Ocr.Tencent/plugin.json`：

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

- [ ] **Step 3: 复制 icon.png**

Run:
```bash
cp src/Plugins/STranslate.Plugin.Ocr.Baidu/icon.png src/Plugins/STranslate.Plugin.Ocr.Tencent/icon.png
```
Expected: 无输出，文件创建成功。

- [ ] **Step 4: 登记 slnx**

修改 `src/STranslate.slnx`，在 OCR 插件区段（`STranslate.Plugin.Ocr.Google` 那行之后）追加：

```xml
    <Project Path="Plugins/STranslate.Plugin.Ocr.Tencent/STranslate.Plugin.Ocr.Tencent.csproj" />
```

即修改后该区段为：
```xml
    <!-- OCR 插件 -->
    <Project Path="Plugins/STranslate.Plugin.Ocr.Baidu/STranslate.Plugin.Ocr.Baidu.csproj" />
    <Project Path="Plugins/STranslate.Plugin.Ocr.OpenAI/STranslate.Plugin.Ocr.OpenAI.csproj" />
    <Project Path="Plugins/STranslate.Plugin.Ocr.WeChatBuiltIn/STranslate.Plugin.Ocr.WeChatBuiltIn.csproj" />
    <Project Path="Plugins/STranslate.Plugin.Ocr.Google/STranslate.Plugin.Ocr.Google.csproj" />
    <Project Path="Plugins/STranslate.Plugin.Ocr.Tencent/STranslate.Plugin.Ocr.Tencent.csproj" />
```

- [ ] **Step 5: 验证编译**

此时项目内无代码文件，编译会因缺少 `Main` 类等失败属正常。先验证 csproj 能被识别：

Run: `dotnet build src/STranslate.slnx`
Expected: 报错（无 Program/Main），但应能识别项目结构。若报「找不到项目」说明 csproj 路径/slnx 有误。

- [ ] **Step 6: Commit**

```bash
git add src/Plugins/STranslate.Plugin.Ocr.Tencent/STranslate.Plugin.Ocr.Tencent.csproj src/Plugins/STranslate.Plugin.Ocr.Tencent/plugin.json src/Plugins/STranslate.Plugin.Ocr.Tencent/icon.png src/STranslate.slnx
git commit -m "feat(tencent-ocr): scaffold plugin project"
```

---

### Task 2: Settings.cs + Extensions.cs

**Files:**
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/Settings.cs`
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/Extensions.cs`

- [ ] **Step 1: 创建 Settings.cs**

文件 `src/Plugins/STranslate.Plugin.Ocr.Tencent/Settings.cs`：

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

- [ ] **Step 2: 创建 Extensions.cs**

文件 `src/Plugins/STranslate.Plugin.Ocr.Tencent/Extensions.cs`（与 Baidu 插件完全相同）：

```csharp
using System.ComponentModel;

namespace STranslate.Plugin.Ocr.Tencent;

internal static class EnumExtensions
{
    /// <summary>
    /// 获取枚举的 Description 特性值
    /// </summary>
    public static string GetDescription(this Enum value)
    {
        if (value == null)
            return string.Empty;

        var fieldInfo = value.GetType().GetField(value.ToString());
        if (fieldInfo == null)
            return value.ToString();

        var attributes = (DescriptionAttribute[])fieldInfo.GetCustomAttributes(
            typeof(DescriptionAttribute), false);

        return attributes.Length > 0 ? attributes[0].Description : value.ToString();
    }
}
```

- [ ] **Step 3: Commit**

```bash
git add src/Plugins/STranslate.Plugin.Ocr.Tencent/Settings.cs src/Plugins/STranslate.Plugin.Ocr.Tencent/Extensions.cs
git commit -m "feat(tencent-ocr): add Settings and enum extensions"
```

---

### Task 3: Main.cs（核心实现）

**Files:**
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/Main.cs`

> 这是最大的文件。包含 `IOcrPlugin` 实现、TC3 签名算法、响应 DTO、`LangConverter`。逻辑移植自 1.0 `/tmp/TencentOCR_ref.cs`，适配 v2.0 接口。

- [ ] **Step 1: 创建 Main.cs**

文件 `src/Plugins/STranslate.Plugin.Ocr.Tencent/Main.cs`：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using STranslate.Plugin.Ocr.Tencent.View;
using STranslate.Plugin.Ocr.Tencent.ViewModel;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using System.Text.Json.Serialization;
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
            TencentOCRAction.GeneralBasicOCR =>
            [
                LangEnum.Auto,
                LangEnum.ChineseSimplified,
                LangEnum.ChineseTraditional,
                LangEnum.Cantonese,
                LangEnum.English,
                LangEnum.Japanese,
                LangEnum.Korean,
                LangEnum.French,
                LangEnum.Spanish,
                LangEnum.Russian,
                LangEnum.German,
                LangEnum.Italian,
                LangEnum.PortuguesePortugal,
                LangEnum.PortugueseBrazil,
                LangEnum.Vietnamese,
                LangEnum.Thai,
                LangEnum.Malay,
                LangEnum.Arabic,
                LangEnum.Hindi,
                LangEnum.Swedish,
                LangEnum.NorwegianBokmal,
                LangEnum.NorwegianNynorsk,
                LangEnum.Dutch
            ],
            TencentOCRAction.GeneralAccurateOCR =>
            [
                LangEnum.Auto,
                LangEnum.ChineseSimplified,
                LangEnum.ChineseTraditional,
                LangEnum.Cantonese,
                LangEnum.English
            ],
            _ => Enum.GetValues<LangEnum>()
        };

    public bool SupportBoxPoints() => true;

    public Control GetSettingUI()
    {
        _viewModel ??= new SettingsViewModel(Context, Settings);
        _settingUi ??= new SettingsView { DataContext = _viewModel };
        return _settingUi;
    }

    public void Init(IPluginContext context)
    {
        Context = context;
        Settings = context.LoadSettingStorage<Settings>();
    }

    public void Dispose() => _viewModel?.Dispose();

    public async Task<OcrResult> RecognizeAsync(OcrRequest request, CancellationToken cancellationToken)
    {
        var ocrResult = new OcrResult();
        var base64Str = Convert.ToBase64String(request.ImageData);

        // 1. 构造 body
        string body;
        if (Settings.Action == TencentOCRAction.GeneralBasicOCR)
        {
            var target = LangConverter(request.Language)
                ?? throw new Exception($"unsupportted language[{request.Language}]");
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

        // 3. POST（传字符串 body，保证签名与实际 body 一致）
        var resp = await Context.HttpService.PostAsync(Url, body, options, cancellationToken);
        if (string.IsNullOrEmpty(resp))
            throw new Exception("请求结果为空");

        // 4. 解析
        var parsedData = JsonSerializer.Deserialize<Root>(resp)
            ?? throw new Exception($"反序列化失败: {resp}");

        // 判断是否出错
        if (parsedData.Response.Error != null)
            return ocrResult.Fail(parsedData.Response.Error.Message);

        // 提取内容
        foreach (var item in parsedData.Response.TextDetections)
        {
            var content = new OcrContent { Text = item.DetectedText };
            foreach (var pg in item.Polygon)
                content.BoxPoints.Add(new BoxPoint(pg.X, pg.Y));
            ocrResult.OcrContents.Add(content);
        }

        return ocrResult;
    }

    /// <summary>
    ///     https://cloud.tencent.com/document/product/866/33526
    /// </summary>
    public string? LangConverter(LangEnum lang)
    {
        return lang switch
        {
            LangEnum.Auto => "auto",
            LangEnum.ChineseSimplified => "zh",
            LangEnum.ChineseTraditional => "zh_rare",
            LangEnum.Cantonese => "zh_rare",
            LangEnum.English => "auto",
            LangEnum.Japanese => "jap",
            LangEnum.Korean => "kor",
            LangEnum.French => "fre",
            LangEnum.Spanish => "spa",
            LangEnum.Russian => "rus",
            LangEnum.German => "ger",
            LangEnum.Italian => "ita",
            LangEnum.Turkish => null,
            LangEnum.PortuguesePortugal => "por",
            LangEnum.PortugueseBrazil => "por",
            LangEnum.Vietnamese => "vie",
            LangEnum.Thai => "tha",
            LangEnum.Malay => "may",
            LangEnum.Arabic => "ara",
            LangEnum.Hindi => "hi",
            LangEnum.Indonesian => null,
            LangEnum.MongolianCyrillic => null,
            LangEnum.MongolianTraditional => null,
            LangEnum.Khmer => null,
            LangEnum.NorwegianBokmal => "nor",
            LangEnum.NorwegianNynorsk => "nor",
            LangEnum.Persian => null,
            LangEnum.Swedish => "swe",
            LangEnum.Polish => null,
            LangEnum.Dutch => "hol",
            LangEnum.Ukrainian => null,
            LangEnum.Uzbek => null,
            _ => "auto"
        };
    }

    #region Tencent Official Support (TC3-HMAC-SHA256)

    protected static string GetAuth(
        string secretId, string secretKey, string host, string contentType,
        string timestamp, string body
    )
    {
        var canonicalURI = "/";
        var canonicalHeaders = "content-type:" + contentType + "\nhost:" + host + "\n";
        var signedHeaders = "content-type;host";
        var hashedRequestPayload = Sha256Hex(body);
        var canonicalRequest = "POST" + "\n"
                                      + canonicalURI + "\n"
                                      + "\n"
                                      + canonicalHeaders + "\n"
                                      + signedHeaders + "\n"
                                      + hashedRequestPayload;

        var algorithm = "TC3-HMAC-SHA256";
        var date = new DateTime(1970, 1, 1, 0, 0, 0, 0, DateTimeKind.Utc).AddSeconds(int.Parse(timestamp))
            .ToString("yyyy-MM-dd");
        var service = host.Split(".")[0];
        var credentialScope = date + "/" + service + "/" + "tc3_request";
        var hashedCanonicalRequest = Sha256Hex(canonicalRequest);
        var stringToSign = algorithm + "\n"
                                     + timestamp + "\n"
                                     + credentialScope + "\n"
                                     + hashedCanonicalRequest;

        var tc3SecretKey = Encoding.UTF8.GetBytes("TC3" + secretKey);
        var secretDate = HmacSha256(tc3SecretKey, Encoding.UTF8.GetBytes(date));
        var secretService = HmacSha256(secretDate, Encoding.UTF8.GetBytes(service));
        var secretSigning = HmacSha256(secretService, Encoding.UTF8.GetBytes("tc3_request"));
        var signatureBytes = HmacSha256(secretSigning, Encoding.UTF8.GetBytes(stringToSign));
        var signature = BitConverter.ToString(signatureBytes).Replace("-", "").ToLower();

        return algorithm + " "
                         + "Credential=" + secretId + "/" + credentialScope + ", "
                         + "SignedHeaders=" + signedHeaders + ", "
                         + "Signature=" + signature;
    }

    protected static string Sha256Hex(string s)
    {
        using var algo = SHA256.Create();
        var hashbytes = algo.ComputeHash(Encoding.UTF8.GetBytes(s));
        var builder = new StringBuilder();
        for (var i = 0; i < hashbytes.Length; ++i)
            builder.Append(hashbytes[i].ToString("x2"));

        return builder.ToString();
    }

    private static byte[] HmacSha256(byte[] key, byte[] msg)
    {
        using var mac = new HMACSHA256(key);
        return mac.ComputeHash(msg);
    }

    #endregion Tencent Official Support (TC3-HMAC-SHA256)

    #region Response DTO

#pragma warning disable IDE1006 // 命名样式
    public class PolygonItem
    {
        [JsonPropertyName("X")] public int X { get; set; }

        [JsonPropertyName("Y")] public int Y { get; set; }
    }

    public class TextDetectionsItem
    {
        [JsonPropertyName("DetectedText")] public string DetectedText { get; set; } = string.Empty;

        [JsonPropertyName("Confidence")] public int Confidence { get; set; }

        [JsonPropertyName("Polygon")] public List<PolygonItem> Polygon { get; set; } = [];

        [JsonPropertyName("AdvancedInfo")] public string AdvancedInfo { get; set; } = string.Empty;
    }

    public class Error
    {
        [JsonPropertyName("Code")] public string Code { get; set; } = string.Empty;

        [JsonPropertyName("Message")] public string Message { get; set; } = string.Empty;
    }

    public class Response
    {
        [JsonPropertyName("TextDetections")] public List<TextDetectionsItem> TextDetections { get; set; } = [];

        [JsonPropertyName("Error")] public Error? Error { get; set; }

        [JsonPropertyName("RequestId")] public string RequestId { get; set; } = string.Empty;

        [JsonPropertyName("Language")] public string Language { get; set; } = string.Empty;
    }

    public class Root
    {
        [JsonPropertyName("Response")] public Response Response { get; set; } = new();
    }
#pragma warning restore IDE1006 // 命名样式

    #endregion Response DTO
}
```

- [ ] **Step 2: Commit（暂不编译，因 View/ViewModel 尚未创建）**

```bash
git add src/Plugins/STranslate.Plugin.Ocr.Tencent/Main.cs
git commit -m "feat(tencent-ocr): implement Main with TC3 signature and OCR logic"
```

---

### Task 4: ViewModel/SettingsViewModel.cs

**Files:**
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/ViewModel/SettingsViewModel.cs`

- [ ] **Step 1: 创建 SettingsViewModel.cs**

文件 `src/Plugins/STranslate.Plugin.Ocr.Tencent/ViewModel/SettingsViewModel.cs`（仿 Baidu，绑定 Action/SecretId/SecretKey）：

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using System.ComponentModel;

namespace STranslate.Plugin.Ocr.Tencent.ViewModel;

public partial class SettingsViewModel : ObservableObject, IDisposable
{
    private readonly IPluginContext _context;
    private readonly Settings _settings;

    public SettingsViewModel(IPluginContext context, Settings settings)
    {
        _context = context;
        _settings = settings;

        Action = settings.Action;
        SecretId = settings.SecretId;
        SecretKey = settings.SecretKey;

        PropertyChanged += OnPropertyChanged;
    }

    private void OnPropertyChanged(object? sender, PropertyChangedEventArgs e)
    {
        switch (e.PropertyName)
        {
            case nameof(Action):
                _settings.Action = Action;
                break;
            case nameof(SecretId):
                _settings.SecretId = SecretId;
                break;
            case nameof(SecretKey):
                _settings.SecretKey = SecretKey;
                break;
            default:
                return;
        }
        _context.SaveSettingStorage<Settings>();
    }

    public List<TencentOCRAction> Actions => [.. Enum.GetValues<TencentOCRAction>()];

    [ObservableProperty] public partial TencentOCRAction Action { get; set; }
    [ObservableProperty] public partial string SecretId { get; set; }
    [ObservableProperty] public partial string SecretKey { get; set; }

    public void Dispose() => PropertyChanged -= OnPropertyChanged;
}
```

- [ ] **Step 2: Commit**

```bash
git add src/Plugins/STranslate.Plugin.Ocr.Tencent/ViewModel/SettingsViewModel.cs
git commit -m "feat(tencent-ocr): add SettingsViewModel"
```

---

### Task 5: View/SettingsView.xaml(.cs)

**Files:**
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/View/SettingsView.xaml`
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/View/SettingsView.xaml.cs`

- [ ] **Step 1: 创建 SettingsView.xaml**

文件 `src/Plugins/STranslate.Plugin.Ocr.Tencent/View/SettingsView.xaml`（仿 Baidu，资源键前缀改为 `STranslate_Plugin_Ocr_Tencent_`，官网链接改为腾讯）：

```xml
<UserControl
    x:Class="STranslate.Plugin.Ocr.Tencent.View.SettingsView"
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
    xmlns:i="http://schemas.microsoft.com/xaml/behaviors"
    xmlns:ikw="http://schemas.inkore.net/lib/ui/wpf"
    xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
    xmlns:plugin="clr-namespace:STranslate.Plugin;assembly=STranslate.Plugin"
    xmlns:s="https://github.com/zggsong/2022/xaml"
    xmlns:ui="http://schemas.inkore.net/lib/ui/wpf/modern"
    xmlns:vm="clr-namespace:STranslate.Plugin.Ocr.Tencent.ViewModel"
    d:DataContext="{d:DesignInstance Type=vm:SettingsViewModel}"
    d:DesignHeight="450"
    d:DesignWidth="800"
    mc:Ignorable="d">

    <ikw:SimpleStackPanel Spacing="12">
        <ui:SettingsCard Header="{DynamicResource STranslate_Plugin_Ocr_Tencent_SecretId}">
            <ui:SettingsCard.HeaderIcon>
                <ui:FontIcon Icon="{x:Static ui:FluentSystemIcons.Key_24_Regular}" />
            </ui:SettingsCard.HeaderIcon>
            <PasswordBox
                MinWidth="300"
                plugin:PasswordBoxAssistant.Attach="True"
                plugin:PasswordBoxAssistant.Password="{Binding SecretId}" />
        </ui:SettingsCard>

        <ui:SettingsCard Header="{DynamicResource STranslate_Plugin_Ocr_Tencent_SecretKey}">
            <ui:SettingsCard.HeaderIcon>
                <ui:FontIcon Icon="{x:Static ui:FluentSystemIcons.PersonKey_20_Regular}" />
            </ui:SettingsCard.HeaderIcon>
            <PasswordBox
                MinWidth="300"
                plugin:PasswordBoxAssistant.Attach="True"
                plugin:PasswordBoxAssistant.Password="{Binding SecretKey}" />
        </ui:SettingsCard>

        <ui:SettingsCard Description="{DynamicResource STranslate_Plugin_Ocr_Tencent_Action_Description}" Header="{DynamicResource STranslate_Plugin_Ocr_Tencent_Action}">
            <ui:SettingsCard.HeaderIcon>
                <ui:FontIcon Icon="{x:Static ui:FluentSystemIcons.TaskListSquareLtr_20_Regular}" />
            </ui:SettingsCard.HeaderIcon>
            <ComboBox ItemsSource="{Binding Actions}" SelectedItem="{Binding Action}" />
        </ui:SettingsCard>

        <ui:SettingsCard Description="{DynamicResource STranslate_Plugin_Ocr_Tencent_Official_Description}" Header="{DynamicResource STranslate_Plugin_Ocr_Tencent_Official}">
            <ui:SettingsCard.HeaderIcon>
                <ui:FontIcon Icon="{x:Static ui:FluentSystemIcons.WebAsset_20_Regular}" />
            </ui:SettingsCard.HeaderIcon>
            <ui:HyperlinkButton Content="https://cloud.tencent.com/product/ocr" NavigateUri="https://cloud.tencent.com/product/ocr" />
        </ui:SettingsCard>
    </ikw:SimpleStackPanel>
</UserControl>
```

- [ ] **Step 2: 创建 SettingsView.xaml.cs**

文件 `src/Plugins/STranslate.Plugin.Ocr.Tencent/View/SettingsView.xaml.cs`：

```csharp
namespace STranslate.Plugin.Ocr.Tencent.View;

public partial class SettingsView
{
    public SettingsView() => InitializeComponent();
}
```

- [ ] **Step 3: Commit**

```bash
git add src/Plugins/STranslate.Plugin.Ocr.Tencent/View/SettingsView.xaml src/Plugins/STranslate.Plugin.Ocr.Tencent/View/SettingsView.xaml.cs
git commit -m "feat(tencent-ocr): add SettingsView"
```

---

### Task 6: Languages 5 套国际化资源

**Files:**
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/Languages/zh-cn.{json,xaml}`
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/Languages/en.{json,xaml}`
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/Languages/ja.{json,xaml}`
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/Languages/ko.{json,xaml}`
- Create: `src/Plugins/STranslate.Plugin.Ocr.Tencent/Languages/zh-tw.{json,xaml}`

- [ ] **Step 1: zh-cn.json + zh-cn.xaml**

`Languages/zh-cn.json`:
```json
{
  "Name": "腾讯 OCR",
  "Description": "适用于 STranslate 的腾讯云 OCR 插件"
}
```

`Languages/zh-cn.xaml`:
```xml
<ResourceDictionary
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:sys="clr-namespace:System;assembly=mscorlib">

    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretId">SecretId</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretKey">SecretKey</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action">识别版本</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action_Description">「通用印刷体识别」为多语种通用版，「通用印刷体识别(高精度版)」为中英文高精度版</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official">官方网站</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official_Description">点击下面连接跳转官方网站进行注册使用</sys:String>

</ResourceDictionary>
```

- [ ] **Step 2: en.json + en.xaml**

`Languages/en.json`:
```json
{
  "Name": "Tencent OCR",
  "Description": "Tencent Cloud OCR plugin for stranslate"
}
```

`Languages/en.xaml`:
```xml
<ResourceDictionary
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:sys="clr-namespace:System;assembly=mscorlib">

    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretId">SecretId</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretKey">SecretKey</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action">Recognition Version</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action_Description">"GeneralBasicOCR" is the multilingual general version, "GeneralAccurateOCR" is the high-precision version for Chinese and English.</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official">Official Website</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official_Description">Click the link below to go to the official website for registration and use.</sys:String>

</ResourceDictionary>
```

- [ ] **Step 3: zh-tw.json + zh-tw.xaml**

`Languages/zh-tw.json`:
```json
{
  "Name": "騰訊 OCR",
  "Description": "適用於 STranslate 的騰訊雲 OCR 插件"
}
```

`Languages/zh-tw.xaml`:
```xml
<ResourceDictionary
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:sys="clr-namespace:System;assembly=mscorlib">

    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretId">SecretId</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretKey">SecretKey</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action">辨識版本</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action_Description">「通用印刷體識別」為多語種通用版，「通用印刷體識別(高精度版)」為中英文高精度版</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official">官方網站</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official_Description">點擊下面連結跳轉官方網站進行註冊使用。</sys:String>

</ResourceDictionary>
```

- [ ] **Step 4: ja.json + ja.xaml**

`Languages/ja.json`:
```json
{
  "Name": "Tencent OCR",
  "Description": "STranslate用のTencent Cloud OCRプラグイン"
}
```

`Languages/ja.xaml`:
```xml
<ResourceDictionary
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:sys="clr-namespace:System;assembly=mscorlib">

    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretId">SecretId</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretKey">SecretKey</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action">認識バージョン</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action_Description">「GeneralBasicOCR」は多言語汎用版、「GeneralAccurateOCR」は中国語・英語の高精度版です。</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official">公式ウェブサイト</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official_Description">以下のリンクをクリックして、登録および使用のために公式ウェブサイトにアクセスしてください。</sys:String>

</ResourceDictionary>
```

- [ ] **Step 5: ko.json + ko.xaml**

`Languages/ko.json`:
```json
{
  "Name": "Tencent OCR",
  "Description": "STranslate용 Tencent Cloud OCR 플러그인"
}
```

`Languages/ko.xaml`:
```xml
<ResourceDictionary
    xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
    xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
    xmlns:sys="clr-namespace:System;assembly=mscorlib">

    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretId">SecretId</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_SecretKey">SecretKey</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action">인식 버전</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Action_Description">"GeneralBasicOCR"은 다국어 일반 버전, "GeneralAccurateOCR"은 중국어·영어 고정밀 버전입니다.</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official">공식 웹사이트</sys:String>
    <sys:String x:Key="STranslate_Plugin_Ocr_Tencent_Official_Description">아래 링크를 클릭하여 등록 및 사용을 위해 공식 웹사이트로 이동하세요.</sys:String>

</ResourceDictionary>
```

- [ ] **Step 6: Commit**

```bash
git add src/Plugins/STranslate.Plugin.Ocr.Tencent/Languages/
git commit -m "feat(tencent-ocr): add 5-language i18n resources"
```

---

### Task 7: 编译验证 + 修复

**Files:**
- 无新建，仅验证与修复

- [ ] **Step 1: 编译整个解决方案**

Run: `dotnet build src/STranslate.slnx`
Expected: 编译通过（0 error）。若报错，记录错误信息。

- [ ] **Step 2: 若有编译错误则修复**

常见可能错误与修复：
- `Namespace STranslate.Plugin.Ocr.Tencent.ViewModel not found` → 检查 `SettingsViewModel.cs` 的 namespace 是否为 `STranslate.Plugin.Ocr.Tencent.ViewModel`
- `Main.cs 中 PostAsync 参数类型不匹配` → 确认 `body` 为 `string`，`PostAsync(string, object, ...)` 接受 string（object 兼容）
- `JsonSerializer.Deserialize<Root>` 反序列化属性大小写问题 → 确认所有 DTO 属性都加了 `[JsonPropertyName("PascalCase")]`
- WPF XAML 命名空间错误 → 确认 `xmlns:vm` 的 `clr-namespace` 与 `assembly` 正确

修复后重新运行 Step 1 直至 0 error。

- [ ] **Step 3: 验证插件输出**

Run: `ls src/.artifacts/Debug/Plugins/STranslate.Plugin.Ocr.Tencent/`
Expected: 包含 `STranslate.Plugin.Ocr.Tencent.dll`、`plugin.json`、`icon.png`、`Languages/` 目录。

- [ ] **Step 4: Commit（如有修复）**

```bash
git add -A
git commit -m "fix(tencent-ocr): resolve build errors"
```
（若 Step 2 无修复则跳过本步）

---

### Task 8: 登记 PrePluginIDs

**Files:**
- Modify: `src/STranslate/Core/Constant.cs:56-61`（`PrePluginIDs` 列表）

- [ ] **Step 1: 追加腾讯 ID 到 PrePluginIDs**

修改 `src/STranslate/Core/Constant.cs`，在 `GoogleOCR` 那行之后追加一行（OCR 区段内）：

修改前：
```csharp
        "2e83ee2f5dbf45249a3bd1457a326abf", //GoogleOCR
```

修改后：
```csharp
        "2e83ee2f5dbf45249a3bd1457a326abf", //GoogleOCR
        "bb65c593ebb04d40bc2c5ad55aecc4e2", //TencentOCR
```

> ID 必须与 `plugin.json` 的 `PluginID` 完全一致，否则插件不会被识别为预装插件。

- [ ] **Step 2: 重新编译验证**

Run: `dotnet build src/STranslate.slnx`
Expected: 0 error。

- [ ] **Step 3: Commit**

```bash
git add src/STranslate/Core/Constant.cs
git commit -m "feat(tencent-ocr): register as preinstalled plugin"
```

---

## 完成检查清单

- [ ] `dotnet build src/STranslate.slnx` 0 error
- [ ] `.artifacts/Debug/Plugins/STranslate.Plugin.Ocr.Tencent/` 含 dll/plugin.json/icon.png/Languages/
- [ ] `plugin.json` PluginID = `bb65c593ebb04d40bc2c5ad55aecc4e2`
- [ ] `Constant.cs` PrePluginIDs 含同 ID
- [ ] `STranslate.slnx` 含新项目引用
- [ ] 5 套语言资源文件齐全（zh-cn/en/ja/ko/zh-tw 的 .json + .xaml）
- [ ] 所有改动已 commit

## 验证（可选，需真实密钥）

填入腾讯云 SecretId/SecretKey 后，对测试图片调用应返回行级文本 + 坐标框。`GeneralBasicOCR` 支持多语种，`GeneralAccurateOCR` 仅中英文高精度。
