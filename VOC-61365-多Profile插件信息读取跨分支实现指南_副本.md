# VOC-61365 多 Profile 插件信息读取——跨分支重新实现指南

日期：2026-07-21  
适用平台：macOS  
目标浏览器：Google Chrome、Microsoft Edge、Mozilla Firefox  
文档用途：在与当前实现差异较大的分支上，按行为契约重新实现，而不是机械套用现有 diff。

> 重要：当前参考实现位于 `codex/voc-61365-chromium-profile-detection` 工作分支，且部分新增文件仍是未跟踪文件。切换、清理或删除当前 worktree 前，应先保留本文件，并在新分支实现、验证完成前保留当前 worktree 作为只读参考。

## 1. 一页结论

本次需求只改变“从哪个浏览器 Profile 读取插件版本和既有协议时间字段”，不改变浏览器插件的安装/卸载、Socket 心跳、窗口状态、健康状态机或上报协议。

在差异较大的分支上，建议仍保持两个边界：

1. 抽出一个只依赖 Foundation 和 macOS 系统 API 的共享读取器，根据浏览器 PID 获取真实 argv/cwd，再定位 Profile 并读取插件元数据。
2. `BrowserMonitor` 只把已经得到的浏览器 PID 传给读取器，并继续使用原有健康状态机处理返回的 `version`、`init_time`。

最终健康数据格式保持：

```json
{
  "version": "3.1.5",
  "init_time": 1700000000
}
```

以下内容不在本次修改范围：

- 不用 Socket 获取版本，也不删除现有 Socket 检测。
- 不改变 `normal`、`damaged`、`nonexistent`、`no_browser` 等状态判定。
- 不改变插件安装、卸载和策略写入。
- 不改变服务端接口或增加 Profile 明细字段。
- 不支持 Brave、Prisma Access Browser 等额外浏览器，除非目标分支明确要求扩展范围。

插件 ID 的默认来源必须继续使用：

```text
/opt/.yunshu/config/BrowserPluginIDs
```

当前机器上的键值仅用于核对，不应在产品代码中硬编码：

| 浏览器 | plist key | 当前值（2026-07-21） |
|---|---|---|
| Chrome | `com.google.Chrome` | `akceklnjbjbeeocfimcgigghddckeimd` |
| Edge | `com.microsoft.Edge` | `ljfbppcmojnhbcobpelgmijdeckmpakg` |
| Firefox | `org.mozilla.firefox` | `dlp_group@eaglecloud.com` |

## 2. 问题与根因

VOC-61365 的原始场景中，Chrome 通过下面的参数使用临时用户目录：

```text
--user-data-dir=/tmp/chrome-debug-session
```

运行实例中的插件版本是 `3.1.5`，但客户端固定读取：

```text
~/Library/Application Support/Google/Chrome/Default/Secure Preferences
```

因此读到了另一个 Profile 中的旧版本 `3.1.2`。

根因是旧逻辑同时做了两个不成立的假设：

- 浏览器总是使用默认 user data 目录。
- 浏览器总是使用 `Default` Profile。

同类问题也存在于 Edge。Firefox 的文件格式不同，但当它通过 `--profile`、`-profile` 或 `-P` 使用其他 Profile 时，固定枚举或固定读取默认 Profile 同样可能选错。

插件版本是 Profile 维度的数据。正确策略不是“扫描后取最高版本”，而是尽量绑定当前 PID 的运行参数；无法精确绑定时，才按稳定的候选顺序取第一个成功结果。

## 3. 必须保持的行为契约

### 3.1 输入

共享读取器至少需要以下输入：

- `browser`：规范化为 `chrome`、`edge`、`firefox`。
- `pid`：当前浏览器主进程 PID；PID 无效或进程信息不可读时允许回退默认目录。
- `extensionID`：从已有缓存或 `BrowserPluginIDs` 读取。
- `userHome`：浏览器所属 console user 的 home，不能盲用 Manager/root 进程自己的 `NSHomeDirectory()`。

为了可重复测试，还应提供一个不读取真实进程的入口，直接传入：

- argv 数组；
- 进程工作目录；
- 其余输入同上。

### 3.2 输出

建议使用结果对象分离“健康协议字段”和“诊断信息”：

```objc
@interface YSBrowserExtensionInfoResult : NSObject
@property (nonatomic, copy, readonly, nullable) NSDictionary *extensionInfo;
@property (nonatomic, copy, readonly) NSString *source;
@property (nonatomic, copy, readonly, nullable) NSString *profilePath;
@property (nonatomic, copy, readonly, nullable) NSString *userDataPath;
@end

@interface YSBrowserExtensionInfoReader : NSObject
+ (nullable YSBrowserExtensionInfoResult *)readBrowser:(NSString *)browser
                                                   pid:(int)pid
                                           extensionID:(NSString *)extensionID
                                              userHome:(NSString *)userHome;

+ (nullable YSBrowserExtensionInfoResult *)readBrowser:(NSString *)browser
                                             arguments:(NSArray<NSString *> *)arguments
                                      workingDirectory:(nullable NSString *)workingDirectory
                                           extensionID:(NSString *)extensionID
                                              userHome:(NSString *)userHome;

+ (NSArray<NSString *> *)processArgumentsForPID:(int)pid;
+ (nullable NSString *)workingDirectoryForPID:(int)pid;
@end
```

`extensionInfo` 只允许进入原健康上报协议的字段：

| 字段 | 类型 | 含义 |
|---|---|---|
| `version` | string | 当前选择 Profile 中的插件版本 |
| `init_time` | number | Unix 秒级时间戳 |

`source`、`profilePath`、`userDataPath` 只用于日志、CLI 和测试，不要混入服务端健康数据。

`init_time` 是沿用的协议字段，但两类浏览器的原始语义并不完全相同：Chromium 映射 `last_update_time`（最后更新时间），Firefox 映射 `installDate`（安装时间）。不能把它对外统一宣称为严格的“插件安装时间”。

### 3.3 “未找到”与“无效输入”

- 支持的浏览器、合法的 `extensionID` 和 `userHome`，即使未找到插件，也返回一个结果对象；此时 `extensionInfo == nil`。
- 不支持的浏览器或缺少必要输入，返回 `nil`。
- `BrowserMonitor` 继续把空 `extensionInfo` 交给原逻辑判定为 `nonexistent`，不要在读取器里产生新的健康状态。

### 3.4 诊断来源值

建议稳定保留这些 `source` 值，便于 CLI 和日志定位：

