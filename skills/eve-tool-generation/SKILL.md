---
name: eve-tool-generation
description: Use when creating, modifying, or disabling eve agent tools under agent/tools/, or when building or changing the shared file-sandbox infrastructure (lib/fs-sandbox.ts) — follow the naming, schema, safety, testing, and reporting rules in this spec.
---

# Agent Tool 生成规范（Tool Generation Spec）

## 1. 文件位置与注册

| 情况 | 位置 |
| --- | --- |
| 新建一个 Tool | `agent/tools/<tool-name>.ts` |
| 覆盖/禁用同名的 eve 内建工具 | `agent/tools/<tool-name>.ts`，内容为 `disableTool()` |
| 文件沙箱（随需要它的首个文件类 Tool 一起生成，此后复用） | `lib/fs-sandbox.ts`（见 §2.1） |

规则：

- **文件名即工具名，必须 snake_case 的 ASCII 小写**（可含下划线），因为文件名会成为模型调用的工具名。
- 一个文件只 `default export` 一个工具（或一个 `disableTool()` 哨兵）；不要创建子目录来放单个工具。
- 无自定义 Registry：eve 自动发现 `agent/tools/*.ts`，`default export` 即注册。新增工具 = 新增 `.ts` 文件；**不要**创建注册表、配置文件或集中导出列表。
- 若需按会话动态提供工具，才用 eve 的 `defineDynamic`（见 `node_modules/eve/docs/tools/overview.mdx`）；普通静态工具不要用。

---

## 2. 共享基础设施（一套实现，不复写）

文件类 Tool 共用的跨工具基建——**文件沙箱、路径解析、读前写读取戳**——有且只有一套实现：随需要它的首个文件类 Tool 一起产出（同一轮交付），此后一律复用，不另起第二套。共享能力放在 `lib/`，工具文件只 `import` 复用。需要改动共享能力时，必须在原文件上改，并在最终报告（§6）说明理由。

### 2.1 文件沙箱 `lib/fs-sandbox.ts`

位置：`lib/fs-sandbox.ts`（单文件，不引入新依赖）。

**生成顺序**：

1. 若项目尚无该文件：**先按下方“公开契约”生成沙箱，再生成引用它的文件类 Tool**；两者同一轮交付，最终报告写明沙箱生成情况（§6.3）。
2. 若已存在：直接复用该 `sandbox` 单例，不创建第二份。

**公开契约（必须生成并公开）**：

- 类型导出 `SandboxMode = "read-only" | "workspace-write" | "danger-full-access"`；
- 类 `FsSandboxController`，须提供：
  - `standingMode`（默认 `"workspace-write"`）与 `projectRoot`（`resolve(process.cwd())`，即本项目 Sandbox Root；相对路径均基于它解析）；
  - `schemaFields()`——返回 `{ sandbox_permissions, justification }` 两个 zod 字段，供展入 mutating 工具的 inputSchema；其中 `sandbox_permissions` 为 `"workspace-write" | "danger-full-access"` 的可空枚举；
  - `approval`——批准函数：带升级参数（`sandbox_permissions` 及配对 `justification`）→ `"user-approval"`（等待用户批准）；参数非法 → `{ type: "denied", reason }`；无升级参数 → `"not-applicable"`；
  - `resolveMode(sandboxPermissions, justification, effectiveMode?)`——校验参数配对并强制“严格更宽”，返回本次调用授予的模式，拒绝非加宽升级；
  - `confine(target, mode?)`——任何写操作前调用：`read-only` 拒绝写入；`workspace-write` 只允许项目目录内路径；`danger-full-access` 对本次调用放开围栏；
  - 路径安全：拒绝 `..` 越权、项目外绝对路径、指向项目外的 symlink——用 `realpathSync.native` 把目标解析为真实路径后做前缀比较即可（内部辅助函数不要求对外公开）；
  - 模型可见的拒绝标记 `[sandbox: file access denied under … mode]` 与同轮升级提示。
- 共享单例：`export const sandbox = new FsSandboxController();`

### 2.2 路径解析 `resolveHostPath`

