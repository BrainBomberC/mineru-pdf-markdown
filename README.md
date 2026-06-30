# MinerU PDF 转 Markdown 文档预处理流程

基于本地部署的 MinerU 视觉模型，将 PDF（书本、论文、扫描件）批量转换为结构化 Markdown 文件，为数据库建设、知识库构建和 RAG 检索系统提供高质量基础语料。

---

## 项目概述

本项目的核心目标不是简单的 OCR 文字提取，而是**尽可能保留 PDF 原始文档的结构信息**，包括标题层级、段落顺序、表格、公式和阅读顺序。传统 OCR 工具在转换过程中通常会破坏这些结构，导致后续数据入库质量下降。

本项目将 MinerU 视觉模型部署在 Linux 服务器上，通过 vLLM 提供 HTTP 推理服务，Windows 客户端安装 MinerU CLI 提交转换任务，最终输出结构化 Markdown 文件。

### 整体架构

```text
┌─────────────────┐         HTTP          ┌─────────────────────────┐
│  Windows 主机    │ ───────────────────→ │  Linux 服务器             │
│  MinerU CLI      │                       │  vLLM + MinerU 模型 + GPU │
│  (PDF 读取/提交) │ ←─────────────────── │  (视觉模型推理)           │
└─────────────────┘      返回 Markdown     └─────────────────────────┘
```

---

## 工作内容

本阶段主要完成了以下 8 项工作：

1. 在公司 Linux 服务器上部署 MinerU 视觉模型（`MinerU2.5-Pro-2604-1.2B`）
2. 使用 vLLM 将模型启动为本地 HTTP 推理服务
3. 配置服务器端 GPU 推理环境（CUDA、float16、显存管理）
4. 通过 nginx 对多个 vLLM 实例进行统一访问和负载分发
5. 在 Windows 主机上安装 MinerU CLI，通过终端命令远程调用服务器模型
6. 对书本、论文和其他 PDF 文件进行批量 Markdown 转换
7. 总结 PDF 转 Markdown 的质量边界和清洗规范
8. 编写本文档，为后续接手人员提供完整说明

---

## 环境与部署

### 服务器端（Linux）

