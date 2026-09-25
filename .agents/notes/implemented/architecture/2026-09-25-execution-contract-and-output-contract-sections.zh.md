# Agent Note: 执行约定身份段与固定输出约定段

Status: implemented

[English](2026-09-25-execution-contract-and-output-contract-sections.md) | 中文

## Problem

组装后的提示词以一个描述性标签开头——`You are an AI agent powered by DeepSeek Harness.`——并且结尾没有任何固定响应格式的内容。这两项事实都归 `dsh-system-prompt` 所有，但二者都没有约束模型产出什么：开场白只是命名了产品，而 `dsh-web-app`、`dsh-headless`、`dsh-sdk-app` 和 `dsh-acp-app` 中的部署 persona 只是重述了角色（`You are a coding agent powered by the {{model}} model.`），既没有执行约定，也没有输出约定。

这种组合带来两个后果。

开场身份不携带任何行为约束，因此未提供自身 persona 的部署继承到的提示词中，唯一的第一方指令只是一个产品名。每个部署作者随后都要在 YAML 中重新推导缺失的行为，四个随包附带的 persona 也就漂移成了同一句角色的近似副本。

第一方提示词中没有任何内容固定响应外壳，因此"以产物开头"和"不要复述工具调用"只是由对话承载的期望，而非提示词中的声明。部署根本无法更改它们，即使有人撰写响应约定，也不存在可供其使用的段位。

## Decision

`dsh-system-prompt` 拥有两个固定第一方段，并新增一个具名位置。

**`HARNESS_IDENTITY`（−1000）处的 `harness:identity`** 承载执行约定而非产品标签：模型是确定性执行引擎，输出以产物开头，推理保持在内部，并禁止确认、重述请求与含糊其辞。权威文本是导出的 `HARNESS_IDENTITY_TEXT`，因此测试与兼容性部署引用同一个字符串，而不必各自重述。`Config.includeHarnessIdentity` 仍可省略该段；段名、顺序与归属均未改变。

**`OUTPUT_CONTRACT`（10150）处的 `harness:output-contract`** 是新增项。它固定允许的响应开头（`## <Artifact Name>`、`[EXECUTING]`、`[COMPLETE]`、`[BLOCKED:<reason>]`）以及被禁止的首个 token。`Config.includeOutputContract`（默认 `true`）可省略该段。该位置位于 `HARNESS_SOURCE`（10000）与 `WEB_SURFACE`（10100）之后、`DEPLOYMENT_PERSONA_SUFFIX`（10200）之前，因此自行撰写结尾约定的部署仍然最后发言。

四个随包附带的应用 bundle 以及 `standard`、`ptc`、`cordis` 预设现在携带执行模式 persona，而不再是角色句子。该 persona 写明 objective/decompose/execute/verify/adapt/deliver 循环与工作目录，并且这些 bundle 的 `personaSuffix` 变为空，因为工作目录已移入前缀，而不是被陈述两次。

`dsh-web-app` 新增第五个预设 `autonomous`，声明于 `presets/autonomous.patch.yml`，并注册进该 bundle 的 `files` 与 `dsh.bundle.patch` 列表。它把标准工具集与自主执行引擎 persona、响应启动后缀以及 `includeRuntimeContext: false` 组合在一起。

## Alternatives considered

**保留产品标签，把执行约定作为单独段加入。** 该标签消耗一次固定的每请求开销且不构成任何约束；第二个开场段会让第一方提示词出现两条彼此排序的身份陈述，而归属规则已经把身份分配给唯一一个段。

**把输出约定做成部署 persona 后缀。** 该约定是每个部署都继承的第一方响应义务，而不是部署撰写的结尾陈述。后缀还会与部署已经拥有的 persona 后缀段位冲突，并且 `complete: true` 的 persona 会静默丢弃它。

**给 `OUTPUT_CONTRACT` 一个数字顺序而不设具名段位。** 仓库内的贡献者通过 `getSectionOrder(name)` 解析位置；裸数字会让一个第一方位置既不可共享，也不出现在完整性门禁遍历的顺序表中。

**只改身份，不动随包 persona。** 这些 persona 的角色句子重述了身份段现在已经陈述的内容，且没有增加信息；保留它们会让同一约定的两条重叠陈述同时发布，而较弱的那条渲染在较强的那条之后。

**把 `autonomous` 预设做成只声明 persona。** 不挂载任何工具行的预设会产生工具注册表为空的 agent——加入预设是替换组合，而不是在部署默认值上追加。因此该预设携带标准工具集。

## Consequences

- 每个部署都无需自行撰写即继承行为约定与输出约定；两者都不要的部署设置 `includeHarnessIdentity: false` 与 `includeOutputContract: false`。
- 产品名不再出现在第一方提示词中。需要它的部署在自己的 persona 中陈述它，这也是陈述模型名的同一位置。
- 提示词因这两个约定而变长：每次请求大约多 25 行。`snapshots/` 下的录制快照夹具在其 `system-prompt.*.expected.md` 附属文件中携带新文本，会话夹具记录更长的 `system/message` 事件。
- 两个 headless compaction 场景（`compaction-output-reserve`、`compaction-summary-headroom`）回放的脚本是按更短提示词定尺寸的；更长的提示词改变了 compaction 触发时机，因此其录制脚本会请求第四次模型调用。它们需要用真实 API key 重新录制（`pnpm run test:snapshot:record`），在此之前以无 key 方式运行会失败。
- `dsh-sdk-minimal` 设置 `includeHarnessIdentity: false`、`includeOutputContract: false` 与 `includeRuntimeContext: false`，因此其提示词仍仅为所配置的 persona。想要输出约定但不要身份的部署只需设置 `includeHarnessIdentity: false`。
- 输出约定中的 `[BLOCKED:<reason>]` 一行写作 `for §1.4 conditions only`，引用的是撰写该文本时所依据的指令；除此之外该段是自包含的。

## Testing

- `packages/core/system-prompt/tests/system-prompt.spec.ts` 固定 `HARNESS_IDENTITY_TEXT` 与 `OUTPUT_CONTRACT_TEXT` 作为两个固定内置段、它们的段名与顺序位置、`includeHarnessIdentity: false` 与 `includeOutputContract: false` 的省略行为，以及顺序表完整性检查所遍历的 `SECTION_ORDER_NAMES` 列表。
- `packages/core/agent-loop/tests/loop.spec.ts`、`packages/preset/persona/tests/persona.spec.ts`、`packages/fs/tool-fs/tests/tools.spec.ts`、`packages/fs/tool-fs-search/tests/tools.spec.ts` 与 `packages/web/tool-web/tests/tool-web.spec.ts` 用导出的常量构造其渲染提示词预期值，因此这些内置段只有一个归属。
- `packages/boot/app-boot/tests/app-boot.spec.ts` 固定身份位于 persona 之前、源码段位于 SDK 段之后。
- `apps/web/tests/replay-round-trip.e2e.ts` 断言已结算 Web 会话中的身份标题与执行模式 persona 标题，并将结尾各段与 `snapshots/web/fresh-round-trip/web-context.expected.md` 比较。
- `snapshots/` 下的录制快照固定每个随包 profile 的渲染提示词以及 SDK 子提示词。
