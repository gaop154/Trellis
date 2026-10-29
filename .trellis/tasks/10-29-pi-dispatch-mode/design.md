# 技术设计：pi.dispatch_mode

## 总体思路

完全镜像 Codex 的 `codex.dispatch_mode` 机制（`inject-workflow-state.py` 的
`resolve_breadcrumb_key` / `workflow_phase.py` 的 `resolve_effective_platform` /
`config.py` 的 `get_codex_dispatch_mode` / `task_store.py` 的 `.codex` 特判），
为 Pi 增加同名机制。唯一分叉点：**默认值 inline**（上游 Codex 默认 auto；
本 fork 用户是唯一消费者，全局不要子代理，零配置生效）。

## 数据流

```
.trellis/config.yaml (pi.dispatch_mode)
  ├─ Python 侧：config.get_pi_dispatch_mode()
  │    ├─ workflow_phase.resolve_effective_platform("pi") → "pi-inline" | "pi-sub-agent"
  │    │    └─ get_context.py --mode phase --platform pi 的 marker 过滤
  │    └─ task_store._has_subagent_platform()：.pi 目录按 mode 判断（jsonl 门槛）
  └─ TS 侧（Pi 扩展 index.ts.txt）
       ├─ readPiDispatchMode(root)：节块行解析 config.yaml（镜像 readContextInjectionLimits 模式）
       └─ workflowBreadcrumb()：mode=inline 时 lookup 优先 `${status}-inline`
```

## 改动点明细（全部在 packages/cli/src/templates/）

| # | 文件 | 改动 |
|---|---|---|
| 1 | `trellis/scripts/common/config.py` | `DEFAULT_PI_DISPATCH_MODE = "inline"`；`get_pi_dispatch_mode()` 镜像 codex 版（auto/inline 归一、sub-agent 别名、非法回退 inline；CLI 侧 warn 一次） |
| 2 | `trellis/scripts/common/workflow_phase.py` | `resolve_effective_platform`：`platform == "pi"` → `pi-sub-agent`（auto）/ `pi-inline`（inline）。注意放在 `_PLATFORM_MARKER_LABELS` 查表之后（pi 无 label 映射，顺序无冲突，但保持与 codex 分支同构） |
| 3 | `trellis/scripts/common/task_store.py` | `_SUBAGENT_CONFIG_DIRS` 移除 `".pi"`；新增 `_PI_CONFIG_DIR = ".pi"`，在 `_has_subagent_platform` 里镜像 codex：`get_pi_dispatch_mode(repo_root) == "auto"` 时才计为子代理平台 |
| 4 | `trellis/workflow.md` | ① 6 个派发 marker 块（1.2/1.3/1.5/Routing/2.1/2.2）`Pi` → `pi-sub-agent`；② 5 个 inline 块（1.2/1.3/Routing/2.1/2.2）加 `pi-inline`；③ "Sub-agent dispatch protocol applies to all platforms…" 段落包进派发平台 marker 块（inline 平台不再看到）；④ 相关 HTML 注释 "Codex-only" → 提及 pi；⑤ Phase Index 1.3 行的平台说明文字更新 |
| 5 | `trellis/config.yaml` | 新增 `pi:` 注释文档段（镜像 codex 段落措辞，注明默认 inline 与回退 auto 的写法） |
| 6 | `pi/extensions/trellis/index.ts.txt` | ① 新增 `readPiDispatchMode(repoRoot): "inline"|"auto"`（节块扫描 `^pi\s*:`，缩进内找 `dispatch_mode`，复用 `stripInlineComment`/`unquoteYaml`；缺文件/坏格式 → 默认 inline，不 warning）；② `workflowBreadcrumb`：`const body = templates[\`${lookup}-inline\`] ?? templates[lookup] ?? …`，仅当 mode=inline 时启用 -inline 优先；③ 顶部缓存（模块级变量，进程内一次解析，config 变更随会话重启生效） |
| 7 | `cli/test/regression.test.ts` | 镜像 codex 用例（约 8022–8314 行区域）：pi 默认 inline 取 -inline 面包屑；显式 auto 取 plain；marker 过滤 pi-inline/pi-sub-agent 各自非空；`_has_subagent_platform` 对 .pi 的两种取值 |