**模型来源**：[opendatalab/MinerU2.5-Pro-2604-1.2B](https://huggingface.co/opendatalab/MinerU2.5-Pro-2604-1.2B)（HuggingFace）

**推理框架**：vLLM

**单 GPU 启动命令**：

```bash
CUDA_VISIBLE_DEVICES=2 \
VLLM_USE_FLASHINFER_SAMPLER=0 \
setsid .venv/bin/vllm serve "/path/to/model/snapshot" \
  --host 0.0.0.0 \
  --port 8000 \
  --served-model-name opendatalab/MinerU2.5-Pro-2604-1.2B \
  --logits-processors mineru_vl_utils:MinerULogitsProcessor \
  --dtype float16 \
  --max-model-len 8192 \
  --max-num-seqs 1 \
  --gpu-memory-utilization 0.85
```

**多 GPU + nginx 扩展**：

```text
                     ┌─ vLLM GPU0 :8000 ─┐
客户端 → nginx :80 ──┼─ vLLM GPU1 :8001 ─┼→ Markdown
                     └─ vLLM GPU2 :8002 ─┘
```

### 客户端（Windows）

**安装 MinerU CLI**：[https://github.com/opendatalab/MinerU](https://github.com/opendatalab/MinerU)

**验证安装**：

```powershell
mineru --help
mineru -v
```

**转换命令示例**：

```batch
mineru -p "document.pdf" ^
       -o "H:\output" ^
       -b vlm-http-client ^
       -u http://192.168.71.38:8000 ^
       -l ch
```

| 参数   | 含义                                                     |
| :----- | :------------------------------------------------------- |
| `-p` | 指定 PDF 文件路径                                        |
| `-o` | 指定 Markdown 输出目录                                   |
| `-b` | 后端模式（我们使用`vlm-http-client`，即远程 HTTP API） |
| `-u` | 服务器 MinerU 服务地址                                   |
| `-l` | 文档语言（`ch` 表示中文）                              |

---

## 后端模式说明

| 模式                        | 架构                                     | 适用场景                |
| :-------------------------- | :--------------------------------------- | :---------------------- |
| `vlm-http-client`（当前） | Windows CLI → HTTP → Linux vLLM + GPU  | 团队共享 GPU 服务器     |
| `vlm-local`               | Windows CLI + 本地模型 + 本地 GPU        | 单机使用，有自己的 GPU  |
| `vllm-engine`             | Windows CLI → vLLM 协议 → Linux Engine | 需要流式/高级 vLLM 特性 |

---

## MinerU 预处理的已知问题

MinerU 对大多数 PDF 能够完成转换，但以下问题在实际使用中**仍然存在**，需要在后续数据库入库前做人工校验或二次清洗：

### 1. 表格识别不完整

- **合并单元格**：复杂表格（跨行/跨列合并）可能被解析为独立单元格，逻辑关系丢失
- **跨页表格**：跨页的连续表格会被断开为两个独立表格，需手动拼接
- **无边框表格**：部分 PDF 中隐式分隔的表格可能被识别为普通段落

### 2. 数学公式解析

- **行内公式**：与中文混排时可能被截断或误识别为普通文字
- **多行公式**：`\begin{aligned}` 等环境可能丢失对齐结构
- **特殊符号**：部分 Unicode 数学符号可能转义异常

### 3. 图文混排与阅读顺序

- **图文环绕**：图片嵌入在文字中间时，输出可能打乱原始阅读顺序
- **图片标题**：图片下方的标题（如"图 1-1 xxx"）偶尔与正文段落合并
- **多栏论文**：双栏/三栏排版的文章，栏间阅读顺序偶尔出错

### 4. 页眉页脚与页码

- 页眉、页脚、页码通常会被当作正文保留在 Markdown 中
- 需要后续用正则或规则脚本批量清理（例如匹配连续出现的页眉文本模式）

### 5. 扫描件质量敏感

- **模糊/倾斜**：低分辨率扫描件或拍照件，识别准确率明显下降
- **水印/印章**：覆盖在文字上的水印可能干扰文字提取
- **手写批注**：文档边缘的手写笔记可能被误识别或干扰正文

### 6. 目录结构无法 100% 还原

- MinerU 输出的标题层级依赖于模型对版面的理解，并非读取 PDF 内部书签/TOC
- 部分非标准排版（如标题居中、特殊字体）的层级可能识别错误
- 建议：入库前对关键文档进行人工目录校验

### 7. 中英文混排

- 中英文混排段落的断行可能不自然
- 部分特殊字符（如全角半角混用）可能导致 token 解析异常

### 8. 大文件与性能

- 页数 > 200 的 PDF 转换时间较长（每页约 3-10 秒，视 GPU 型号而定）
- 单页超时可能中断整份文档的转换，需重试机制
- 多用户并发时需排队，受 `--max-num-seqs` 参数限制

---

## 后续清洗建议

| 问题类别      | 建议处理方式               | 优先级 |
| :------------ | :------------------------- | :----- |
| 页眉页脚/页码 | 正则批量过滤               | 高     |
| 跨页表格拼接  | 人工校验 + 规则拼接        | 高     |
| 公式校对      | 关键文档人工复核           | 中     |
| 目录层级校验  | 抽样对比原 PDF 目录        | 中     |
| 图片标题归属  | 规则匹配（"图 X-X" 模式）  | 低     |
| 扫描件后处理  | 提高源文件质量优先于后处理 | 低     |

---

## 快速上手指南（接手指南）

### 你需要准备

1. **服务器访问**：SSH 到部署了 MinerU 模型的 Linux 服务器
2. **GPU 确认**：`nvidia-smi` 确认 GPU 状态和可用显存
3. **Windows 主机**：已安装 MinerU CLI，可 ping 通服务器 `192.168.71.38`

### 日常使用流程

```text
1. SSH 登录服务器 → 检查 vLLM 服务是否正常运行
2. Windows 终端执行 mineru 命令 → 提交 PDF 转换
3. 等待转换完成 → 检查输出目录下的 .md 文件
4. 对 Markdown 做必要清洗（页眉页脚、表格拼接等）
5. 导入数据库 / 知识库
```

### 常见排查

| 现象                | 可能原因           | 处理                                |
| :------------------ | :----------------- | :---------------------------------- |
| `mineru` 连接超时 | 服务器 vLLM 未启动 | SSH 检查`ps aux \| grep vllm`      |
| 转换结果为空        | 模型加载失败       | 查看 vLLM 日志                      |
| GPU 显存不足        | 其他进程占用       | `nvidia-smi` 查看，必要时重启实例 |
| 单页报错            | 页面图片异常       | 跳过该页重试，或检查原 PDF 是否损坏 |

---

## 相关链接

- MinerU 模型：[HuggingFace - MinerU2.5-Pro-2604-1.2B](https://huggingface.co/opendatalab/MinerU2.5-Pro-2604-1.2B)
- MinerU CLI：[GitHub - opendatalab/MinerU](https://github.com/opendatalab/MinerU)
- vLLM：[GitHub - vllm-project/vllm](https://github.com/vllm-project/vllm)

---

## License

本项目文档仅供内部交接参考。
