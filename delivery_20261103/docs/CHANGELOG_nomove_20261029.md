# 全局禁用 Move 动作（no_move_guard）变更记录

## 需求与范围

用户指示：在所有训练中去掉所有 Move 动作（动作号 196–244，共 49 个 `Move{from,to}`）。
按项目既定一致性原则（训练采样/logp/PPO/锚 KL/评估/竞技场适配器共用同一有效 mask），
本次变更同时覆盖训练、库评估与竞技场适配器，并保持探针诊断同口径。

## 实现

新增 `no_move_guard`（0/1），屏蔽方式与 repeat_freeze 护栏同模式，逐决策生效：

1. **训练采样** `AutoTavern/train_ppo.py::collect`：learner 与 league 对手 mask 在
   guard_mask/trinket_mask/repeat_freeze 之前整体清零 Move 列；`aug_mask` 存入
   batch["m"]，因此 logp、PPO update、锚 KL 自动同口径。新计数
   `no_move_masked`（原本合法的 Move 条目数）、`no_move_rows`、
   `opponent_no_move_masked` 进入 diagnostics。
2. **训练入口** `opponent_library/train.py`：`--no-move-guard`（默认继承 checkpoint
   `policy_guards.no_move_guard`）；显式引入/退出必须出现在命令行，resume 时与
   checkpoint 不一致则拒绝（引入时打印审计行）；checkpoint 保存
   `policy_guards.no_move_guard`。
3. **库评估** `opponent_library/evaluate.py`：`resolve_no_move_guard` 从 checkpoint
   继承、多模型必须一致；play() 同序屏蔽；协议在启用时记录 `no_move_guard: 1`
   并改变 protocol_id，防止与旧协议成绩互比。
4. **竞技场适配器** `my_agent_v3.py`：`guard_settings` 读取
   `NO_MOVE_GUARD` 环境变量或 checkpoint `policy_guards.no_move_guard`（旧权重
   默认 0，行为不变）；`select_action` 在其他护栏之前清零 Move。
5. **诊断工具**：`arena_preflight.py::reference_greedy`、
   `probe_policy_behavior.py::make_selector`、`run_behavior_probe.py`
   （`--no-move-guard`，默认取 checkpoint）同口径。

## 部署

- 新运行时根（隔离副本）：`…/optimization_20261029_01/nomove_20261029/`
  （由 recovery_code 复制；注意 recovery_code 内 `AutoTavern` 与 `packages` 是符号
  链接，分别指向 `code/engine_source` 与 `/ai/ceshi/.../packages`，本次由此查明
  `import train_ppo` 实际解析自各运行时根的 `engine_source/train_ppo.py`）。
- 回归：`test_evaluate_guards` / `test_train_safety` / `test_action_guard` /
  `test_environment` / `test_repeat_freeze_integration` /
  `test_training_correctness` 全部通过；`test_batch.py` 因 venv 无 pytest 跳过。
- 冒烟：真实权重 60 拍 collect，`no_move_masked=2088`、`no_move_rows=73`、
  `actual_move_n=0`、logp 全有限。
- 三路切换（SIGTERM 安全停 → 保留 Adam/锚/RNG resume → `--no-move-guard 1`）：
  | 运行 | 切换点 | 恢复后验证 |
  |---|---|---|
  | control_permanent_eidfix（GPU0） | iter 1590s | actual_move_n=0 |
  | finite_anchor_eidfix（GPU1） | iter 1840s | actual_move_n=0 |
  | finite_anchor_fastlr_eidfix（GPU2，金丝雀） | iter 1398 | actual_move_n=0 |
  切换后每迭代屏蔽约 71–85 万个原本合法的 Move 条目；吞吐不变（约 2050–2330 决策/秒）。
- 端到端：新 checkpoint（policy_guards 含 no_move_guard=1）导出候选 → 256 样本
  预检 256/256 greedy 一致；库评估 1 局 C 档动作号 0–54 无 Move，协议记录
  `no_move_guard:1`（protocol_id 5f295f20…）。

## 行为影响与边界

- Move 占切换前决策的 7.5–15.2%（GPU1 最高，10 回合后尤甚），该部分动作空间
  被永久关闭；站位类收益（攻击顺序、磁力拼接位）改由 Play/Magnetize 的插入位置表达。
- 旧 checkpoint（无该字段）在适配器/评估中默认允许 Move，行为与历史一致；
  新 checkpoint 起全局禁用。
- 单局库 dev 名次短期波动属预期（策略需要适应 49 个动作的缺失）；
  晋升仍以双门槛（原生固定回归 + 库 dev，新协议 ID）为准。
- 本文档与 launch.json（各 run 目录，含 no_move_guard=1 与 resumed_from）为审计依据。
