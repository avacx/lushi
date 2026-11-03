# 套路混合训练（library_v2_comp）变更记录

## 需求

从实战样本（人工胜局库）学习流派，从成行样本（每条轨迹的最终面板）与成行回合数中
提取关键阵容角色，形成阵容套路册，之后持续对各套路做混合训练。

## 挖掘结论（train split, 814 轨迹 / 10,229 条目）

- 轨迹内 archetype 标注 100% 一致（0 冲突）；
- **成行样本集中在第 12–13 回合**（均值 11.6–13.0，最大 17–18），显著缓解 v1 后期覆盖断层；
- 取 n≥27 的 13 个流派 + unlabeled_mixed(132) = **14 个套路**，每个套路记录
  成行回合分布、成行档位分布与签名卡（如 beast_lobster：美味龙虾/逐亡陆行鸟/猎头狮鹫；
  demon_boost_shop：邪能野猪人/热气球测绘员 等）。

## 实现

1. **library_v2_comp**（`build_library_v2.py`）：复制 v1 全部条目与 split 划分，
   新增 `sampling.mode=trajectory_comp_balanced`、套路册与轨迹索引；
   `library_id=033c0c35afb9…`（digest 规则与 v1 一致：排除 library_id/created_utc 后
   全量 canonical sha256），`parent_library_id` 指向 v1。构建脚本末尾调用
   validate_library 自检（含文件哈希与 split 泄漏检查）。
2. **library.py**：`trajectory_mode` 库暴露 `assign_trajectory`（每局每席均匀抽套路、
   套路内均匀抽轨迹；空套路回退全轨迹）与 `sample_trajectory`（轨迹内最近过去条目，
   无更早条目返回 None）；dev/exam 分支按条目 archetype 映射套路，未命中归入
   unlabeled_mixed。
3. **environment.py**：`_reset` 清空每局轨迹绑定；`_inject_and_fight` 在
   trajectory_mode 下每席绑定轨迹后按 `sample_trajectory` 注入（同席跨回合跟随同一
   对手阵容发育），无更早条目回退旧逐回合采样并记录 `trajectory_fallback`；
   trace 记录 comp/trajectory；新增 stats `trajectory_draws/trajectory_fallbacks`。
4. **train.py**：`--allow-library-switch`（默认 0）——resume 换 library_id 必须显式
   声明并打印审计行，否则维持原拒绝行为。

## 验证

- 单测 `test_comp_mix.py`（套路加载/14 套路、1400 次指派均衡性、轨迹连贯与时序规则、
  回退、真机环境集成：trace 含 comp/trajectory、同局同席轨迹唯一、多套路覆盖）
  + `test_library.py` / `test_environment.py` / `test_repeat_freeze_integration.py`
  全部通过。
- 三路切换（SIGTERM 安全停 → 保留 Adam/锚/RNG resume，`--allow-library-switch 1`）：
  全部在 v2 库上恢复，`actual_move_n=0` 保持，吞吐不变（2064–2559 决策/秒）。
- 实时 encounters 抽查：最近 56 次注入覆盖 13–14 个套路、0 回退、36/36 席位轨迹连贯；
  library_id 均为新 id。

## 边界与后续

- 套路册由 train split 构建；dev/exam 仅按 archetype 映射用于评估，未参与套路归纳。
- 轨迹模式下面板强度跟随真实人工对局（不再受 C 档曲线约束），launch 参数中的
  curriculum 仍记录但仅作审计；这是有意的强度上移（C 档已近解决：前四率 62.5%）。
- 对比评估须使用 v2 库（protocol 含 library_id，自动与 v1 成绩隔离）。
- 后续观察：v2 采样下的库 dev 与原生回归需在训练推进后用同协议重评。
