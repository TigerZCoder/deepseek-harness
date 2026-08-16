# Agent Note: 跳过输入 schema 超出 harness JSON Schema 子集的 MCP 工具

Status: implemented

[English](2026-08-16-mcp-client-input-schema-containment.md) | 中文

## Problem

[MCP 客户端](../feature/2026-07-07-mcp-client-plugin.md)会把每个工具的 `inputSchema` 原样传入已注册的 `ToolDefinition.parameters`。强制执行的 JSON Schema 子集（`assertSupportedJsonSchema`）只作用于工具的**输出** schema——既在注册时，也通过桥接的 `supportedOutputSchema` 兜底——却从不作用于**输入** schema。

DeepSeek provider 在每次请求中逐字发送 `parameters`，当任一函数 schema 非法时，API 会拒绝整条请求。因此一个畸形工具会破坏该服务器**所有工具**的每一轮运行，而不仅是畸形的那一个。观察到的案例是 OpenAPI→MCP 网关为 multipart 文件参数生成了 `"type": "MultipartFile"`，另有若干 `enum` 关键字出现在 `type: "array"` 节点上；这三类都不在子集内。

## Decision

在 `syncTools`（`packages/mcp/mcp-client/src/tools.ts`）的 fetch 阶段，构建每个工具定义之前都会用 `assertSupportedJsonSchema` 检查其 `inputSchema`。schema 超出子集的工具会被跳过，并用 `ctx.logger.warn` 记下工具名；该服务器其余工具照常注册。检查发生在 fetch 阶段、swap 之前，因此被跳过的工具不会进入注册循环，而一次去掉某工具的重同步只是普通的世代替换。

这是 `supportedOutputSchema` 的输入侧镜像，区别在于「跳过」而非「降级」：输出 schema 回退到不受约束的 `JsonValue` 不会丢失调用方依赖的任何东西，但输入 schema **本身就是**模型用来构造调用的参数契约，所以不存在无损的「不受约束」替代——唯一不重写 schema 的容错就是不注册该工具。

不新增任何配置。输出路径没有开关（它总是容错），也没有消费方要求过严格的「让服务器失败」模式；修复前的行为——注册畸形工具并让每一轮都失败——是缺陷，而不是值得用默认值保留的模式。

注册循环保持其全或无回滚。schema 非法的工具已在 fetch 阶段被过滤，因此仍能到达 `ctx.tools.register` 的失败只剩真正的名称冲突——外部注册抢占 `mcp__<serverName>__` 命名空间——它仍会大声回滚，正与原始插件记录的要求一致。

## Alternatives considered

**把输入 schema 降级为空对象并保留该工具。** 否决：与输出不同，输出回退到不受约束的 `JsonValue` 不会丢失调用方依赖的东西，而输入 schema 是模型用来构造调用的参数形状。把它替换成空对象会隐藏所有参数，产生服务器只能逐条拒绝的调用；跳过是诚实的容错，因为模型永远不会看到一个它无法正确调用的工具。

**新增 `invalidToolSchema: "skip" | "fail"` 配置项。** 否决为无依据的配置化：输出路径没有这种开关且总是容错，也没有消费方要求严格失败。修复前的「fail」是缺陷，不是值得保留的模式；严格模式可以在有证据后再加。

**给每个 `ctx.tools.register` 单独包 try/catch，失败就跳过。** 否决：schema 非法的工具已在 fetch 阶段被过滤，而注册循环在冲突时的回滚归零是有意的不变量（绝不保留部分集合）。让冲突跳过工具会在外部抢占下静默丢弃一次注册，而不是把它暴露出来。

**清洗非法 schema，例如把 `"MultipartFile"` 替换成 `string`。** 否决：桥接不拥有 MCP schema 语义，也无法猜测正确的类型。这正是原始插件记录已经划定的启发式／清洗界限。

## Testing

`packages/mcp/mcp-client/tests/mcp-client.spec.ts` 固定了跳过行为：服务器列出含嵌套 `"MultipartFile"` 类型的一个工具和一个合法工具时，只注册合法的那个、不注册畸形名称并记录 warn。现有用例已经固定无参数的对象根 schema（`{"type":"object"}` 与 `{"type":"object","properties":{}}`）会注册，因此该检查不可能退化成丢弃无参工具。

## Consequences

一个畸形输入 schema 不再破坏整个服务器或任何一轮运行；它只损失该工具，并通过 warn 呈现。可用工具集现在取决于 schema 合法性——这是服务器自身工具列表的纯函数——而非加载顺序或跨服务器冲突，因此原始「绝不静默跳过／绝不依赖加载顺序」不变量保持其边界：这次跳过是响亮且确定性的。桥接仍然从不重写 schema；它只是拒绝注册一个它无法表达的 schema。
