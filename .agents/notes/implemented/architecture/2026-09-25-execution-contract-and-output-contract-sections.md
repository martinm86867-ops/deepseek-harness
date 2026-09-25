# Agent Note: Execution-contract identity and a fixed output contract

Status: implemented

English | [中文](2026-09-25-execution-contract-and-output-contract-sections.zh.md)

## Problem

The assembled prompt opened with a descriptive label — `You are an AI agent powered by DeepSeek Harness.` — and closed with nothing that fixed the response format. Both facts were owned by `dsh-system-prompt`, but neither constrained what the model produced: the opener named the product, and the deployment personas across `dsh-web-app`, `dsh-headless`, `dsh-sdk-app`, and `dsh-acp-app` restated a role (`You are a coding agent powered by the {{model}} model.`) without an execution or output contract.

Two consequences followed from that composition.

The opening identity carried no behavioral constraint, so a deployment that supplied no persona of its own inherited a prompt whose only first-party instruction was a product name. Every deployment author then re-derived the missing behavior in YAML, and the four shipped personas drifted into near-duplicates of the same role sentence.

Nothing in the first-party prompt fixed the response envelope, so "begin with the artifact" and "do not narrate the tool call" were expectations carried by conversation rather than stated in the prompt. A deployment could not change them at all, and no section slot existed for a response contract even if one were authored.

## Decision

`dsh-system-prompt` owns two fixed first-party sections with a new named placement.

**`harness:identity` at `HARNESS_IDENTITY` (−1000)** carries an execution contract rather than a product label: the model is a deterministic execution engine, output begins with the artifact, reasoning stays internal, and acknowledgment, request restatement, and hedging are prohibited. The authoritative text is the exported `HARNESS_IDENTITY_TEXT`, so tests and compatibility deployments reference one string instead of restating it. `Config.includeHarnessIdentity` still omits it; the section name, its order, and its ownership are unchanged.

**`harness:output-contract` at `OUTPUT_CONTRACT` (10150)** is new. It fixes the permitted response openings (`## <Artifact Name>`, `[EXECUTING]`, `[COMPLETE]`, `[BLOCKED:<reason>]`) and the prohibited first tokens. `Config.includeOutputContract` (default `true`) omits it. The placement sits after `HARNESS_SOURCE` (10000) and `WEB_SURFACE` (10100) and before `DEPLOYMENT_PERSONA_SUFFIX` (10200), so a deployment that authors its own closing contract still speaks last.

The four shipped application bundles and the `standard`, `ptc`, and `cordis` presets now carry an execution-mode persona instead of the role sentence. The persona names the objective/decompose/execute/verify/adapt/deliver loop and the working directory, and the bundles' `personaSuffix` becomes empty because the working directory moved into the prefix rather than being stated twice.

`dsh-web-app` gains a fifth preset, `autonomous`, declared in `presets/autonomous.patch.yml` and registered in the bundle's `files` and `dsh.bundle.patch` lists. It composes the standard tool set with an autonomous execution-engine persona, a response-initiation suffix, and `includeRuntimeContext: false`.

## Alternatives considered

**Keep the product label and add the execution contract as a separate section.** The label costs a fixed per-request line and constrains nothing; a second opener would have made the first-party prompt two identity statements ordered against each other, and the ownership rule already assigns identity to one section.

**Make the output contract a deployment persona suffix.** The contract is a first-party response obligation that every deployment inherits, not a deployment-authored closing statement. A suffix would also collide with the persona-suffix slot that deployments already own, and `complete: true` personas would silently drop it.

**Give `OUTPUT_CONTRACT` a numeric order without a named slot.** Repository contributors resolve placements through `getSectionOrder(name)`; a bare number would have made a first-party placement unshareable and unlisted in the order table that the completeness gates walk.

**Leave the shipped personas alone and change only the identity.** The personas' role sentence restates what the identity now states and adds nothing; leaving them would have shipped two overlapping statements of the same contract, with the weaker one rendering after the stronger.

**Add the `autonomous` preset as a persona-only declaration.** A preset that mounts no tool rows produces an agent with an empty tool registry — joining a preset replaces the composition rather than adding to the deployment default. The preset therefore carries the standard tool set.

## Consequences

- Every deployment inherits a behavioral and an output contract without authoring either, and a deployment that wants neither sets `includeHarnessIdentity: false` and `includeOutputContract: false`.
- The product name no longer appears in the first-party prompt. A deployment that needs it states it in its persona, which is the same place the model name is stated.
- The prompt is longer by the two contracts: roughly 25 lines on every request. Recorded snapshot fixtures under `snapshots/` carry the new text in their `system-prompt.*.expected.md` sidecars, and the session fixtures record the longer `system/message` event.
- Two headless compaction scenarios (`compaction-output-reserve`, `compaction-summary-headroom`) replay scripts sized against the shorter prompt; the longer prompt changes when compaction fires, so their recorded scripts request a fourth model call. They require re-recording with a live API key (`pnpm run test:snapshot:record`) and fail keylessly until then.
- `dsh-sdk-minimal` sets `includeHarnessIdentity: false`, `includeOutputContract: false`, and `includeRuntimeContext: false`, so its prompt remains its configured persona alone. A deployment that wants the output contract with no identity sets only `includeHarnessIdentity: false`.
- The output contract's `[BLOCKED:<reason>]` line reads `for §1.4 conditions only`, a reference to the directive under which the text was authored; the section is otherwise self-contained.

## Testing

- `packages/core/system-prompt/tests/system-prompt.spec.ts` pins `HARNESS_IDENTITY_TEXT` and `OUTPUT_CONTRACT_TEXT` as the two fixed built-ins, their section names and order positions, the `includeHarnessIdentity: false` and `includeOutputContract: false` omissions, and the `SECTION_ORDER_NAMES` list that the order-table completeness check walks.
- `packages/core/agent-loop/tests/loop.spec.ts`, `packages/preset/persona/tests/persona.spec.ts`, `packages/fs/tool-fs/tests/tools.spec.ts`, `packages/fs/tool-fs-search/tests/tools.spec.ts`, and `packages/web/tool-web/tests/tool-web.spec.ts` build their rendered-prompt oracles from the exported constants, so the built-ins have one owner.
- `packages/boot/app-boot/tests/app-boot.spec.ts` pins the identity before the persona and the source section after the SDK section.
- `apps/web/tests/replay-round-trip.e2e.ts` asserts the identity heading and the execution-mode persona heading in a settled Web session, and compares the closing sections against `snapshots/web/fresh-round-trip/web-context.expected.md`.
- The recorded snapshots under `snapshots/` pin the rendered prompt of every shipped profile and the SDK child prompts.