| source | 含义 |
|---|---|
| `default` | Chromium 未指定有效自定义 user data，读取默认目录 |
| `runtime` | Chromium 使用了可读取的 `--user-data-dir` |
| `default_after_invalid_runtime` | Chromium 指定路径无效/不可读，回退默认目录 |
| `runtime_path` | Firefox 使用有效的 `--profile` / `-profile` |
| `runtime_name` | Firefox 使用有效的 `-P` 命名 Profile |
| `profiles_ini` | Firefox 从 `profiles.ini` 候选命中 |
| `profile_scan` | Firefox 目录扫描命中，或最终未命中 |

诊断字段在关键分支上的结果必须一致：

| 场景 | source | profilePath | userDataPath | extensionInfo |
|---|---|---|---|---|
| Chromium 默认目录命中 | `default` | 命中的 Profile 目录 | 默认 user data | `{version, init_time}` |
| Chromium 有效 runtime 命中 | `runtime` | runtime 内命中的 Profile | runtime | `{version, init_time}` |
| Chromium 有效但空的 runtime | `runtime` | `nil` | runtime | `nil` |
| Chromium runtime 无效，默认目录命中 | `default_after_invalid_runtime` | 默认目录内命中的 Profile | 默认 user data | `{version, init_time}` |
| Chromium 最终未命中 | 当前 Chromium source | `nil` | 最终选定的 user data；无法构造时为 `nil` | `nil` |
| Firefox 有效显式路径但未装插件 | `runtime_path` | 显式 Profile 目录 | Firefox 默认基目录 | `nil` |
| Firefox 有效命名 Profile 但未装插件 | `runtime_name` | 命名 Profile 目录 | Firefox 默认基目录 | `nil` |
| Firefox INI 候选命中 | `profiles_ini` | 命中的 Profile 目录 | Firefox 默认基目录 | `{version, init_time}` |
| Firefox 扫描命中 | `profile_scan` | 命中的 Profile 目录 | Firefox 默认基目录 | `{version, init_time}` |
| Firefox 最终未命中 | `profile_scan` | `nil` | Firefox 默认基目录 | `nil` |

## 4. 推荐架构与调用流

```text
BrowserMonitor.healthInfoByBrowser
              │ 已有 browser + pid
              ▼
getExtensionManifest(browser, pid)
              │ 已有插件 ID + console user home
              ▼
YSBrowserExtensionInfoReader
      ├─ PID -> argv（KERN_PROCARGS2）
      ├─ PID -> cwd（proc_pidinfo）
      ├─ Chromium Profile 选择 + Secure Preferences
      └─ Firefox Profile 选择 + extensions.json
              │
              ▼
       { version, init_time }
              │
              ▼
原有 Socket/心跳/窗口/健康状态机（保持不变）
```

CLI demo 直接调用同一个共享读取器，不能复制一份选择算法，否则 demo 通过并不能证明产品代码正确。

共享读取器应避免依赖 `BrowserMonitor`、`Utility`、日志框架或工程内其他业务类，使它可以被独立 `clang` 编译并用于契约测试。

## 5. 获取真实进程上下文

### 5.1 为什么必须保留 argv 边界

不能用把整条命令拼成字符串的 `getCommandLine:` 再按空格拆分。`--user-data-dir`、Profile 路径或 cwd 可能包含空格；字符串拆分会产生错误路径。

### 5.2 PID -> argv

macOS 实现使用 `sysctl`：

1. 通过 `CTL_KERN/KERN_ARGMAX` 取得缓冲区大小。
2. 通过 `CTL_KERN/KERN_PROCARGS2/<pid>` 读取进程参数。
3. 缓冲区开头是 `argc`。
4. 跳过可执行文件路径和连续 NUL。
5. 精确读取 `argc` 个 NUL 分隔的 UTF-8 argv 项。
6. 任一边界或编码校验失败时返回空数组，不返回半截 argv。

读取失败可能来自 PID 已退出、权限限制或竞态。失败时不要报错终止健康检查，而应让浏览器选择逻辑走默认回退。

### 5.3 PID -> cwd

使用：

```text
proc_pidinfo(pid, PROC_PIDVNODEPATHINFO, ...)
```

读取 `pvi_cdir.vip_path`。cwd 主要用于解析 Firefox 的相对 `-profile` 路径；无法取得 cwd 时，相对路径视为无效并进入 Firefox 配置回退。

### 5.4 参数解析通用规则

- 遇到独立的 `--` 后停止解析，之后的内容都是位置参数。
- 保留空格，不对 argv 项再次 shell 分词。
- 不记录完整 argv 到常规产品日志；argv 可能包含 URL、文件名或其他敏感信息。
- CLI 只有显式指定 `--debug-argv` 时才输出完整 argv。

当前参考实现的参数语义为：

- Chromium：扫描到 `--` 为止，最后一个非空的同名原始参数值生效，再做路径/Profile 合法性校验。最后一个值非法时不会恢复更早的合法值。
- Firefox：显式路径参数族整体优先于命名 Profile 参数族；每个参数族内部只取 argv 中第一个匹配项。完整真值表见 7.2 节。

如果目标分支统一参数工具改变了这一点，必须同时更新契约测试，不能无意改变。

## 6. Chrome / Edge 实现契约

### 6.1 默认目录

| browser | 默认 user data 目录 |
|---|---|
| `chrome` | `$userHome/Library/Application Support/Google/Chrome` |
| `edge` | `$userHome/Library/Application Support/Microsoft Edge` |

注意 Edge 的应用 bundle ID、进程映射 key 和插件 ID plist key 在旧分支里可能不完全相同。插件 ID 必须使用 `com.microsoft.Edge` 对应值，不要误用应用安装检测常量。

### 6.2 支持的命令行参数

- `--user-data-dir=/path`
- `--user-data-dir /path`，读取器防御性支持；Chrome/Edge 自身在真实启动中应使用等号形式。
- `--profile-directory=Profile 1`
- `--profile-directory "Profile 1"` 对应 argv 中的两个完整项。

`--user-data-dir` 路径规则：

- 支持绝对路径、`~` 和 `~/...`。
- `~` 必须用传入的浏览器用户 home 展开，而不是 Manager 自身 home。
- 相对路径无效。
- 标准化路径后再读取目录。

Profile 名称规则：

- 必须是单个目录名，`lastPathComponent` 必须等于原值。
- 拒绝包含 `/` 的值和目录穿越。
- 忽略隐藏目录。
- 忽略 `System Profile` 和 `Guest Profile`。

### 6.3 user data 目录的隔离语义

这是跨分支迁移时最容易误改的规则：

