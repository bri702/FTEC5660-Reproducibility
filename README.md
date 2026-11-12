# 第 27 项 Codified Context · MCP 检索复现

本项目复现论文 §3.3.1 的本地上下文检索。给作者的 MCP 服务一条开发任务，它按关键词给代码库的子系统打分，再返回相关模块和建议查看的文件路径。这个流程和 HW2 的 SocialGraph MCP 类似，都是把问题交给工具，再读取结构化的返回结果。这里查询的是项目资料。 [论文 §3.3.1](https://arxiv.org/html/2602.20478v1#S3.SS3.SSS1) · [作者的工具说明](https://github.com/arisvas4/codified-context-infrastructure/tree/6b0408fe86d8e802bcbd00cd6d2d057f288f518a/mcp-server)

## 论文背景

作者把编程 Agent 需要的项目知识分为三层。项目规则在每次会话开始时加载。网络同步、代码审查等专家 Agent 按任务调用。具体模块的设计文档则通过 MCP 按需检索。本实验验证第三层的一个工具 `find_relevant_context`，用本地 Python 客户端通过标准输入输出（stdio）连接作者的服务。安装依赖后可以离线运行，所需的输入是本地任务描述和作者随服务发布的索引。 [论文三层架构](https://arxiv.org/html/2602.20478v1#S3)

## 安装和运行

需要 Python 3.12 和 Git。在 PyCharm 的 **Terminal** 或系统终端逐行执行：

```bash
git clone https://github.com/bri702/FTEC5660-Reproducibility.git
cd FTEC5660-Reproducibility
python3.12 -m venv .venv
./.venv/bin/python -m pip install -r requirements.txt
./.venv/bin/python -u run_project27.py
```

也可以用 PyCharm 打开克隆后的项目，把解释器设为项目里的 `.venv/bin/python`，再右键 `run_project27.py` 选择 **Run**。脚本默认读取 `cases.json`，同时运行原版和修改版，并打印结果目录。安装依赖后，实验可以离线运行，无需模型或 API Key。

## 实验怎么做

1. **复现作者记录的行为。** 对作者说明中的检索示例调用原版 MCP，核对返回的子系统和文件路径。示例结果单独记录。
2. **固定 26 条游戏项目相关题。** `cases.json` 包含 15 条原有的开发任务和 11 条按游戏开发场景编写的故障描述。每题标注一个预期优先查看的模块。故障题包含容易命中其他模块的词，例如截图里的 `screenshot` 会触发瞄准模块的 `screen`。
3. **只改子系统关键词的命中条件。** 原版检查子串，所以 `repair` 会命中 `ai`。新版要求词边界，并允许关键词最后一个单词的常规复数，比如 `sprites` 命中 `sprite`。`ai`、`ui` 这类短词只做整词匹配。两版使用同一份索引、计分权重、排序和文件返回规则。生成修改版的源码差异保存在 `whole_word_plural.diff`。
4. **同一批题调用两版 MCP。** 客户端通过本地标准输入输出启动两个服务，核对工具列表和子系统索引，再逐题调用 `find_relevant_context`。

| 指标 | 怎么算 |
| --- | --- |
| 第一名命中率 | 有目标模块的题里，第一名属于预期标签的比例 |
| 前三名命中率 | 有目标模块的题里，前三名至少出现一个预期模块的比例 |
| 相关题前三中的未标注候选数 | 对每条相关题，统计前三名里不在其预期标签中的模块，再把数量加起来 |

## 26 题实测结果

两版各处理了 26 条评估题，另外运行了作者说明中的示例 `fix camera offset`。示例中，`rendering` 和 `aiming` 都因命中 `camera` 各得 1 分，`rendering` 排第一，建议路径包含 `coordinate-systems.md`。客户端还对第一名调用了 `get_files_for_subsystem`。 [作者的 MCP 说明](https://github.com/arisvas4/codified-context-infrastructure/blob/v1.0.0/mcp-server/README.md#key-data-structures)

| 指标 | 原版子串匹配 | 整词＋复数 |
|---|---:|---:|
| 26 条游戏相关题第一名命中 | 21/26（80.8%） | 23/26（88.5%） |
| 26 条游戏相关题前三名命中 | 25/26（96.2%） | 26/26（100%） |
| 前三中的未标注候选总数 | 42 | 23 |

- 读档后音乐播放两次时，原版在 `saved` 和 `loading` 中分别命中 `save` 和 `load`，让服务层和资源加载排在音频前面。改版把音频从第三升到第一。
- 无界面截图全黑时，原版从 `screenshot` 里命中 `screen`，把瞄准模块排在第二。改版只返回开发工具模块。
- 手柄拔掉再插回去后识别不到时，原版把 `fails` 里的 `ai` 算成命中。改版去掉这次误匹配，`input` 从第二升到第一。
- 步枪按一次扳机却发射两枚弹丸时，改版仍把瞄准排第一、战斗排第二。`fire` 和 `projectile` 是瞄准索引里的完整词，词边界和复数处理无法判断句子在说哪个环节。
- 血条颜色错误时，`ui` 从第四升到第三。改版去掉了 `health` 里的 `heal`，但 `health` 仍是战斗和伤害模块的完整关键词。
- 原有 15 条相关题在两版都保持了第一名命中。复数规则保住了 `sprites` 对 `sprite`、`textures` 对 `texture` 的有用匹配，同时去掉 `textures` 对 `text` 的词内匹配。

[汇总指标](runs/verified-26-game/metrics.csv) · [逐题对照](runs/verified-26-game/case_results.csv) · [MCP 原始响应](runs/verified-26-game/raw_mcp.json) · [环境与输入校验](runs/verified-26-game/run.json) · [源码差异](runs/verified-26-game/whole_word_plural.diff)


## 怎么查看输出

默认每次运行都在 `runs` 下创建一个以 UTC 时间命名的新目录。两次 MCP 服务的标准错误输出保存在同目录的日志文件。

| 文件 | 内容 |
| --- | --- |
| `metrics.csv` | 原版和修改版的汇总指标 |
| `case_results.csv` | 每道题的预期模块、实际排序和命中结果 |
| `raw_mcp.json` | MCP 服务返回的原始结果，包括命中关键词和文件建议 |
| `run.json` | 本次运行的输入、源码与环境记录 |
| `whole_word_plural.diff` | 原版和修改版的源码差异 |

## 范围和局限

论文 §4 的开发会话和案例观察覆盖了更大的系统。这里的数字只衡量这组任务上的 MCP 检索结果，无法据此估计 Agent 写代码的正确率或开发效率。测试题由本项目编写，其中故障题刻意覆盖容易误匹配的开发场景。预期模块根据公开的子系统索引和任务描述人工指定。未标注的模块有时也能帮助排查，结果适用于这组题。建议路径来自作者的索引，公开仓库只提供部分原项目资料。 [论文评估方法与范围](https://arxiv.org/html/2602.20478v1#S4.SS1) · [作者的数据说明](https://github.com/arisvas4/codified-context-infrastructure/tree/6b0408fe86d8e802bcbd00cd6d2d057f288f518a/data)

## 调试记录

- 用 Python 直接启动作者的 `server.py`，通过 MCP 客户端完成握手和查询。这样可以直接核对作者发布的源码和工具。
- 运行时观察到 MCP 依赖关于类型注解的警告，保存在服务日志里。两组服务均完成查询并生成结果。

## 源码与许可

`upstream/server.py` 保留作者发布的源码，固定于 v1.0.0 的提交 `6b0408fe86d8e802bcbd00cd6d2d057f288f518a`。上游代码采用 MIT 许可证，许可文件随源码提供。 [固定版本的作者源码](https://github.com/arisvas4/codified-context-infrastructure/tree/6b0408fe86d8e802bcbd00cd6d2d057f288f518a) · [MIT 许可证](https://github.com/arisvas4/codified-context-infrastructure/blob/6b0408fe86d8e802bcbd00cd6d2d057f288f518a/LICENSE)
