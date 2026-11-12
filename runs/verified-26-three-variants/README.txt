项目 27 同题三版本复跑

每版运行 26 道评估题和一条作者示例。汇总指标只统计 26 道评估题。
metrics.csv 是汇总表，case_results.csv 是 81 行逐题记录，raw_mcp.json 是 MCP 原始响应。
压缩包内的 cases.json 与仓库根目录的题目文件一致。

两次修改按顺序记录在 01_original_to_word_boundary.diff 和 02_word_boundary_to_plural.diff 中。
第一步改为严格整词匹配。第二步保留词边界，并在关键词未命中时尝试常规复数。

如需复跑，在原项目仓库根目录安装依赖，然后执行：

    ./.venv/bin/python -B runs/verified-26-three-variants/run_three_variants.py --project . --out runs/three-variant-26
