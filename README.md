# 第 27 项 Codified Context · MCP 检索复现

本项目复现 Codified Context 的本地上下文检索。MCP 服务读取一条游戏开发任务，按关键词给代码库的子系统打分，再返回相关模块和建议查看的文件路径。实验比较三种关键词匹配规则在同一批 26 道评估题上的表现。 [论文 §3.3.1](https://arxiv.org/html/2602.20478v1#S3.SS3.SSS1) · [作者的工具说明](https://github.com/arisvas4/codified-context-infrastructure/tree/6b0408fe86d8e802bcbd00cd6d2d057f288f518a/mcp-server)

作者将项目知识分为项目规则、领域专家智能体和按需查阅的文档三层。本实验复现第三层的 `find_relevant_context` 工具。 [论文三层架构](https://arxiv.org/html/2602.20478v1#S3)

## 三步实验

三版使用同一份 [cases.json](cases.json)。其中有 26 道自编评估题和一条作者文档里的示例。每道评估题标注一个预期优先查看的子系统。三版共用子系统索引、计分权重、排序和文件推荐规则，依次运行同样的查询。

1. **作者原版（子串匹配）。** 以作者 v1.0.0 的 MCP 服务为基线。原版检查关键词是否出现在任务描述里，例如 `screenshot` 会触发瞄准模块的 `screen`。用 `fix camera offset` 核对作者示例，原版将 `rendering` 排在第一位，建议路径包含 `coordinate-systems.md`。作者示例单独记录，汇总指标只计 26 道评估题。 [作者示例](https://github.com/arisvas4/codified-context-infrastructure/blob/v1.0.0/mcp-server/README.md#key-data-structures)
2. **严格整词。** 把关键词命中改为词边界匹配，排除 `screenshot` 中的 `screen`。这一步也漏掉了正常复数。输入 `Sprites and textures flicker.` 时，严格整词版返回空列表。 [第一步补丁](runs/verified-26-three-variants/01_original_to_word_boundary.diff)
3. **整词加常规复数。** 先检查完整关键词，未命中时再尝试关键词最后一个词的常规复数。于是 `sprites` 和 `textures` 可以命中 `sprite` 和 `texture`，`text` 仍无法命中 `textures`。该题的渲染模块回到第一位。 [第二步补丁](runs/verified-26-three-variants/02_word_boundary_to_plural.diff)

## 安装和复跑

需要 Python 3.12 和 Git。在 PyCharm 的 **Terminal** 或系统终端执行以下命令。

```bash
git clone https://github.com/bri702/FTEC5660-Reproducibility.git
cd FTEC5660-Reproducibility
python3.12 -m venv .venv
./.venv/bin/python -m pip install -r requirements.txt
./.venv/bin/python -B runs/verified-26-three-variants/run_three_variants.py --project . --out runs/three-variant-26
```

已有项目时，从项目根目录执行最后一行即可。每次复跑请给 `--out` 指定一个新目录。脚本自动生成临时改版，依次启动三套本地 MCP 服务。装好依赖后可离线运行，无需模型或 API Key。 [复跑脚本](runs/verified-26-three-variants/run_three_variants.py)

## 26 题实测结果

| 版本 | Top-1 命中 | Top-3 命中 | 前三名未标注候选总数 |
| --- | ---: | ---: | ---: |
| 作者原版 | 21/26（80.8%） | 25/26（96.2%） | 42 |
| 严格整词 | 18/26（69.2%） | 21/26（80.8%） | 20 |
| 整词加常规复数 | 23/26（88.5%） | 26/26（100%） | 23 |

Top-1 和 Top-3 分别统计预期子系统排第一、进入前三的题数。未标注候选总数是逐题累加前三名中未列入该题人工标签的候选。

加入常规复数后，相比严格整词，两项命中各增加 5 道。相比作者原版，第一名命中多 2 道，前三名命中多 1 道，未标注候选总数从 42 降到 23。

- 手柄重连题中，原版把 `fails` 里的 `ai` 计给 AI 模块。改进版排除这次命中，`input` 从第二升到第一。
- 读档后音乐播放两次时，原版从 `saved` 和 `loading` 命中 `save` 和 `load`。改进版将 `audio` 从第三升到第一。
- 血条颜色错误时，排除 `health` 里的 `heal` 让 `ui` 从第四升到第三。`health` 仍是战斗和伤害模块的完整关键词。
- 步枪按一次扳机却发射两枚弹丸时，改进版仍将瞄准排第一、战斗排第二。`fire` 和 `projectile` 在瞄准模块的索引中都是完整关键词。

## 结果文件

| 文件 | 内容 |
| --- | --- |
| [metrics.csv](runs/verified-26-three-variants/metrics.csv) | 三个版本的汇总指标，每版一行 |
| [case_results.csv](runs/verified-26-three-variants/case_results.csv) | 81 行逐题结果，每版有 26 道评估题和一条作者示例 |
| [raw_mcp.json](runs/verified-26-three-variants/raw_mcp.json) | MCP 原始响应，包括子系统排名、命中关键词和建议文件 |
| [01_original_to_word_boundary.diff](runs/verified-26-three-variants/01_original_to_word_boundary.diff) | 从原版到严格整词 |
| [02_word_boundary_to_plural.diff](runs/verified-26-three-variants/02_word_boundary_to_plural.diff) | 从严格整词到整词加常规复数 |

复跑时，指定目录还会保存 `run.json` 和各版服务日志。

报告中原版与最终版的两版本对照也有独立记录。 [两版本对照](runs/verified-26-game)

## 范围和局限

本实验只测量 MCP 在这 26 道任务上的检索结果。每道题只标注一个优先模块，其余候选也可能对排查有帮助。测试题包含刻意设计的词内误匹配案例，结果适用于这组题。作者的论文还研究了开发会话和案例，本实验的指标无法估计智能体的代码正确率或开发效率。 [论文评估方法与范围](https://arxiv.org/html/2602.20478v1#S4.SS1)

## 调试记录

### 从严格整词匹配到整词加复数

第一轮用严格整词匹配减少词内误命中。为比较规则调整的影响，三个版本按相同顺序运行同一批 26 道评估题，使用同一份标签和评分规则。相比作者原版，严格整词版的 Top-1 从 21/26 降到 18/26，Top-3 从 25/26 降到 21/26。

逐题检查发现，`render-plurals`（`Sprites and textures flicker.`）在严格整词版中返回空列表。题目用了 `sprites` 和 `textures`，而索引中的关键词是 `sprite` 和 `texture`。复数的 `s` 紧接在关键词后面，严格整词在这个位置找不到词边界，因此漏掉了两处匹配。

第二轮保留词边界，并在完整关键词未命中时尝试关键词最后一个词的常规复数。该题的 `rendering` 回到第一名，`textures` 中的 `text` 仍被排除。复跑同一批题后，Top-1 升至 23/26，Top-3 升至 26/26，均比严格整词版多命中 5 道。前三名未标注候选总数从 20 增至 23，仍低于作者原版的 42。 [逐题结果](runs/verified-26-three-variants/case_results.csv) · [三版本汇总](runs/verified-26-three-variants/metrics.csv)

## 源码与许可

`upstream/server.py` 保留作者 v1.0.0 的源码，固定于提交 `6b0408fe86d8e802bcbd00cd6d2d057f288f518a`。上游代码采用 MIT 许可证，许可文件随源码提供。 [固定版本的作者源码](https://github.com/arisvas4/codified-context-infrastructure/tree/6b0408fe86d8e802bcbd00cd6d2d057f288f518a) · [MIT 许可证](https://github.com/arisvas4/codified-context-infrastructure/blob/6b0408fe86d8e802bcbd00cd6d2d057f288f518a/LICENSE)
