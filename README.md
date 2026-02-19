# Report_note_generation

这个项目解决一件很具体的事：把“报告原文（PDF/EPUB）”稳定地转成“大模型更容易处理的 Markdown”，再通过专用 GPT 生成“结构化、简化版的报告笔记”，最终形成从原始文档到笔记的可复用工作流。

## 核心工作流

整体 pipeline 分三段，每一段都把“不稳定/不可控”的环节尽量前置变成“可检查/可重复”的本地处理。

(1) 文档 → Markdown（让 LLM 更容易读）

* 输入：`input_pdf/`、`input_epub/`
* 处理：PDF 走 OCR/转换；EPUB 走解析/转换；统一产出 Markdown
* 输出：`output_markdown/`
* 对应 notebook：`pdf_to_md_2.ipynb`、`epub_to_md.ipynb`

(2) Markdown refine（清洗、去噪、为分段做准备）

* 输入：`output_markdown/`
* 处理：refine 清洗（常见目标：去掉无意义换行/页眉页脚/目录噪声；统一标题层级；把“视觉分页”改成“语义结构”）
* 输出：`cleaned_md/`
* 对应 notebook：`refine_md.ipynb`
* 
(3) cleaned Markdown → 报告笔记（专用 GPT + 分段输出）

* 输入：`cleaned_md/`
* 处理：将 cleaned Markdown 按“可控长度”的分段喂给专用 GPT；输出“简化笔记”
* 专用 GPT：`https://chatgpt.com/g/g-6958c62859308191a56c55be410caad7-bao-gao`
* 指令文件位置：仓库目录 `读书笔记/`（包含专用 GPT 的提示词/规则，尤其是分段与 Hook 机制）

## 为什么必须“预处理 + 分段输出”

LLM 在长文档场景的主要失败模式不是“理解不了”，而是“自动截断/丢上下文”，导致你得到的是一个看似完成、实则缺页的笔记。官方文档也明确建议在 token/上下文受限时把长文本拆分成更小片段处理。

同时，即便平台允许上传很大的文件，模型在一次生成里可用的上下文与输出仍然存在上限；“能上传”不等于“能一次性稳定产出完整笔记”。

因此本项目的设计取舍是：

* 在进入 LLM 之前，先把格式统一成 Markdown（减少 PDF 布局噪声，把结构显式化）。
* 把“长文”切成“可以被可靠处理”的段落序列（而不是赌一次性生成）。
* 通过 Hook 机制让分段输出具备“可续写、可校验、可回滚”的能力。

## Hook 机制（用于对抗截断，保证可续写）

分段输出要解决两个问题：一是“怎么知道写到哪了”，二是“怎么继续且不重写/不漏写”。这个项目把“续写控制信息”显式编码为 Hook，放进每段输出的末尾（具体 Hook 格式以 `读书笔记/` 内指令为准）。

实践上，Hook 一般至少包含三类字段（建议你在指令里固定下来）：

* progress：已完成的章节/标题锚点（最好能精确到某个二级/三级标题）
* next：下一段从哪里开始（标题名、原文片段首句、或行号范围三选一，越确定越好）
* constraints：续写时必须保持的输出 schema 与禁止项（防止格式漂移）

只要 Hook 是确定性的，后续就可以用“把 Hook + 下一段原文”发给专用 GPT 的方式迭代跑完全文。

## 目录结构

仓库目前的布局是按“输入—中间产物—输出”的思路组织，便于你在每一步做抽查与回放。

```
Report_note_generation/
  input_pdf/          # 原始 PDF（扫描版/文本版均可）
  input_epub/         # 原始 EPUB
  output_markdown/    # 初步转换出的 Markdown
  cleaned_md/         # refine 之后可供 LLM 稳定处理的 Markdown
  读书笔记/            # 专用 GPT 指令（含分段与 Hook 规则）
  pdf_to_md_2.ipynb
  epub_to_md.ipynb
  refine_md.ipynb
```

## 本地部署前置条件（OCR / 转换）

PDF（尤其是扫描版）要稳定变成可用 Markdown，OCR 环境是前置依赖。你这里采用 PaddleOCR / PaddlePaddle GPU 路线的话，有两个关键点：一是 GPU 环境与 CUDA/cuDNN 兼容，二是版本组合尽量固定，避免“能装但不稳定”。PaddlePaddle 官方安装文档明确提到 GPU 需要 CUDA、cuDNN 等依赖，并给出安装指引。

一个可复用的“固定版本组合”：

* Python 3.10（Python 官方提供 3.10 系列发布与下载）
* paddlepaddle-gpu 2.6.2（PyPI 版本页）
* paddleocr 2.7.3（PyPI 版本页）
* numpy 1.26.4（PyPI 版本页）
* OpenCV：`opencv-python` 的 PyPI 项目说明强调其默认 wheel 是 CPU-only，如需 CUDA 需要自己编译/定制构建；这点对“为什么 OCR 跑在 GPU 但 OpenCV 不一定上 GPU”很重要。

如果你希望 README 提供“最小可运行”的环境搭建命令，可以放一个参考（按需改成 conda/pip 任一套即可）：

```bash
# 例：conda 创建环境（你也可以用 venv）
conda create -n report_note python=3.10 -y
conda activate report_note

# 固定关键版本（按你的稳定组合）
python -m pip install -U pip
python -m pip install paddlepaddle-gpu==2.6.2 paddleocr==2.7.3 numpy==1.26.4

# 其他依赖（按 notebook 实际 import 补齐）
python -m pip install opencv-python
```

PaddlePaddle 安装完成后，用官方建议的方式做一次自检（`paddle.utils.run_check()`）可以快速排除“装了但不可用”的情况。

## 如何使用（按 notebook 顺序跑）

这个项目的使用方式是“先离线把文本变得可控，再把可控文本交给专用 GPT”。

第一步：把 PDF/EPUB 放入输入目录，然后分别跑转换 notebook。

* PDF：`input_pdf/` → 跑 `pdf_to_md_2.ipynb` → `output_markdown/`
* EPUB：`input_epub/` → 跑 `epub_to_md.ipynb` → `output_markdown/`

第二步：跑 `refine_md.ipynb`，把 `output_markdown/` 清洗到 `cleaned_md/`。

第三步：打开 `cleaned_md/` 的 Markdown，按 `读书笔记/` 里的指令把内容分段喂给专用 GPT，并严格依赖 Hook 迭代直到结束。

## 设计优点（你做对了什么）

(1) 把“不可控的 PDF 布局噪声”变成“可控的 Markdown 结构”，降低 LLM 在格式层面浪费的注意力。

(2) 把“长文一次性生成”改成“分段 + Hook 的确定性迭代”，对抗截断并让进度可追踪。

(3) 把每一步的产物落盘到目录（`output_markdown/`、`cleaned_md/`），使得抽查与回归测试变简单：任何异常都能定位在“转换阶段 / 清洗阶段 / LLM 阶段”。

如果你希望 README 更“贴合你实际 prompt 的 Hook 细节”（例如 Hook 的确切字段名、分段粒度规则、失败重试策略），把 `读书笔记/` 里那份指令文件用 raw 链接贴出来（或直接复制文本）我就能把 README 的 Hook 部分从“设计说明”升级成“可照抄执行的协议”。
