# 普通模式 v2 (Roslyn) 说明文档 ✅

## 概述 🔧
- **执行引擎**：使用 **Roslyn** 编译并运行 C# 脚本，支持较新版本的 C# 语法特性。
- **性能**：首次（冷启动）编译耗时；后续使用已缓存的程序集以提速。
- **执行位置**：代码在 Quicker 主进程中执行，可通过 `IStepContext` 访问动作变量与上下文。

---

## 脚本结构与接口要求 📄
- 脚本**必须包含**一个静态方法 `Exec`，参数类型为 `IStepContext`，可以带返回值或不带返回值。示例：

```csharp
using Quicker.Public;

public static class Script
{
    // 无返回值
    public static void Exec(IStepContext context)
    {
        // 使用 context 读写动作变量、执行操作等
        context.SetVariable("result", "ok");
    }

    // 或者 有返回值
    public static object Exec(IStepContext context)
    {
        // ...
        return "结果";
    }
}
```

- **注意**：普通模式 v2 的 `Exec(IStepContext)` 与低权限模式的 Exec 声明不同，**不可混用**。

---

## 模块参数说明 ⚙️
- **脚本内容**：要运行的 C# 源代码（含 `Exec(IStepContext)`）。
- **引用 DLL（Reference）**：每行一个路径；若已加入 GAC 可直接写 DLL 名称，否则写完整路径。
- **允许缓存程序集**：是否缓存编译后的程序集（建议开启以提升重复运行速度）。
  - 缓存目录：Windows 临时目录。
  - 程序升级会清空缓存。
- **执行线程（线程模型）**：
  - 自动：由 Quicker 自动选择
  - UI 线程：主界面线程（请避免长耗时操作）
  - 后台线程（MTA）：无 UI/COM 场景优先使用
  - 后台线程（STA）：共享 STA 线程，用于短时 COM/剪贴板操作
  - 后台线程（STA 独立线程）：为长期需 STA 的任务创建独立线程
- **失败后停止**：脚本抛异常时是否停止当前动作（布尔选项）。
- **等待返回**：是否等待 `Exec` 执行完并取回返回值（若不等待，“返回内容”为空）。

---

## 推荐实践与注意事项 ⚠️
- 避免在运行时通过文本插值频繁生成不同脚本内容（每种不同内容会生成新的程序集，降低缓存复用率）。
- 优先开启程序集缓存并保持脚本内容稳定以减少冷启动次数。
- 遇到 Office/COM 互操作异常时，尝试使用 STA 线程或低权限模式 v2，确认线程模型是否匹配。
- 从网页复制代码可能带有不可见字符，若遇奇怪的编译错误请先清理不可见字符。

---

## 常见故障排查建议 🛠️
- **编译错误**：检查不可见字符、缺失引用或语法不兼容（确认 Roslyn 支持的语法）。
- **COM/Office 运行时错误**：尝试切换到 STA（共享或独立）线程并测试；必要时使用低权限模式 v2。
- **性能慢**：若为冷启动，建议启用缓存并减少脚本内容变化。

---

## 示例（读取/写入动作变量）

```csharp
using Quicker.Public;

public static class Example
{
    public static object Exec(IStepContext context)
    {
        // 读取变量
        var name = context.GetVariable("Name");

        // 写入变量
        context.SetVariable("Greeting", "Hello " + name);

        return context.GetVariable("Greeting");
    }
}
```

---

文档由 `quikcker-prompt` 仓库自动生成，若需我：
1. 将此文件加入项目 `README.md` 的链接位置（创建或修改），或
2. 扩展为包含更多 `IStepContext` API 示例（需要具体 API 列表），
请告诉我你的偏好。💡