- 未指定 `--user-data-dir`：读取默认 user data 目录。
- 指定的路径存在且可枚举：只在该运行时目录内查找，哪怕里面没有插件，也不能跨到 home 默认目录。
- 指定的路径不存在、不是可读目录或枚举失败：回退默认目录，并标记 `default_after_invalid_runtime`。

换言之，“有效但空”与“无效/不可读”必须区分。这样才能避免临时 Chrome 实例没有插件时，被默认 Profile 中的插件误判为当前实例已安装。

### 6.4 Profile 候选顺序

在最终选定的 user data 目录内，按下列顺序去重：

1. `--profile-directory`
2. `Local State.profile.last_used`
3. `Local State.profile.last_active_profiles` 中的每一项
4. `Default`
5. user data 目录下其余合法 Profile，按 `localizedStandardCompare` 排序

候选只有存在普通文件：

```text
{userDataPath}/{profileName}/Secure Preferences
```

时才进入 JSON 解析。

伪代码：

```text
defaultPath = browserDefaultPath(userHome)
requested = argvValue("--user-data-dir")

if requested is readable directory:
    userDataPath = requested
    source = runtime
else:
    userDataPath = defaultPath
    source = requested ? default_after_invalid_runtime : default

if userDataPath cannot be enumerated:
    return result(nil, source, nil, userDataPath)

candidates = dedupe([
    argvValue("--profile-directory"),
    LocalState.profile.last_used,
    ...LocalState.profile.last_active_profiles,
    "Default",
    ...sortedDirectoryScan
])

for profile in candidates:
    info = parseSecurePreferences(profile)
    if info is valid:
        return first successful info

return result(nil, source, nil, userDataPath)
```

如果候选文件不存在、JSON 损坏、插件 ID 不存在、`version` 或 `last_update_time` 无效，应继续下一个候选。

禁止把所有候选版本收集后取最高版本。能确定显式 Profile 时，它必须优先；不能确定时也只能按上述候选顺序取第一个有效结果。

当前已经验证的 Chromium 行为是“显式 Profile 优先，但不隔离”：即使 `--profile-directory` 语法合法，只要它的 Secure Preferences 缺失、JSON 损坏、未含目标插件或字段无效，仍继续尝试 Local State、`Default` 和扫描候选。这与 Firefox 的显式 Profile 隔离语义不同，是原始 VOC 候选回退要求和现有 31 个用例所对应的行为。跨分支移植时必须明确保留，不能凭直觉改成隔离。

该行为存在已知误报窗口：当前显式 Chromium Profile 未装插件，但同一 user data 下其他 Profile 已安装时，可能返回其他 Profile 的版本。如果产品希望改成“显式 Chromium Profile 严格隔离”，应作为新的行为变更单独批准，并新增“显式 Profile 未装插件必须返回 nil”的契约测试；不能在本次移植中静默改变。

### 6.5 Secure Preferences 读取

读取路径：

```text
extensions.settings[extensionID].manifest.version
extensions.settings[extensionID].last_update_time
```

当前实现对“有效”的精确定义是：

- `extensions`、`settings`、目标 extension 和 `manifest` 都必须是 JSON object。
- `version` 必须是非空 JSON string。
- `last_update_time` 必须是非空 JSON string。
- 时间转换直接使用 Objective-C `longLongValue`；当前没有额外的十进制格式、负数、溢出或 epoch 前范围校验。为保持跨分支一致性，应先复刻该行为；若要收紧校验，需单独评审并新增失败路径测试。

两个字段满足上述类型/非空条件后返回：

```objc
long long initTime = [lastUpdateTime longLongValue] / 1000000LL
                   - 11644473600LL;
```

即把 Chromium 使用的 1601 epoch 微秒值转换为 Unix 秒。

### 6.6 “未指定 user_dir，但打开其他 Profile”是否可识别

可以，但精度分层：

- 有 `--profile-directory`：精确优先读取该 Profile。
- 没有 `--profile-directory`：先用默认 user data 下的 `Local State.last_used` 和 `last_active_profiles`。
- Local State 不能命中：最后扫描所有 Profile，找到第一个含合法插件信息的 Profile。

最后两种属于磁盘候选推断，不保证能区分同一浏览器主进程中同时打开的多个 Profile 窗口；当前服务端又只有一个版本字段，因此本次不表达 Profile 明细。

## 7. Firefox 实现契约

### 7.1 默认基目录

```text
$userHome/Library/Application Support/Firefox
```

配置文件：

```text
$basePath/profiles.ini
```

插件数据：

```text
{profilePath}/extensions.json
```

### 7.2 运行时 Profile 参数

按参数族优先级处理：

1. 显式路径族：`--profile`、`-profile`
2. 命名 Profile 族：`-P`、`--P`
3. `profiles.ini`
4. `Profiles/*` 排序扫描

参数名比较不区分大小写。支持 split 和等号形式，例如 `--profile /x`、`--profile=/x`、`-P work`、`-P=work`；不支持拼接形式 `-Pwork`。空值视为没有可用值。该大小写和 `--P` 支持是读取器的容错行为，不代表所有 Firefox 版本都把这些形式作为官方启动语法。

显式路径族整体优先于命名族，不取决于它们在 argv 中的相对顺序；每个族只检查第一个匹配 switch，不继续检查该族后面的重复项。结果真值表：

| argv（省略可执行文件） | 当前行为 |
|---|---|
| `-P work --profile /x` | `/x` 可读则 `runtime_path`；不可读再尝试 `work` |
| `--profile /x -P work` | 同上，显式族优先 |
| `--profile /missing -P work` | 显式路径不可读，`work` 可解析则 `runtime_name` |
| `-P missing --profile /x` | `/x` 可读则 `runtime_path`；显式族仍优先 |
| `--profile /x --profile /y` | 只使用第一个 `/x`；即使 `/x` 无效也不尝试 `/y` |
| `-P first -P second` | 只解析 `first`；无效时不尝试第二个命名 switch，进入配置回退 |
| `--profile --profile /y` | 第一个显式 switch 缺值，后一个显式 switch不再检查；随后尝试命名族或配置回退 |
| `-- --profile /x -P work` | 两个参数族都忽略，进入配置回退 |

显式路径支持：

- 绝对路径；
- `~`、`~/...`，使用传入的浏览器用户 home；
- 相对路径，以真实浏览器进程 cwd 为基准。

拒绝 `~otherUser/...`。如果相对路径但 cwd 缺失或不是绝对路径，则该参数无效。

### 7.3 显式/命名 Profile 的隔离语义

这是 Firefox 最关键的规则：