## 关键设计决策

1. **默认 inline vs 上游 auto**：本 fork 唯一消费者明确全局不要子代理 → 默认 inline 零配置生效。
   代价：与上游语义分叉。缓解：分叉点集中在 `DEFAULT_PI_DISPATCH_MODE` 常量与 workflow.md
   marker 列表，合并上游时可用 git 定位。
2. **marker 命名 `pi-sub-agent` / `pi-inline`**（而非继续用 `Pi` + 布尔开关）：
   与 `codex-sub-agent`/`codex-inline` 命名同构，`filter_platform` 模糊匹配天然支持，
   registry-invariants 测试（每个平台 id 路由非空）在两种 mode 下都能命中对应块。
3. **TS 侧不做通用 YAML 解析**：复用扩展里已有的「节块行解析」模式
   （`readContextInjectionLimits` 先例），只解析 `pi:` 节的 `dispatch_mode` 键。
   通用解析是过度设计；损坏配置安全回退默认，无每回合 warning。
4. **不改 `trellis_subagent` 工具注册**：显式派发能力保留（研究/并行场景可能用），
   只是流程文本（面包屑 + 步骤详情）不再指挥派发。删除工具是额外行为变更，超出需求。
5. **兼容与迁移**：旧项目 workflow.md 的 marker 是 `Pi`，新 resolve 出的 `pi-inline`/
   `pi-sub-agent` 都匹配不上 → 步骤详情路由为空。**修复路径就是既有的 `trellis update`
   整文件替换（workflow-state-contract.md 的 Update boundary 条款）**，不新增迁移代码。
   过渡态安全：面包屑 `-inline` 块旧文件里已存在（Codex inline 用），单回合指引不丢。
   面包屑 plain 块（auto 模式）旧文件同样存在。仅 `Phase Index`/步骤详情降级，update 后恢复。

## 测试与验证

- 单测（vitest，cli 包）：`pnpm -C packages/cli test`（regression.test.ts 新增 pi 用例）。
- 手工冒烟（本仓库 dogfood）：
  - `python .trellis/scripts/get_context.py --mode phase --platform pi` → 输出含
    `pi-inline` 路由块（trellis-before-dev 流程），不含 "Spawn the implement sub-agent"。
  - 设 `pi.dispatch_mode: auto` → 同命令输出派发块。
  - Pi 扩展：模拟 workflowBreadcrumb 输入，inline 时含 "Do not dispatch"。
- 试点项目验证（xizhao-design-project）：装 tarball → `trellis update` → 重跑上述命令 +
  开 Pi 会话看每回合 `<workflow-state>` 内容。

## 回退方案

- 代码：git revert（源码仓单任务单提交）。
- 行为：不想 inline 的项目在自身 config.yaml 写 `pi.dispatch_mode: auto`。
- 分发：tarball 不覆盖已装版本（npm i 指定路径才升级）；项目侧 `trellis update`
  有 hash 保护，用户本地改过的文件不会被静默覆盖。

## 风险

| 风险 | 缓解 |
|---|---|
| 上游合并冲突（默认值 + marker 列表分叉） | 分叉点集中、提交独立；fork 用户接受 |
| 旧项目未跑 update → pi 步骤详情为空 | 分发流程硬性要求 update；面包屑不丢；PRD 已注明 |
| TS 侧节块解析对非典型 yaml（flow style、锚点）不识别 | 与上游 `parse_simple_yaml` 同一能力边界；安全回退默认 inline |
| workflow-state-contract.md 的 registry 不变量（平台路由非空） | 两种 mode 下均有对应 marker 块，回归测试断言 |