语义固定：

1. 输入先 `trim()`；
2. `$HOME/` 前缀展开到宿主家目录；
3. 绝对路径原样放行；
4. 其余相对路径 `resolve(process.cwd(), …)`（基于项目根解析）。

### 2.3 读前写读取戳

- 键：`globalThis["eve.tools.file.read-stamps"]`，值 `Map<hostPath, { mtimeMs, size }>`；
- 读工具成功读取后记戳；mutating 工具写前校验：**未读过、或戳与当前 `stat` 不一致（文件被改过）则抛错**；写入成功后重新打戳。

### 2.4 文件类 mutating 工具接入沙箱（三步）

1. `inputSchema` 中展开 `...sandbox.schemaFields()`；
2. 设置 `approval: sandbox.approval`；
3. `execute` 中先 `const mode = sandbox.resolveMode(sandbox_permissions, justification)`，路径按 §2.2 `resolveHostPath` 解析后立刻 `sandbox.confine(path, mode)`，再执行写入。

> 只读文件工具：只做 `resolveHostPath`，可不调用 `confine`（必要时可对 `read-only` 模式调用 `confine` 加固）；写入类一律强制 `confine`。

---

## 3. Tool 定义

标准骨架（以 `<tool-name>` 替换实际工具名；本骨架即实现依据，不依赖、不改动任何既有实现）：

```ts
import { defineTool } from "eve/tools";
import { z } from "zod";
// 文件类 mutating Tool 额外：
import { sandbox } from "../../lib/fs-sandbox";

export default defineTool({
  description: "（面向模型的一句话描述：做什么、何时用、边界/默认值/截断行为）",
  inputSchema: z.object({
    // 每个字段用 .describe(...) 写清语义/默认值；可选项用 .optional()
  }),
  // 可选：结构化输出时提供，同时约束 execute 返回类型
  // outputSchema: z.object({ ... }),
  // 文件类 mutating Tool 必须接入沙箱（§2.4）：
  // approval: sandbox.approval,
  async execute(input, ctx) {
    // 1) （文件类）resolveMode → resolveHostPath → confine（§2.4）
    // 2) 执行真实业务逻辑
    // 3) 返回 JSON 可序列化的普通对象（禁止 Date/Map/Set/循环引用）
  },
  // 可选：只需向模型投影摘要时使用，通道仍能看到完整输出
  // toModelOutput(output) { return { type: "text", value: "…" }; },
});
```

要求：

- **命名**：文件名决定工具名，对象内不再写 `name`（§1）。
- **输入**：`inputSchema` 为 zod object（或任意 Standard Schema / 纯 JSON Schema）；每个字段 `.describe()` 一句，语义上必有默认值的用 `.optional()`。
- **输出**：`execute` 返回 JSON 可序列化的普通对象，且按工具类型返回 `content` 字段——
  - **内容视图型**（`read_file` / `write_file` / `glob` / `grep` 等“读取/展示内容”的工具）：`content` 为 `<path>…</path>\n<type>file</type>\n<content>…</content>` 信封，另附结构化字段（如 `path`、`truncated`）；
  - **操作确认型**（`edit` 等“只在成功时给一句确认”的工具）：`content` 为面向模型的一句话文本确认，不必套信封。
  - 需要时用 `outputSchema` 声明并约束返回类型；只需向模型投影摘要时用 `toModelOutput`，通道仍能看到完整输出。