- 显式路径解析后是可读取目录：只检查该 Profile，并立即返回结果；插件不存在时也不能继续从其他 Profile 借一个版本。
- `-P` 能从 `profiles.ini` 解析到可读取目录：同样只检查该命名 Profile。
- 显式或命名 Profile 缺失、路径无效或不可读：才允许进入 `profiles.ini`/目录扫描回退。

这样可避免“当前 Firefox Profile 没装插件，但另一个 Profile 装了”时误报当前实例正常。

### 7.4 profiles.ini 解析

解析器至少要支持：

- UTF-8 文本和开头 BOM；
- 空行；
- `;`、`#` 注释；
- `[section]`；
- 首个 `=` 分隔的 `key=value`。

当前解析的确定语义：

- section、key 和 value 都去除首尾空白；只处理出现在有效 section 之后的键值。
- section 类型前缀 `Profile` / `Install` 比较不区分大小写；key 名使用精确大小写，例如只识别 `Name`、`Path`、`IsRelative`、`Default`。
- 同一 section 重复 key 时，后一个值覆盖前一个值。
- section 和候选保持文件出现顺序，不另外排序。
- `-P` 使用 `[Profile*].Name` 精确、区分大小写匹配；重复 Name 取文件中第一个匹配 section。
- 精确的 `[Install]` 不作为 install section；当前识别的是名称以 `Install` 开头且后面还有字符的 section，例如 `[InstallABC]`。
- `-Pname` 不支持；`-P=name` 和 split `-P name` 支持。

候选顺序：

1. 所有 `[Install*]` section 的 `Default`
2. 所有 `[Profile*]` 且 `Default=1` 的条目
3. 其余合法 `[Profile*]` 条目
4. `$basePath/Profiles/*` 中非隐藏目录，排序扫描

路径安全规则：

- `[Profile*]` 必须包含 `Path`，并且 `IsRelative` 只能严格为 `0` 或 `1`。
- `IsRelative=0` 时，`Path` 必须是绝对路径或 `~/...`。
- `IsRelative=1` 时，`Path` 必须是相对路径。
- `profiles.ini` 的相对路径标准化后必须仍位于 Firefox 基目录内，拒绝 `../` 穿越。
- 重复路径只检查一次。

`[Install*].Default` 不读取 `IsRelative`：绝对路径或 `~/...` 按显式路径标准化；其他值按 Firefox 基目录的相对路径处理，并做同样的词法 containment 检查。

这里的 containment 是 `stringByStandardizingPath` 后的词法检查，不解析符号链接，因此不是完整文件系统沙箱。如果目标环境把 `profiles.ini` 视为不可信输入，需要另行增加 realpath/symlink 校验及对应测试；本次移植先保持现有行为。

回退扫描中，损坏的 `extensions.json` 或不含目标插件的 Profile 应继续下一个候选。

伪代码：

```text
if argv has usable --profile/-profile:
    return read exactly that profile, even when extension is absent

if argv has -P name and profiles.ini resolves it to usable profile:
    return read exactly that profile, even when extension is absent

for profile in iniCandidates + sortedProfilesDirectoryScan:
    if profile has valid target extension metadata:
        return first successful info

return not found
```

### 7.5 extensions.json 读取

遍历：

```text
addons[]
```

找到：

```text
addon.id == extensionID
```

ID 使用精确、区分大小写的字符串比较。当前文件中第一个 ID 匹配的 addon 决定结果；如果该项字段无效，不再尝试同一文件中后续的重复 ID 项。

读取：

```text
addon.version
addon.installDate
```

返回：

```objc
@{
    @"version": version,
    @"init_time": @([installDate longLongValue] / 1000LL),
}
```

当前字段有效性为：`version` 必须是 JSON string，但允许空字符串；`installDate` 只需响应 `longLongValue`，因此 string/number 都可能被接受。转换直接做整数除以 `1000`，没有额外范围校验。回退扫描中字段无效会继续下一个 Profile；有效显式/命名 Profile 中字段无效仍保持隔离并返回 `extensionInfo=nil`。

### 7.6 Firefox 不带 Profile 参数时的能力边界

可以通过 `profiles.ini` 和目录扫描发现其他 Profile 中的插件，因此不会再固定只读默认 Profile。

但 Firefox 没有可直接等价于 Chromium `Local State.profile.last_used`、且能可靠绑定当前 PID 的统一事实来源。`Install*.Default` 和 `Profile*.Default=1` 只是配置优先级，不等于“这个 PID 当前正在使用的 Profile”。

因此：

- 带 `--profile` / `-profile` / `-P` 时可以精确绑定。
- 不带参数时可以按配置和扫描发现插件，但只能作为有序回退。
- 多个独立 Firefox 主进程同时使用不同 Profile、且 argv 又没有 Profile 参数时，本方案不能可靠区分每个 PID。

## 8. BrowserMonitor 最小接入方式

目标分支文件和方法名可能不同，应按“职责”定位，不要按本分支行号定位。

接入前必须确认输入前提：

| 产品内部值 | 传给读取器的规范值 | 主进程名参考 | 插件 ID plist key |
|---|---|---|---|
| Chrome channel | `chrome` | `Google Chrome` | `com.google.Chrome` |
| Edge channel | `edge` | `Microsoft Edge` | `com.microsoft.Edge` |
| Firefox channel | `firefox` | `firefox` | `org.mozilla.firefox` |

- PID 必须是目标浏览器主进程，不应是 renderer/helper/content process。
- 当前参考分支用 `Utility isProcessRunning:pid:` 取第一个同名可执行文件 PID；它不按 console user、父子关系或启动参数筛选。单主进程场景可用，但多个独立浏览器主进程或多登录用户场景无法保证选中期望实例。
- 前面的真实场景测试把场景测试器刚启动的精确 PID 直接传给 CLI，并没有覆盖 `Utility isProcessRunning:pid:`。因此新分支至少要增加“产品 PID 发现 -> reader”的集成检查；若需求包含多个独立主进程，应先定义选哪个实例或如何聚合，不能让“第一个 PID”成为隐含规则。
- `userHome` 应来自目标分支已有的 console user/home API。读取器目前不校验 PID owner 与 `userHome` 是否属于同一用户；在多用户环境中应避免把其他用户 PID 与当前 console user home 组合。
- 浏览器未运行时当前产品传 `pid=0`，读取器得到空 argv 并读取默认目录，以保留“安装但未运行仍可读取版本”的旧行为；CLI 为避免误用则拒绝 `pid<=0`。

### 8.1 必要改动

1. 引入共享读取器头文件。
2. 在健康检查已经取得浏览器 PID 的位置，把调用从：

