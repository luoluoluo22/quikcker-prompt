# 生成 Quicker 动作 JSON 的 AI 提示词（普通模式 v2 - Roslyn）

## 目的 ✨
本文件提供一个可直接喂给 AI（如 ChatGPT）的提示词模板，用于生成能够被 Quicker 接收并初始化为动作的 JSON 对象，尤其针对“普通模式 v2 (Roslyn)”的用例。

---

## 输出格式要求（严格）📋
- 输出必须为一个 **单一 JSON 对象**，且只输出 JSON（不要额外说明或注释）。
- 所有字符串必须使用 **双引号**（"）。
- `Id`：随机 UUID（v4）字符串。
- `LastEditTimeUtc` / `CreateTimeUtc`：使用 UTC 的 ISO 8601 格式字符串，形如 `2024-05-20T12:00:00Z`（末尾带 Z）。
- `Data` 字段：必须是 **JSON 字符串**（即 Data 的值本身是一个被 JSON 编码/转义后的字符串），内部表示动作定义（含 Steps、Variables 等）。
- `Data` 内部的每一行字符串（尤其 C# 代码）都要正确转义（双引号、反斜杠和换行需转义为 `\"`、`\\`、`\r\n` 或 `\n` 等）。
- `ActionType`：数值，组合动作为 `24`。

---

## 字段说明 🔧
- Row/Col：可选布局相关数值，示例中使用 `0`。 
- Title：动作标题（短文本）。
- Description：动作描述（可包含用途或注意事项）。
- Icon：FontAwesome 图标，使用 `fa:命名` 形式（例如 `fa:Solid_Code`）。
- Data：字符串形式的 JSON，需包含：
  - `LimitSingleInstance`（bool）
  - `Steps`（数组），每项包含 `StepRunnerKey`（示例：`sys:csscript`）和 `InputParams`（脚本相关参数，例如 `mode: normal_roslyn`, `script`, `runOnUiThread` 等）
  - `Variables`（数组，动作变量）
- CreateTimeUtc / LastEditTimeUtc：ISO 8601 UTC 时间字符串。

---

## 转义规则示例（最关键）🔐
假设有一段 C# 代码：

```csharp
public static object Exec(Quicker.Public.IStepContext context)
{
    context.SetVariable("Greeting", "Hello " + context.GetVariable("Name"));
    return context.GetVariable("Greeting");
}
```

把它嵌入到 `Data` 内部的 `script` 字段时，需要先把该代码作为 JSON 字符串转义：
- 双引号 " => \" 
- 反斜杠 \\ => \\\\（仅当存在反斜杠时）
- 换行 => `\r\n` 或 `\n`（推荐使用 `\r\n`）

转义后（示例片段）：

```
"public static object Exec(Quicker.Public.IStepContext context)\r\n{\r\n    context.SetVariable(\"Greeting\", \"Hello \" + context.GetVariable(\"Name\"));\r\n    return context.GetVariable(\"Greeting\");\r\n}"
```

---

## 可选执行线程值（runOnUiThread 对应说明）🧵
- `auto`：自动选择（Quicker 根据规则判断需要使用哪个线程）。
- `ui`：在 UI 线程执行（主界面线程，慎用长耗时操作）。
- `background`：后台线程（MTA），推荐用于非 UI/COM 操作。
- `sta`：共享 STA 线程（适合短时的 COM/剪贴板操作）。
- `staLongRun`：独立 STA 线程（适合需要长时间等待且需 STA 的任务）。

---

## 推荐 AI 提示词（模板）✍️
下面的提示词可直接发给通用 LLM：

