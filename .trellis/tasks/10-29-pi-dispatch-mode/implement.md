# 执行计划：pi.dispatch_mode

前置：design.md 已定稿；改动全部在 `C:\idealProject\github\Trellis\packages\cli\src\`（模板层）。

## 执行清单（有序）

### A. 模板层实现

- [x] A1 `templates/trellis/scripts/common/config.py`：`DEFAULT_PI_DISPATCH_MODE = "inline"` +
      `get_pi_dispatch_mode()`（共享 `_normalize_dispatch_mode`，与 codex 版同一套归一化/容错/告警）。
- [x] A2 `templates/trellis/scripts/common/workflow_phase.py`：
      `resolve_effective_platform` 增加 `pi` 分支（auto → `pi-sub-agent`，inline → `pi-inline`）。
- [x] A3 `templates/trellis/scripts/common/task_store.py`：`.pi` 从 `_SUBAGENT_CONFIG_DIRS`
      移到 codex 式按 dispatch_mode 判断。
- [x] A4 `templates/trellis/workflow.md`：marker 修改（6 派发块 `Pi`→`pi-sub-agent`；
      5 inline 块 +`pi-inline`；dispatch protocol 段落包进派发 marker；注释与 Phase Index 文字更新）。
- [x] A5 `templates/trellis/config.yaml`：`pi:` 文档段。
- [x] A6 `templates/pi/extensions/trellis/index.ts.txt`：`readPiDispatchMode`（带 per-root 缓存）+
      `workflowBreadcrumb` 的 `-inline` 优先 + `trellis_subagent` 工具指导按模式切换（inline 时 explicit-only）。

### B. 测试

- [x] B1 `cli/test/regression.test.ts`：重写 2 个 [pi] 用例为双模式；新增 3 个 [pi-dispatch-mode] 用例
      （resolve 命名空间、jsonl 播种门、扩展模板内容）；`trellis.test.ts` marker 字符串更新。
- [x] B2 `pnpm -C packages/cli test`：28 失败全部与无改动基线一致（Windows 环境噪音：无 python3、
      marketplace 目录缺、会话环境污染），本任务 3 个破坏已修复，无新增失败。
- [x] B3 typecheck ✓ / eslint ✓ / basedpyright 0 errors（68 预存 warning）✓。

### C. dogfood（本仓库自身）

- [x] C1 `.trellis/scripts/**`（字节一致校验通过）、`.trellis/workflow.md`、
      `.pi/extensions/trellis/index.ts` 已同步新模板。
- [x] C2 冒烟：默认 inline 路由 / auto 派发路由 / 无效值回退 inline，全部符合预期。

### D. spec 更新（Phase 3.3 义务）

- [x] D1 `workflow-state-contract.md`：Runtime contract 第 6 条加 Pi 映射。
- [x] D2 `script-conventions.md`（jsonl 播种契约 + 验证矩阵加 pi 两行）、
      `platform-integration.md`（运行时注册表、Class-3 表格、agentCapable 种子条款、测试要求）。

### E. 构建与分发

- [x] E1 `pnpm -C packages/core build` + `pnpm -C packages/cli build`。
- [x] E2 版本 0.6.17 → 0.6.18-gp.1；pnpm pack 出双包至 `dist-package/`（workspace 依赖已解析为精确版本）。
- [x] E3 试点项目（xizhao-design-project）：npm i -g 双 tarball + trellis update（6 文件自动更新，
      备份于 .trellis/.backup-2026-10-29T05-37-40/）+ 验收通过（2.1 inline 五步、Routing inline、
      扩展含 readPiDispatchMode、config.yaml 含 pi 段）。Pi 会话需重启后生效面包屑。
- [ ] E4 其余业务项目同步：由用户执行（安装命令与更新命令见下）。

## 验证命令速查

```bash
pnpm -C C:/idealProject/github/Trellis/packages/cli test
pnpm -C C:/idealProject/github/Trellis/packages/cli build
python -c "import subprocess,sys; sys.exit(subprocess.call([sys.executable, r'C:/idealProject/github/Trellis/.trellis/scripts/get_context.py', '--mode', 'phase', '--platform', 'pi'], cwd=r'C:/idealProject/github/Trellis'))"
```

## 回滚点

- A 区每个文件独立可 revert；B 失败阻塞 C。
- E3 试点失败 → 检查 update hash 冲突路径，必要时项目侧手动替换 workflow.md（等价内容）。

## Review 门

- A+B 完成（测试全绿）后：向用户汇报一次再进入 C/D/E（C 之后是构建分发，涉及外部项目）。