- **执行**：`async execute(input, ctx)`，可直接 `await`（需要时可写成 async generator 流式 yield 阶段性结果）；`ctx` 提供 `abortSignal`、`callId`、`toolName`、`session`、`getSandbox()` 等，只用需要的字段。
- **错误**：`throw new Error("对模型可读的中文或英文说明")`，文件类用清晰措辞（如“File not found: …”、“not a regular file”、“contains NUL bytes … binary file”）；升级参数非法时交给 `sandbox.resolveMode` 抛错，不要自行处理。
- **只 `default export`**；无命名导出、无 import/export 副作用逻辑。
- **写前读**：文件 mutating 工具改已有文件前必须已读过该文件（§2.3）。
- **依赖与风格**：不引入新的运行时依赖（确实需要时在 `package.json` 登记并 bun 安装、更新锁文件）；2 空格、分号、双引号、合适处 `readonly`。
- **容量/字节常量**：字节类容量常量必须写成显式倍数表达式，保持单位人读可懂，**禁止**直接写乘积后的裸整数。例如 `50 * 1024`（50 KiB）、`1 * 1024 * 1024`（1 MiB），而不是 `51200`、`1048576`；更大容量按倍数继续叠写（如 `64 * 1024 * 1024` = 64 MiB），不要写 `67108864`。同类带单位常量同理用显式倍数：时间毫秒写 `30 * 60 * 1000`（30 分钟），而不是 `1800000`。

---

## 4. 安全要求

- **不得绕过沙箱**：文件 Tool 只能经 `sandbox` 与 `node:fs/promises` 白名单 API 访问文件系统；不得直接 import 其他底层实现、不得把路径交给内存态缓存绕过 `confine`。围栏由 `realpathSync.native` 规范路径比较保证，工具代码不得用字符串拼接、`endsWith` 等朴素前缀判断替代。
- **不得用 shell 绕过 Tool 职责**：文件 Tool 不得 `exec`/`spawn` shell 来读写文件，文件操作一律走文件 API，保持“工具只做声明的那一件事”。
- **写前读**：写/改已有文件必须通过读取戳校验（§2.3），防止覆盖模型未阅读的变更。
- **输出清理**：不回显敏感内容；不把 `process.env`、token、数据库凭据打进工具输出；返回超大内容前按项目惯例截断并告知。
- **批准门**：任何 `danger-full-access` 类宽权请求必须走 `approval: sandbox.approval` 等待用户批准，模型侧不得自行放行。

---

## 5. 测试与验证

1. **Agent 面构建/发现**：`bun run build:eve`（eve 编译 `agent/`，能发现语法/结构错误）。
2. **类型检查**：`bun run typecheck`——硬性门禁，**必须通过**。
3. **lint / test**：项目提供时执行 `bun run lint`、`bun run test` 并记录结果；不提供则记录“无”。
4. **生产构建**（需要时）：`bun run build`（Next 生产构建）。
5. **手动验证**：在聊天流里实测新工具，至少覆盖：
   - 正常场景：输入合法参数，返回预期结构；
   - 异常场景：坏参数、文件不存在、目标不是普通文件、超大/二进制内容，模型得到可读报错；
   - 安全场景：`../` 越权路径、项目外绝对路径、指向项目外的 symlink 必须被沙箱拒绝且升级/批准路径生效。
6. 结论记录进最终报告（§6）。

---

## 6. 最终报告

1. **创建/修改的文件**：路径与变更类型（如“新增 `agent/tools/<tool-name>.ts`”）。
2. **输入/输出参数**：工具名、每个输入字段（类型/可选/描述）、`execute` 返回的每个字段（类型）、是否提供 `outputSchema` / `toModelOutput`。
3. **Sandbox / 共享基础设施变化**：本次是否**随工具生成了 `lib/fs-sandbox.ts`**（§2.1 公开契约是否齐备），还是**复用先前生成**的副本；使用了哪些方法（`schemaFields` / `approval` / `resolveMode` / `confine`）；未改动实现时明确写“复用，无改动”。
4. **测试与验证结果**：`bun run build:eve`、`bun run typecheck`（及项目提供且已执行的 lint/test、按需 `bun run build`）各自的通过情况；手动场景（正常/异常/安全）逐项结果。
5. **偏离说明**：如有任何偏离本规范之处，逐条说明原因。

---

## 附：生成时可参考的官方/工程文档

- eve 官方约定：`node_modules/eve/docs/tools/overview.mdx`（`defineTool`、`disableTool`、`defineDynamic`）、`node_modules/eve/dist/src/public/definitions/tool.d.ts`（类型契约）
- 项目工程约定：`AGENTS.md`（命令、风格、提交规范）、`package.json`（脚本）