> 你需要生成一个单独的 JSON 对象，用于在 Quicker 中初始化一个“组合动作”。请严格遵守下面的约束：
> 1. 仅输出一个 **有效的 JSON 对象**，不要添加任何额外的文本或解释。  
> 2. `ActionType` 必须为数字 24（代表组合动作）。  
> 3. `Id` 请使用随机 UUID（v4），格式如 `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`。  
> 4. `CreateTimeUtc` 和 `LastEditTimeUtc` 必须使用 UTC 的 ISO 8601 格式并带 `Z`。  
> 5. `Data` 字段必须是一个字符串类型，内部是一个 **符合 JSON 语法的字符串**（即 Data 的值本身是把内部 JSON 字符串化/转义后的结果）。  
> 6. 内部的 `Data` JSON 必须包含 `LimitSingleInstance`、`Steps`（非空数组）、`Variables`（可以为空数组）等字段。  
> 7. 在 `Steps` 中，使用 `sys:csscript` 作为 `StepRunnerKey` 并在 `InputParams` 中设置 `mode` 为 `normal_roslyn`，`script` 为经过正确 JSON 转义的 C# 源代码字符串，`runOnUiThread` 使用上面的线程值之一。  
> 8. C# 源代码字符串中所有双引号、反斜杠和换行必须被正确转义（示例使用 `\r\n` 表示换行）。  
> 9. 输出 JSON 中不得含注释或非 JSON 内容。  

> 请根据下列输入生成一个符合上述约束的 JSON：
> - Title: "{{Title}}"
> - Description: "{{Description}}"
> - Icon: "{{Icon}}"（例如 `fa:Solid_Code`）
> - 脚本语言：C#，请把下面给定的脚本作为 `script` 内容正确转义并放入 `Data` 内：
>
> ```csharp
> {{CSharpCode}}
> ```
>
> - 执行线程：`{{RunOnUiThread}}`（例如 `auto`, `ui`, `background`, `sta`, `staLongRun`）
> - LimitSingleInstance：`{{LimitSingleInstance}}`（true/false）
>
> 请输出最终的 JSON 对象。

---

## 示例输入与输出 🧾
输入示例（给 AI 的参数替换）：
- Title: "示例：问候变量"
- Description: "读取 Name 变量并写入 Greeting"
- Icon: "fa:Solid_Code"
- RunOnUiThread: "ui"
- LimitSingleInstance: true
- CSharpCode:

```csharp
using Quicker.Public;
public static object Exec(IStepContext context)
{
    var name = context.GetVariable("Name");
    context.SetVariable("Greeting", "Hello " + name);
    return context.GetVariable("Greeting");
}
```

AI 生成的（部分）JSON 输出应类似：

```json
{
  "Row": 0,
  "Col": 0,
  "ActionType": 24,
  "Title": "示例：问候变量",
  "Description": "读取 Name 变量并写入 Greeting",
  "Icon": "fa:Solid_Code",
  "Data": "{\"LimitSingleInstance\":true,\"Steps\":[{\"StepRunnerKey\":\"sys:csscript\",\"InputParams\":{\"mode\":{\"Value\":\"normal_roslyn\"},\"script\":{\"Value\":\"using Quicker.Public;\\r\\npublic static object Exec(IStepContext context)\\r\\n{\\r\\n    var name = context.GetVariable(\\\"Name\\\");\\r\\n    context.SetVariable(\\\"Greeting\\\", \\\"Hello \\\" + name);\\r\\n    return context.GetVariable(\\\"Greeting\\\");\\r\\n}\\r\\n\"},\"runOnUiThread\":{\"Value\":\"ui\"}}}],\"Variables\":[]}",
  "Id": "a1b2c3d4-...",
  "LastEditTimeUtc": "2024-05-20T12:00:00Z",
  "CreateTimeUtc": "2024-05-20T12:00:00Z"
}
```

> 注意：上例中 `Data` 的值是一个更长的字符串，内部 JSON 与 C# 代码均已转义为单一 JSON 字符串。


文档创建于仓库 `quikcker-prompt` 的 `docs` 目录下，文件名：`提示词-生成-Quicker-动作JSON.md`。
