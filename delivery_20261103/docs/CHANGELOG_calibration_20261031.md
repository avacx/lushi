# 强度校准（library_v3_calibrated）变更记录

## 动机

论文(2610.33845v1)第 6 节/附录 C:对手库最大的单项收益来自"按学习者实际遭遇的
强度逐回合校准"(Table 3 消融:未校准 20.9% → 校准 27.2% 第一率,同 checkpoint 同预算)。
本项目测量证实了失配的严重程度:

| 回合 | 库面板中位数(未校准) | 学习者实测面板 | 比值 |
|---:|---:|---:|---:|
| 8 | 555 | 76 | 0.14 |
| 10 | 2946 | 170 | 0.06 |
| 12 | 11067 | 309 | 0.03 |
| 15 | 44070 | 555 | 0.013 |

胜者面板(幸存者偏差+指数增长)在第 10 回合后是学习者实际强度的 20–70 倍——
"库内表现弱"的主因。测量脚本 `maintenance/measure_strength_curves.py`
(64 局原生对局,`board_strength_many` 与库 `strength.panel` 同口径)。

## 实现

1. **library_v3_calibrated**(`build_library_v3.py`,library_id `cbda709de209…`,
   parent=v2 `033c0c35…`):sampling.mode=`strength_calibrated`,
   target_curve=学习者实测逐回合均值(difficulty 1.0=对齐档;>1 可做更难臂),
   strength_band=0.5,nearest_k=8。
2. **library.py** `sample_calibrated`:最近过去回合条目池内,优先目标 ±50% 带内
   均匀抽,空则放宽为 |panel-target| 最近 8 个;绝不取未来回合;
   trace 记录 target_panel/chosen_panel/calibration_band。
3. **environment.py**:按 manifest 模式分派(校准/轨迹/v1 逐回合),stats 新增
   `calibrated_draws/calibration_band_hits`。
4. 教训:manifest 内 dict 的 int 键在 JSON 往返后变 str,canonical 的 sort_keys
   对两种键类型排序不同 → library_id 校验失败;构建时一律用 str 键。

## 验证

- `test_strength_calibration.py`(5 项)通过:采样均值贴近目标且远优于原始池
  中位数(14 回合校准均值 665 vs 目标 473.7 vs 池中位 28375——库在该回合的
  最弱可用面板即 ~665,为库本身的现实约束)、无未来回合、真机环境集成
  (注入面板/目标比值均值 ≈1)。comp_mix/library 回归不受影响。
- GPU2 切换(launch.json 记录 allow-library-switch):首条注入
  target 170.1 / chosen 244.0 / 带内命中。

## 部署矩阵与边界

| GPU | 运行 | 库 | 角色 |
|---|---|---|---|
| 0 | control_permanent | v2 轨迹库(033c0c35) | 对照 |
| 1 | finite_anchor | v2 轨迹库(033c0c35) | 最强线/对照 |
| 2 | lr1e5 对照(从 5760 快照) | **v3 校准库(cbda709d)** | 校准实验臂 |

- difficulty=1.0 为"对齐档";论文的分级阶梯(更难臂)用 difficulty>1 另建库版本。
- 库在 14+ 回合没有贴近学习者的候选(最弱面板即 ~665),这是数据本身的限制;
  校准已选到最近可行值。后续可从非胜者来源补充弱面板。
- 评估对比须用各库自己的 protocol(library_id 已隔离)。