```objc
[self getExtensionManifestWithBrowser:browser]
```

改为等价的：

```objc
[self getExtensionManifestWithBrowser:browser pid:pid]
```

3. manifest 方法继续复用目标分支现有插件 ID 缓存；缓存没有值时，再从现有 `readLocalEidInfo` 或等价入口读取 `BrowserPluginIDs`。
4. 使用目标 console user 的 home 调用共享读取器。
5. 只把 `result.extensionInfo` 返回给原健康逻辑。
6. 日志可以记录 `browser`、`pid`、`source`、Profile 末级目录名和版本，不记录完整 argv。

参考形态：

```objc
- (NSDictionary *)getExtensionManifestWithBrowser:(NSString *)browser pid:(int)pid {
    NSString *extensionID = [self existingExtensionIDForBrowser:browser];
    if (!extensionID.length) {
        return nil;
    }

    YSBrowserExtensionInfoResult *result =
        [YSBrowserExtensionInfoReader readBrowser:browser
                                              pid:pid
                                      extensionID:extensionID
                                         userHome:[Utility getUserHomeDictory]];
    return result.extensionInfo;
}
```

`existingExtensionIDForBrowser:` 是示意名。应适配目标分支已有映射，不要为了本需求重构整个浏览器 metadata 体系。

### 8.2 明确不得顺带修改

- `BrowserWebSocketServer` 和 heartInfo 更新。
- 5～10 秒 CLI 轮询不能替代产品内的 Socket 心跳。
- `healthInfoByBrowser:` 的状态分支。
- 窗口快照、重开宽限和睡眠宽限。
- 浏览器安装检测。
- 插件安装、卸载、managed policy、LNA 配置。
- Firefox 安装/卸载使用的 Profile helper；它与“运行 PID 的插件信息读取”职责不同。

如果目标分支已支持 Brave/Prisma，保留其原逻辑。本次不要把只支持 Chrome/Edge 的 Secure Preferences 规则强行套到特殊 Chromium 浏览器。

共享读取器会同步执行 sysctl 和磁盘 JSON/目录读取。当前产品每 60 秒在 BrowserMonitor 串行队列中依次检查三种浏览器；本机默认 Profile 实测每次 CLI 读取约 `0.01s`，当前没有性能阻断。仍应避免在持有 Socket/heartInfo 锁时调用读取器，并在 Profile 数量大或磁盘异常环境记录耗时；只有观测到明显延迟后再考虑独立 IO 队列或短期缓存，不能让缓存破坏运行 PID/Profile 的实时选择。

## 9. Xcode 工程接入

参考实现新增：

```text
YunshuManager/Sources/Monitor/EventAudit/Browser/YSBrowserExtensionInfoReader.h
YunshuManager/Sources/Monitor/EventAudit/Browser/YSBrowserExtensionInfoReader.m
```

在目标分支中需要：

1. 将 `.h`、`.m` 加入与 `BrowserMonitor` 同职责的 group。
2. 将 `.m` 加入实际编译 `BrowserMonitor` 的 target 的 Sources Build Phase。
3. 确认没有误加到测试 target、重复 target 或遗漏 Manager target。
4. 如果目标分支工程采用自动文件同步或其他生成机制，使用该分支惯例，不要照抄当前 `project.pbxproj` 的 UUID。

读取器使用：

```objc
#import <Foundation/Foundation.h>
#import <libproc.h>
#import <sys/sysctl.h>
```

独立编译需要 Foundation；`libproc` 和 `sysctl` 来自系统 SDK，不需要引入业务库。

## 10. 独立 CLI demo

### 10.1 目的

CLI 用于：

- 对真实浏览器 PID 验证 argv/cwd 获取；
- 手工观察所选 `source`、user data 和 Profile；
- 每 5～10 秒重复读取插件磁盘信息；
- 与 Socket 完全解耦。

CLI 不修改浏览器 Profile，也不安装插件。

当前桌面迁移包已包含可执行文件和自包含源码快照：

```text
/Users/jasonlee/Desktop/VOC-61365-多Profile插件信息读取迁移包
```

其中 `source/tools/browser-extension-info-demo/build.sh` 直接编译包内同一份 `source/YunshuManager/.../YSBrowserExtensionInfoReader.m`。

### 10.2 建议文件

```text
tools/browser-extension-info-demo/main.m
tools/browser-extension-info-demo/build.sh
tools/browser-extension-info-demo/README.md
tools/browser-extension-info-demo/.gitignore
```

`main.m` 必须直接链接生产共享读取器：

```shell
clang \
  -fobjc-arc \
  -fmodules \
  -Wall \
  -Wextra \
  -framework Foundation \
  -I YunshuManager/Sources/Monitor/EventAudit/Browser \
  tools/browser-extension-info-demo/main.m \
  YunshuManager/Sources/Monitor/EventAudit/Browser/YSBrowserExtensionInfoReader.m \
  -o tools/browser-extension-info-demo/build/browser-extension-info-demo
```

### 10.3 参数

```text
--browser chrome|edge|firefox   必填
--pid <pid>                     必填
--extension-id <id>             可选，覆盖 plist
--plugin-ids <path>             默认 /opt/.yunshu/config/BrowserPluginIDs
--user-home <path>              默认当前用户 home；测试其他用户时显式传入
--watch                         循环读取
--interval <seconds>            默认 5，最小 1
--count <number>                watch 模式读取次数，0 表示持续
--debug-argv                    显式输出完整 argv
--help
```

退出码：

| code | 含义 |
|---:|---|
| `0` | 正常完成；`found=false` 仍是一次成功读取 |
| `64` | 参数非法、浏览器不支持或输入不完整 |
| `66` | 找不到对应插件 ID |

### 10.4 使用示例

查浏览器 PID：

```shell
ps -ax -o pid=,command= | rg 'Google Chrome|Microsoft Edge|firefox'
```

读取一次：

```shell
./tools/browser-extension-info-demo/build/browser-extension-info-demo \
  --browser firefox \
  --pid 12345
```

每 10 秒读取，最多 3 次：

```shell
./tools/browser-extension-info-demo/build/browser-extension-info-demo \
  --browser chrome \
  --pid 12345 \
  --watch \
  --interval 10 \
  --count 3
```

每次输出一行排序 JSON，例如：

```json
{"browser":"firefox","checked_at":1784600000,"extension_id":"dlp_group@eaglecloud.com","extension_info":{"init_time":1700000000,"version":"3.1.4"},"found":true,"pid":12345,"profile_path":"/tmp/firefox-profile","source":"runtime_path","user_data_path":"/Users/test/Library/Application Support/Firefox"}
```

