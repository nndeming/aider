# Aider 源码学习指南

> 个人研究笔记：基于对代码库的整体梳理，记录推荐的研究顺序与测评方法。
> 代码位置引用格式为 `文件路径:行号`。

## 一、整体架构（一句话版）

aider 的本质是一个**聊天循环**：用户输入 → 组装提示词（含仓库地图 + 待编辑文件内容）→ 调 LLM（通过 litellm）→ 解析回复中的编辑内容 → 应用到磁盘 → git 提交。

各层职责：

- **入口/配置**：`aider/main.py`（`main()` 在 `main.py:451`）、`aider/args.py`
- **聊天循环**：`aider/coders/base_coder.py`（`Coder` 基类，约 2485 行，核心中的核心）
- **编辑格式**：`aider/coders/` 下的各种 Coder 子类（diff / whole / udiff / patch…）
- **模型层**：`aider/models.py`（`Model` 类）、`aider/llm.py`（litellm 懒加载封装）
- **仓库理解**：`aider/repomap.py`（tree-sitter + PageRank 生成仓库地图）、`aider/repo.py`
- **终端交互**：`aider/io.py`（`InputOutput`，prompt_toolkit + Rich）、`aider/commands.py`（斜杠命令）

## 二、推荐研究顺序

### 第 0 步：跑起来

按 `CONTRIBUTING.md` 建 venv、`pip install -r requirements/requirements-dev.txt`，然后 `pytest` 确认测试全绿。调试入口是 `aider/main.py:451` 的 `main()`，它支持 `return_coder=True`，测试里常用这个直接拿到 coder 对象驱动，调试时也可以利用。

### 第 1 步：启动流程（半天）

顺着 `main()` 读一遍：参数解析（`args.py`）→ git 仓库初始化 → 创建 `Model` / `GitRepo` / `InputOutput` / `Commands` → `Coder.create()`。目标是知道"一个 Coder 对象诞生前需要哪些部件"。

### 第 2 步：聊天主循环（重点，1-2 天）

精读 `aider/coders/base_coder.py`，从 `run()`（约 :876）→ `run_one()` → `send_message()` 这条线读。这是全项目价值密度最高的文件。配套看 `aider/prompts.py`（只有 61 行，系统提示词）和 `aider/diffs.py`。

### 第 3 步：编辑格式的解析与应用

从默认的 `editblock_coder.py` 入手：LLM 输出搜索/替换块 → 模糊匹配容错 → 应用编辑。然后挑一个对比着读，比如 `udiff_coder.py` 或 `wholefile_coder.py`，理解为什么需要多种格式、`Coder.create()` 工厂（`base_coder.py:125`）怎么切换。`tests/basic/test_editblock.py` 是绝配读物——大量边界用例。

### 第 4 步：模型层（快读）

`aider/llm.py` 很短，亮点是 `LazyLiteLLM`（litellm import 要 1.5 秒所以懒加载）。然后看 `models.py` 里 `Model.send_completion()` 如何组装参数、模型名前缀如何决定默认 edit_format、`model-settings.yml` 如何补充元数据。

### 第 5 步：RepoMap（aider 最有特色的部分，值得单独研究）

`aider/repomap.py`：tree-sitter 提取符号 → 符号引用图做 PageRank 排名 → 按 token 预算截断。`aider/queries/` 下的 `.scm` 文件是 tree-sitter 查询。建议先跑 `tests/basic/test_repomap.py` 里的用例，看输入输出，再回头看实现。

### 第 6 步：外围层（按需）

`io.py`（终端交互抽象，测试时换假 IO）、`commands.py`（`/add`、`/diff`、`/commit` 等斜杠命令，`cmd_` 前缀方法分发）、`watch.py`、`history.py` 等。这些相对独立，用到再读。

### 两个实用建议

1. **测试是最好的文档**。`tests/basic/` 基本一个模块对一个 `test_*.py`，每研究一个模块就配上对应测试读，比干读代码效率高很多。
2. **带着实验读**：每个阶段都实际跑一下，比如在调试器里断在 `send_message()`，看一次真实的消息组装过程，比纯读印象深得多。

## 三、测评（Benchmark）

`benchmark/` 目录是专门的测评工具集，官网 leaderboard 数据是其产物。包含两套基准：

### 1. Polyglot Benchmark（主力）

基于 [Exercism](https://github.com/exercism/python) 编程练习题的端到端测评：给 LLM 一个自然语言需求，让它用 aider 编辑代码并通过单元测试。同时测 LLM 编程能力和 aider 的**编辑格式解析能力**。`benchmark/benchmark.py` 是主入口，`benchmark/prompts.py` 放提示词。

### 2. SWE-bench 支持

`benchmark/swe_bench.py` 是绘图/统计脚本，`swe-bench.txt` 和 `swe-bench-lite.txt` 是各模型成绩数据文件（真实 GitHub issue 修复任务的通过率）。

### 周边工具

- `rungrid.py` — 批量跑多个模型 × 编辑格式的组合测评
- `benchmark.py --stats <目录>` — 从测评结果目录生成统计报告
- `over_time.py`、`plots.py`、`plot.sh`、`problem_stats.py` — 趋势图和题目维度分析
- `Dockerfile`、`docker.sh`、`docker_build.sh` — 测评专用容器（**必须跑在 Docker 里**，因为要无人工审查地执行 LLM 生成的代码）
- `test_benchmark.py` — 测评脚本自身的测试

### 测评报告

核心指标是 `pass_rate_1` / `pass_rate_2`（第 1/2 次尝试就全部测试通过的百分比），报告为 yaml 格式，还记录模型、edit_format、git commit hash、每题耗时、总成本、格式错误数等，保证可复现。官方历史成绩存放在 `aider/website/_data/`，即 [aider 官网 leaderboard](https://aider.chat/docs/leaderboards/) 的数据来源。

### 运行方式

```bash
mkdir tmp.benchmarks
git clone https://github.com/Aider-AI/polyglot-benchmark tmp.benchmarks/polyglot-benchmark
./benchmark/docker_build.sh   # 需要 Docker
./benchmark/docker.sh         # 进容器
pip install -e .[dev]
./benchmark/benchmark.py my-run --model gpt-4o --edit-format diff --threads 10 --exercises-dir polyglot-benchmark
./benchmark/benchmark.py --stats tmp.benchmarks/...  # 生成报告
```

常用参数：`--model`（模型名，与 aider 一致）、`--edit-format`（编辑格式）、`--threads`（并行数，新模型先从 1 开始）、`--num-tests`（只跑前 N 题，调试时用）、`--keywords`（按名字过滤题目）。

注意：脚本面向开发/研究用途；部分脚本为 bash 编写，Windows 上需走 Docker 或 WSL。

### 与研究源码的关系

benchmark 是很好的"实验台"：改了 edit format 或提示词后，用 `--num-tests 20` 先小规模验证，再全量跑 225 题对比 pass_rate。`benchmark/benchmark.py` 本身也展示了如何以编程方式驱动 aider（本质就是不断调用 aider 的 coder 循环），与精读 `base_coder.py` 互为印证。
