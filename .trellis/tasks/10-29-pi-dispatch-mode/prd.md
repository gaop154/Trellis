# Pi 平台 dispatch_mode（inline 模式支持）

## Goal

为 Pi 平台镜像 Codex 已有的 `dispatch_mode` 机制：新增 `pi.dispatch_mode` 配置（**默认 inline**），
Pi 主会话直做实现/检查/研究（`trellis-before-dev` → 编辑 → `trellis-check` skill 流程），
不再派 `trellis-implement` / `trellis-check` / `trellis-research` 子代理；
改完打 npm tarball 并同步到各业务项目。

## 背景与动机

- 用户（gp）实际痛点：Pi 上 Trellis 的 implement/check 子代理冷启动慢、上下文注入开销大，
  而事后编译+自检才是主要时间成本，子代理环节纯增延迟。
- 用户仍需要：任务生命周期（task.py create/start/archive）、PRD/design/implement 规划件、
  spec 沉淀（trellis-update-spec）、journal——即「任务功能 + 沉淀能力」。
- 上游已为 Codex 解决过同一问题（`codex.dispatch_mode`，0.5.9 起 Codex 默认 inline），
  机制完全同构，Pi 只是没有接线。

## Requirements

1. **配置**：`.trellis/config.yaml` 新增 `pi.dispatch_mode`，取值 `inline` / `auto`
   （`sub-agent` 为 `auto` 的兼容别名，非法值回退 `inline`）。**默认 `inline`**（本 fork 的决策；
   上游语义为 auto，分叉点记录在 design.md）。
2. **inline 行为**（默认）：
   - 每回合 `<workflow-state>` 面包屑取 `planning-inline` / `in_progress-inline` 变体
     （workflow.md 中已存在这些块，Codex inline 复用同一套）。
   - `get_context.py --mode phase --platform pi` 的步骤详情按 `pi-inline` 平台过滤，
     命中 inline 路由块（trellis-before-dev → 编辑 → trellis-check skill）。
   - `task.py` 的 jsonl 编排门槛不再对 Pi 生效（`_has_subagent_platform` 对 `.pi` 目录
     按 dispatch_mode 判断，镜像 `.codex` 的处理）。
3. **auto 行为**（显式配置时）：与现状完全一致——派发子代理、jsonl 注入、
   `pi-sub-agent` marker 块，可回退。
4. **分发**：`pnpm build` + `npm pack` 出 tarball；各业务项目安装后跑 `trellis update`
   刷新 workflow.md / scripts / .pi 扩展（hash 跟踪整文件替换）。
5. **不影响其他平台**：Codex / Claude / Cursor 等平台行为零变化。

## Acceptance Criteria

- [ ] `pi.dispatch_mode: inline`（默认）时：`resolve_effective_platform("pi", config)` 返回 `pi-inline`；
      `auto`/`sub-agent` 时返回 `pi-sub-agent`。
- [ ] workflow.md 模板：6 个派发 marker 块中 `Pi` → `pi-sub-agent`；5 个 inline marker 块加入 `pi-inline`；
      回归测试断言 pi（默认与 auto 两种取值）步骤路由非空。
- [ ] Pi 扩展模板：`workflowBreadcrumb` 在 inline 模式优先取 `-inline` 面包屑；
      config.yaml 缺失/损坏时安全回退 plain 状态（不 warning 刷屏）。
- [ ] `_has_subagent_platform`：`.pi` 存在且 `pi.dispatch_mode=inline` 时返回 False（不要求 jsonl）。
- [ ] 镜像 codex 的回归测试用例新增 pi 用例，`pnpm test`（vitest）全绿。
- [ ] `pnpm build` 成功；`npm pack` 产出 tarball。
- [ ] 本仓库（Trellis 源码仓自身 dogfood）的 `.trellis/workflow.md`、`.pi/extensions/trellis/index.ts`
      与新模板一致。
- [ ] 试点项目（xizhao-design-project）安装 tarball 并 `trellis update` 后：
      `get_context.py --mode phase --platform pi` 输出 inline 路由块；
      每回合面包屑为 in_progress-inline 内容。
- [ ] spec 更新：`workflow-state-contract.md` Runtime contract 第 6 条及平台矩阵反映 Pi dispatch_mode。

## 非目标（Out of Scope）

- 不改 Codex / 其他平台的 dispatch 行为。
- 不移除 `trellis_subagent` 工具注册（保留显式调用能力，流程文本不再指向它）。
- 不做 `pi.dispatch_mode` 的动态热切换（改配置需重启会话，与 Pi 其他配置一致）。
- 不发布到公共 npm registry（本地 tarball 分发）。

## Notes

- 用户明确要求「全局改」：改动落在源码仓库模板层，通过 npm 包 + `trellis update` 链路分发。
- 旧项目 workflow.md marker 仍是 `Pi`：升级 CLI 后必须跑 `trellis update`，
  否则 pi 平台步骤详情路由为空（面包屑因 `-inline` 块早已存在而仍可用）。