默认输出不得含 `process_arguments`；只有 `--debug-argv` 才能包含。

## 11. 真实浏览器场景测试器

建议保留独立场景测试器：

```text
tools/browser-extension-info-demo/live_scenarios.m
tools/browser-extension-info-demo/build-live-scenarios.sh
```

它验证的不是签名插件真实安装，而是：真实浏览器进程启动参数 -> PID argv/cwd -> Profile 选择 -> 共享读取器返回结果。测试元数据只能在隔离临时 Profile 中、浏览器成功启动后写入。

### 11.1 必须覆盖的 6 个真实启动场景

| # | 浏览器 | 启动/选择方式 | 期望 |
|---:|---|---|---|
| 1 | Chrome | `--user-data-dir=...` + `--profile-directory=Profile 1` | `runtime`，命中 Profile 1；默认 5 秒 watch 两次 |
| 2 | Chrome | 自定义 user data，多 Profile，无显式 Profile | `Local State.last_used` 命中 Profile 2 |
| 3 | Edge | 含空格的自定义路径 + Profile 2 | `runtime`，命中 Profile 2 |
| 4 | Firefox | 绝对 `--profile <path>` | `runtime_path` |
| 5 | Firefox | 相对 `-profile "relative profile"` | 用真实 cwd 解析，`runtime_path` |
| 6 | Firefox | `-P work` | 通过隔离 `profiles.ini` 解析，`runtime_name` |

Chromium 真实启动应使用：

```text
--user-data-dir=<path>
```

Chrome/Edge 当前会拒绝拆分的 `--user-data-dir <path>` 启动形式并以状态 13 退出；读取器仍防御性支持拆分 argv，该分支由确定性测试覆盖。

### 11.2 进程安全要求

- 每轮创建唯一 `NSTemporaryDirectory()/yunshu-browser-live-<UUID>`。
- 浏览器通过测试器自身的 `--process-group-exec` 子模式执行 `setpgid(0,0)` 后 `execv`。
- 只记录本轮启动并确认拥有的 process group。
- 清理时只向这些 group 的负 PGID 发送 `SIGTERM`，超时后再 `SIGKILL`。
- 每个场景使用 `@try/@finally` 停止自己启动的浏览器。
- 捕获 `SIGINT`、`SIGTERM` 和未捕获异常，清理进程及临时目录。
- 禁止使用 `pkill`、命令行 substring 扫描或杀掉用户已经运行的浏览器。
- 报告必须包含 `leftover_process_groups=[]` 和 `temporary_root_removed=true`。

运行：

```shell
./tools/browser-extension-info-demo/build.sh
./tools/browser-extension-info-demo/build-live-scenarios.sh
./tools/browser-extension-info-demo/build/browser-extension-info-live-scenarios \
  ./tools/browser-extension-info-demo/build/browser-extension-info-demo \
  /tmp/browser-extension-info-live-scenarios.json
```

## 12. 在差异较大分支上的落地顺序

不要直接按当前 diff 的行号套补丁。按下面职责逐步移植，每一步都可独立检查。

### 步骤 0：保护现场与记录基线

- 保留当前 worktree，直到新分支实现通过。
- 记录新分支名称、HEAD、工作区是否干净。
- 阅读新分支自己的 `AGENTS.md` / 构建说明。
- 先跑目标分支现有 build 和浏览器测试，记录基线；已有失败不能归因给本次修改。

### 步骤 1：定位目标分支职责点

通过语义搜索，而不是行号：

```shell
rg -n 'healthInfoByBrowser|getExtensionManifest|Secure Preferences|extensions.json|BrowserPluginIDs|heartInfo|BrowserWebSocketServer' .
```

确认：

- 谁拿到浏览器 PID；
- 谁解析插件版本；
- 谁提供 console user home；
- 谁提供插件 ID；
- 哪个 target 编译 BrowserMonitor；
- 目标分支是否已支持更多浏览器或新的 metadata 映射。

### 步骤 2：先加入独立共享读取器

- 按第 3～7 节行为契约实现 `.h/.m`。
- 不导入业务类。
- 先接确定性契约测试，确保选择算法独立通过。

### 步骤 3：最小接入 BrowserMonitor

- 只新增 `pid` 传递和读取器调用。
- 复用目标分支现有 ID/home 获取。
- 删除或停用的旧 helper 仅限“插件信息磁盘读取”重复代码。
- 不删除 Firefox 安装/卸载 helper，不动 Socket 和健康状态。

### 步骤 4：接入目标工程

- 按目标分支工程管理方式添加 target membership。
- 做完整 Debug build，解决编译集成问题后再继续。

### 步骤 5：加入 CLI 和确定性测试

- CLI 直接编译共享读取器。
- 先测默认读取和参数错误，再测真实 PID。

### 步骤 6：跑真实浏览器 6 场景

- 使用隔离 Profile 和受控 process group。
- 不触碰用户已有浏览器。
- 核对报告清理字段。

### 步骤 7：跑目标分支邻近回归

- 健康状态矩阵。
- 插件安装/卸载/LNA 流程。
- Socket/心跳状态逻辑。
- 完整 app build。

### 步骤 8：人工核对

- 使用真实手工安装插件的 Chrome/Edge/Firefox Profile 各至少一次。
- 对比浏览器 UI 显示版本、CLI 的 Profile/source/version 和产品上报。
- 结束测试时只关闭本次明确启动的进程。

## 13. 验证清单与命令

### 13.1 共享读取器契约测试

参考测试文件：

```text
docs/YunshuManager/tests/browser-plugin-profile-manifest.test.m
```

编译运行：

```shell
cd docs/YunshuManager/tests
clang -fobjc-arc -fmodules -framework Foundation \
  browser-plugin-profile-manifest.test.m \
  ../../../YunshuManager/Sources/Monitor/EventAudit/Browser/YSBrowserExtensionInfoReader.m \
  -o /tmp/voc-61365-browser-plugin-profile-manifest.test
/tmp/voc-61365-browser-plugin-profile-manifest.test
```

预期：

```text
=== 31 passed, 0 failed ===
```

31 个行为用例必须全部保留：

