# 酒馆战棋 AI 交付存档

本仓库保存 `delivery_20261103` 的原始目录与文件，包括三路训练的模型检查点、运行配置、评估结果、竞技场记录及交接文档。

## 目录

- `delivery_20261103/models/`：三份 PyTorch 训练检查点，使用 Git LFS 保存。
- `delivery_20261103/runs/`：启动参数和训练摘要。
- `delivery_20261103/evals/`：离线评估与行为测试结果。
- `delivery_20261103/arena/`：竞技场对局结果和验证记录。
- `delivery_20261103/docs/`：交接说明及变更日志。

完整交接说明见 [DELIVERY_20261103.md](delivery_20261103/docs/DELIVERY_20261103.md)。文档中的服务器路径和训练状态属于交付时点的记录。

## 获取完整模型

安装 [Git LFS](https://git-lfs.com/) 后执行：

```bash
git lfs install
git clone https://github.com/avacx/lushi.git
cd lushi
git lfs pull
git lfs ls-files
```

三份检查点共约 1.34 GiB。未使用 Git LFS 下载时，Git 中的 `.pt` 文件只是指针，不是完整模型。

## 使用范围

这是交付存档，未包含原始训练源码、完整推理程序、对手数据集或炉石客户端控制工具，不能直接运行。模型配置记录的架构为 `mlp-v2-residual`、观测版本为 2；接入时需要核对状态编码、卡牌词表和动作布局。

## 相关项目

- 模拟环境与评测集：[MCG-NJU/AutoTavern](https://github.com/MCG-NJU/AutoTavern)
- 参考论文：[Build, Don't Play](https://arxiv.org/html/2610.33845v1)
- AI 对战平台：[TavernArena](https://rizling.top)

本仓库保存的模型与实验记录不代表论文作者发布的官方 TavernAgent 权重。