1. Chrome 次级 Profile 中存在插件。
2. Edge 次级 Profile 中存在插件。
3. 忽略 Chromium `System Profile`。
4. 损坏的 Secure Preferences 跳到下一候选。
5. 显式运行 Profile 胜出，不能选择最高版本。
6. 拆分 Chromium 参数保留含空格路径。
7. `Local State.last_used` 先于 `Default`。
8. `last_active_profiles` 跳过缺失 Profile。
9. 损坏 Local State 回退运行目录的 `Default`。
10. 运行目录扫描是 Chromium 最终候选。
11. 有效但空的 runtime 不跨到 home `Default`。
12. 缺失的 runtime 回退 home `Default`。
13. 忽略 `--` 之后的 Chromium switch。
14. runtime 枚举失败时回退 home `Default`。
15. 拒绝 Chromium Profile 目录穿越。
16. Edge 自定义 user data + 显式 Profile。
17. `~/` 使用传入的浏览器用户 home 展开。
18. Firefox 次级 Profile 中存在插件。
19. 损坏的 Firefox `extensions.json` 跳到下一回退候选。
20. Firefox 显式 `~/` Profile 胜出。
21. Firefox 重复 Profile switch 取第一个。
22. Firefox 相对 Profile 使用进程 cwd。
23. Firefox 命名 `-P` Profile 胜出。
24. 有效命名 Profile 未装插件时保持隔离。
25. Firefox `Install*.Default` 先于 `Profile*.Default=1`。
26. 有效但空的显式 Firefox Profile 保持隔离。
27. 缺失的显式 Firefox Profile 回退 `profiles.ini`。
28. 忽略 `--` 之后的 Firefox switch。
29. 损坏 `profiles.ini` 时回退目录扫描。
30. 忽略缺失/非法 `IsRelative`。
31. 拒绝 `profiles.ini` 相对路径穿越。

建议补充但当前参考测试尚未覆盖：

- 精确断言 Chromium 和 Firefox 的 `init_time` 转换值。
- 可注入模拟 `KERN_PROCARGS2`/`proc_pidinfo` 权限失败和 PID 退出竞态。

### 13.2 健康状态回归

```shell
cd docs/YunshuManager/tests
clang -fobjc-arc -framework Foundation -framework CoreServices \
  browser-plugin-health.test.m -o /tmp/voc-61365-browser-plugin-health.test
/tmp/voc-61365-browser-plugin-health.test
```

参考结果：

```text
35 passed, 0 failed
```

该测试的 addon/heart/window 状态主要通过参数注入，不代替 Profile IO 测试和真实 PID 测试。

### 13.3 安装/卸载/健康流程回归

```shell
cd docs/YunshuManager/tests
clang -fobjc-arc -framework Foundation \
  browser-plugin-flow.test.m -o /tmp/voc-61365-browser-plugin-flow.test
/tmp/voc-61365-browser-plugin-flow.test
```

参考结果：

```text
57 passed, 0 failed
```

如果差异分支的用例数已变化，应以该分支测试为准，但必须确认本需求没有删减其浏览器或状态覆盖。

### 13.4 完整工程 build

根据目标分支 scheme 调整，当前参考命令形态：

```shell
xcodebuild \
  -workspace Yunshu.xcworkspace \
  -scheme Yunshu \
  -configuration Debug \
  build
```

### 13.5 CLI 负向检查

- 不支持浏览器：退出 `64`。
- 缺失插件 ID plist 项：退出 `66`。
- `found=false`：CLI 自身退出 `0`，JSON 明确表示未找到。
- 默认输出中没有 `process_arguments`。
- `SIGINT`/`SIGTERM` 能停止 watch。
- 场景测试器被 `SIGINT` 时应退出 `130`，停止所属进程组并删除临时根目录。

可执行示例（在仓库根目录，先运行 CLI build）：

```shell
cli=./tools/browser-extension-info-demo/build/browser-extension-info-demo

# 非法浏览器；预期 exit 64
"$cli" --browser safari --pid $$

# 空 plist；预期 exit 66
plutil -create xml1 /tmp/voc-61365-empty-plugin-ids.plist
"$cli" --browser chrome --pid $$ \
  --plugin-ids /tmp/voc-61365-empty-plugin-ids.plist

# 对真实 Chrome 主 PID 使用一个必不存在的 ID；预期 exit 0 且 found=false
"$cli" --browser chrome --pid <Chrome主PID> \
  --extension-id voc-61365-definitely-missing

# 默认输出不得有 process_arguments
"$cli" --browser chrome --pid <Chrome主PID> | \
  /usr/bin/python3 -c 'import json,sys; assert "process_arguments" not in json.load(sys.stdin)'
```

这些命令中的非零退出码需由调用脚本显式保存并断言；若外层 shell 使用 `set -e`，应临时关闭或分别运行，避免第一个预期失败提前终止整组检查。

### 13.6 变更卫生

```shell
git diff --check
git status --short
```

确认没有生成的测试二进制、临时 JSON、Xcode 用户文件或无关格式化进入 diff。

## 14. 当前参考实现的验证证据

> 本节只证明 2026-07-21 当前参考 worktree，不可替代新分支重新执行后的验收结果。

截至 2026-07-21，当前参考 worktree 已得到：

- `Yunshu.xcworkspace` Debug build：通过。
- 共享生产读取器契约：31/31 通过。
- 浏览器健康状态矩阵：35/35 通过。
- 浏览器安装/卸载/LNA/健康流程：57/57 通过。
- 真实 Chrome/Edge/Firefox PID 场景：6/6 通过。
- CLI 非法浏览器退出 64、缺失 ID 退出 66：通过。
- 场景测试器 SIGINT 清理：退出 130，所属进程组已停止，临时目录已删除。
- `git diff --check`：通过。
- 独立代码审查：没有 P0/P1/P2/P3 问题。
- 用户已手工测试 CLI：通过。
- Firefox 手工安装插件后重开，PID `77078` 被识别为 `runtime_path`，插件 ID `dlp_group@eaglecloud.com`，版本 `3.1.4`。

真实隔离场景当时使用浏览器版本：Chrome `150.0.7871.129`、Edge `150.0.4078.83`、Firefox `152.0.5`。这些版本只记录证据，不应成为代码判断条件。

## 15. 完成验收条件

### 功能

- [ ] Chrome 自定义 `--user-data-dir` 能读取对应目录。
- [ ] Chrome/Edge 显式 `--profile-directory` 优先。
- [ ] Chromium 无显式 Profile 时按 Local State -> Default -> 扫描顺序。
- [ ] 有效 runtime 为空时不越界到默认用户目录。
- [ ] Edge 使用正确默认路径和插件 ID key。
- [ ] Firefox 绝对、`~/`、相对 `--profile`/`-profile` 均按规则解析。
- [ ] Firefox `-P` 能通过 `profiles.ini` 解析。
- [ ] 有效显式/命名 Firefox Profile 未装插件时保持隔离。
- [ ] Firefox 无参数时支持配置回退和目录扫描。
- [ ] 返回健康协议仍只有 `version`、`init_time`。

### 回归

- [ ] Socket/心跳代码未删除、未替换为轮询。
- [ ] 健康状态机 diff 无行为变化。
- [ ] 安装/卸载/策略逻辑无行为变化。
- [ ] 目标分支已有其他浏览器逻辑未受影响。
- [ ] 完整 Debug build 通过。
- [ ] 确定性测试、健康矩阵和流程测试通过。

### 真实场景与安全

- [ ] 6 个真实浏览器启动场景全部通过。
- [ ] 测试器只清理自己拥有的 process group。
- [ ] `leftover_process_groups=[]`。
- [ ] `temporary_root_removed=true`。
- [ ] 至少一次真实手工安装插件的 Profile 校验通过。
- [ ] 默认日志和 CLI 输出不泄露完整 argv。

## 16. 已知限制与剩余风险

1. Chromium `Local State.last_used` 不是强实时事实；只有显式 `--profile-directory` 才能精确指定优先 Profile。
2. 同一个 Chromium 主进程同时承载多个 Profile 窗口时，单个版本字段不能表达所有 Profile 状态。
3. Firefox 不带 Profile 参数时没有可靠的 PID 级 `last_used` 等价物；`profiles.ini` 和扫描只是回退。
4. 多个独立 Firefox 主进程、不带 Profile 参数且使用不同 Profile 的场景未解决。
5. 磁盘配置与运行中扩展实例可能有短暂竞态；长期最准确方案仍是让插件心跳携带自身 manifest version，但不属于本次范围。
6. 当前真实场景测试向隔离 Profile 写入受控元数据，不等价于浏览器成功安装和加载签名插件。
7. `KERN_PROCARGS2`、`proc_pidinfo` 的权限失败和 PID 消失竞态已有安全回退，但尚缺可注入的自动化故障测试。
8. 当前契约测试主要断言版本选择，尚未逐项精确断言 `init_time`。
9. 产品端 Socket heartbeat -> reader -> server 的完整 E2E 未覆盖；现有状态测试使用注入值。
10. 产品 PID 发现仍使用“第一个同名可执行文件”，未覆盖多个独立主进程、多登录用户和 PID owner/home 不一致。
11. 进程 argv/cwd 读取失败后回退默认目录只保证健康检查可继续，不保证仍绑定正确实例；自定义 runtime 可能因此短暂读到默认 Profile。
12. `localizedStandardCompare` 可能受当前 locale 影响；现有测试使用简单 ASCII/Profile 数字名称，未验证跨 locale 的非 ASCII 排序一致性。
13. Profile containment 目前是词法标准化检查，不解析 symlink。
14. 磁盘元数据存在只证明配置文件记录了插件，不能证明插件已启用、签名有效、已加载或正在发送心跳。

## 17. 常见误区

- 不要继续固定读取 `Default`。
- 不要把命令行拼成字符串后按空格拆分。
- 不要取扫描结果中的最高版本。
- 不要在有效自定义 user data 或有效显式 Firefox Profile 未命中时越界到其他 Profile。
- 不要把 `source/profile_path/user_data_path` 混入健康上报。
- 不要硬编码当前机器上的插件 ID。
- 不要使用 Manager/root 的 home 代替 console user home。
- 不要在常规日志中打印完整 argv。
- 不要用 CLI 的 5～10 秒轮询替换产品 Socket 状态检测。
- 不要为验证方便使用 `pkill` 或按命令行模糊杀浏览器。
- 不要假称隔离 Profile fixture 是签名插件安装测试。
- 不要直接复制当前 `project.pbxproj` UUID 到差异分支。
- 当前 `docs/YunshuManager/browser-plugin-health.md` 存在历史冲突标记；迁移时只提取本需求相关说明，不要整文件覆盖到新分支。

## 18. 回滚方案

如果新分支接入后出现回归，按相反顺序回滚：

1. 将 `BrowserMonitor` manifest 调用恢复为目标分支原实现。
2. 从实际 app target 的 Sources Build Phase 移除共享读取器 `.m`。
3. 删除仅为本需求新增的共享读取器、CLI 和测试文件。
4. 保留本文件和测试报告用于问题复盘。
5. 重新执行目标分支原有 build/测试，确认回到基线。

不要通过删除或修改 Socket、健康状态、安装策略来规避读取器问题；那会扩大回滚影响面。

## 19. 当前 worktree 的参考文件清单

以下路径是 2026-07-21 当前参考实现的位置。切到其他分支后它们可能不存在，因此本文件已完整描述必须保持的行为。

### 产品代码

```text
YunshuManager/Sources/Monitor/EventAudit/Browser/YSBrowserExtensionInfoReader.h
YunshuManager/Sources/Monitor/EventAudit/Browser/YSBrowserExtensionInfoReader.m
YunshuManager/Sources/Monitor/EventAudit/Browser/BrowserMonitor.m
Yunshu.xcodeproj/project.pbxproj
```

### 测试与说明

```text
docs/YunshuManager/tests/browser-plugin-profile-manifest.test.m
docs/YunshuManager/tests/browser-plugin-health.test.m
docs/YunshuManager/tests/browser-plugin-flow.test.m
docs/YunshuManager/browser-plugin-health.md
```

### CLI 与真实场景

```text
tools/browser-extension-info-demo/main.m
tools/browser-extension-info-demo/build.sh
tools/browser-extension-info-demo/README.md
tools/browser-extension-info-demo/.gitignore
tools/browser-extension-info-demo/live_scenarios.m
tools/browser-extension-info-demo/build-live-scenarios.sh
```

### 桌面交付物

```text
/Users/jasonlee/Desktop/browser-extension-info-demo
/Users/jasonlee/Desktop/browser-extension-info-live-scenarios.json
/Users/jasonlee/Desktop/browser-extension-info-scenario-report.md
/Users/jasonlee/Desktop/VOC-61365-多Profile插件信息读取跨分支实现指南.md
/Users/jasonlee/Desktop/VOC-61365-多Profile插件信息读取迁移包/
```

## 20. 新分支实现后的交付报告模板

```text
目标分支 / HEAD：

变更内容：
- 共享读取器：
- BrowserMonitor 接入点：
- 工程 target：
- CLI / 测试：

文件清单：
-

验证结果：
- Debug build：PASS / FAIL
- Profile 契约：x/x
- 健康矩阵：x/x
- 流程回归：x/x
- 真实浏览器：x/6
- 手工安装插件：PASS / 未验证
- git diff --check：PASS / FAIL

未验证项及原因：
-

剩余风险：
-

回滚点：
-
```
