# /document-review — 通用文档审校命令

## 调用方式

```
/document-review <文件路径>
```

### 支持的文件类型

| 扩展名 | 处理方式 | 说明 |
|--------|---------|------|
| `.pdf` | 路由到 `/pdf-review` | 全量PDF学术审校（15 Agent） |
| `.ppt` / `.pptx` | 路由到 `ppt-quality-review` Skill | PPT内容审校（7 Phase） |
| `.md` | 轻量审校模式 | Markdown直接审校 |
| `.doc` / `.docx` | 轻量审校模式 | Word文档审校（需python-docx） |

### 不支持的类型

`.exe`, `.zip`, `.tar`, `.gz`, `.jpg`, `.png`, `.gif`, `.mp3`, `.mp4` 等二进制/媒体文件直接报错退出。

---

## 执行总流程

```
Step 0  文件验证 → 检查文件存在性、扩展名、非空
Step 1  类型路由 → 根据扩展名分发到对应引擎
Step 2  引擎执行 → PDF/PPT/MD/DOC各自审校
Step 3  统一输出 → 收集引擎输出，包装为统一结构
Step 4  生成确认包 → B类radio + C类引文译名 + D类checkbox
Step 5  等待确认 → Web 8230推送（降级：本地编辑）
Step 6  最终报告 → 综合总结
```

---

## Step 0 — 文件验证

```python
import os
SUPPORTED = {'.pdf', '.ppt', '.pptx', '.md', '.doc', '.docx'}
BLOCKED   = {'.exe', '.zip', '.tar', '.gz', '.jpg', '.png', '.gif', '.mp3', '.mp4'}

path = args  # 用户传入的文件路径
ext  = os.path.splitext(path)[1].lower()

# 不存在 → 报错退出
# 为空 → 报错退出
# ext 在 BLOCKED 中 → 报错退出
# ext 不在 SUPPORTED 中 → 报错退出，列出支持格式
```

---

## Step 1 — 类型路由

根据 `ext` 选择路由：

```python
ROUTES = {
    '.pdf':     'pdf',
    '.ppt':     'ppt',
    '.pptx':    'ppt',
    '.md':      'text',
    '.doc':     'text',
    '.docx':    'text',
}
route = ROUTES[ext]
```

---

## Step 2 — 引擎执行

### 路由A：.pdf → /pdf-review

1. **读取 `pdf-review` 命令**：打开并遵循 `.claude/commands/pdf-review.md`
2. **PDF解析（Step 1）**：PyMuPDF 提取文本、脚注、分块
3. **多Agent审校（Step 2）**：15 Agent 并发（文字/引文/译名/格式/学术/交叉验证）
4. **问题分类（Step 3）**：A类(A1-A6) + B类(B1-B2) + D类
5. **生成Markdown报告（Step 4）**：完整问题报告 + 综合总结 + 核查报告 + 修改决策清单
6. **生成人工确认包（Step 5）**：剩余问题总表 + B类确认 + D类确认 + A类未完成

**输出位置**（旧路径，与 `/pdf-review` 一致）：
```
output/PDF审校/<PDF文件名>/
```

**完成后** → 进入 Step 3（统一输出/复制）。

### 路由B：.ppt / .pptx → ppt-quality-review Skill

1. **读取 PPT Skill 指令**：打开 `.claude/skills/ppt-quality-review/SKILL.md` 获取完整流程
2. **执行 7 个 Phase**：
   - Phase 1 环境分析
   - Phase 2 内容分析（页面文本、标题、OCR、备注）
   - Phase 3 问题分类（A/B/C/D）
   - Phase 4 修改（A+B类）
   - Phase 5 PPT生成（修改版）
   - Phase 6 质量检查
   - Phase 7 归档

**输出位置**（旧路径，与 Skill 一致）：
```
output/PPT审校问题报告/
output/PPT审校修改报告/
output/PPT审校修改版/
output/PPT审校最终归档/
```

**完成后** → 进入 Step 3（统一输出/复制）。

### 路由C：.md / .doc / .docx → 轻量审校（内联定义）

轻量审校使用 3-5 个 Agent（文本审校 + 结构逻辑 + 一致性），不涉及 PDF 特有的脚注提取、引文格式校对、译名规范性检查（除非文件内容明确需要）。

#### C.1 文件解析

| 类型 | 解析方式 |
|------|---------|
| `.md` | 直接读取文件内容 |
| `.docx` | `python3 -c "import docx; doc = docx.Document(path); print('\\n\\n'.join([p.text for p in doc.paragraphs]))"` |
| `.doc` | 先用 LibreOffice 转换为 docx：`soffice --headless --convert-to docx`，再按 docx 解析 |

> ⚠️ `.docx` 解析需要 `python-docx` 库。如果未安装：
> ```
> pip3 install python-docx
> ```
> `.doc` 转 `docx` 需要 LibreOffice。检查：
> ```
> ls /Applications/LibreOffice.app/Contents/MacOS/soffice 2>/dev/null || which soffice 2>/dev/null
> ```

输出到：`output/document-review/<文件名>/md/`：

```
md/
├── 全文文本.txt       ← 提取的全部文本
├── 元数据.json        ← 行数、字数、段落数
└── 文本分块.json      ← 按合理长度切分（每块约5000字）
```

#### C.2 多Agent审校（3-5 Agent）

| Agent | 任务 | 审查内容 |
|-------|------|---------|
| Agent-1 | 文字审校 | 错别字、漏字、语法、标点、表达 |
| Agent-2 | 结构逻辑 | 段落结构、逻辑连贯性、标题层级 |
| Agent-3 | 一致性检查 | 术语一致、人名一致、格式一致 |
| Agent-4 | 学术细节（可选） | 如果文件包含引用、数据、专名 |

**每条发现JSON格式**（与PDF审校完全一致）：

```json
{
  "location": "第N段 / 第N行 / 标题名",
  "original": "原文内容",
  "suggestion": "修改建议",
  "reason": "修改原因",
  "type": "A | B | C | D",
  "category": "typo | punctuation | format | grammar | expression | citation | name | academic | structure"
}
```

**category 扩展**：轻量审校增加 `structure`（结构问题）类别。

#### C.3 问题分类

与PDF审校相同的 A/B/C/D 分类体系，外加轻量审校特有的子类：

| 类型 | 说明 |
|------|------|
| A类（确定错误） | 错字、漏字、标点、格式 |
| B类（建议优化） | 表达、结构、逻辑 |
| C类（引文/译名） | 引用格式、专有名词翻译 |
| D类（待确认） | 内容疑问、事实查证 |

#### C.4 生成Markdown报告

输出到 `output/document-review/<文件名>/reports/`：

```markdown
# 文档审校报告

## 基本信息

| 项目 | 值 |
|------|-----|
| 文件名 | ... |
| 格式 | MD / DOCX |
| 总字数 | ... |
| 总行数 | ... |

## 总体统计

| 分类 | 数量 |
|------|------|
| A类（确定错误） | ... |
| B类（建议优化） | ... |
| C类（引文/译名） | ... |
| D类（待确认） | ... |
| **合计** | ... |

## A类问题
...

## B类问题
...

## C类问题
...

## D类问题
...
```

格式细节与PDF审校的 `完整问题报告.md` 完全一致（同款表格、同款 markdown 结构）。

---

## Step 3 — 统一输出

在所有引擎完成后，创建统一输出目录结构。

### 3.1 目录创建

```bash
# 提取文件名（不含扩展名）
FILENAME=$(basename "$FILEPATH" | sed 's/\.[^.]*$//')

# 创建统一目录
mkdir -p "output/document-review/$FILENAME"
```

### 3.2 按路由复制/生成

```
output/document-review/<文件名>/
```

#### 路由PDF（从旧路径复制）

```bash
# PDF：从 output/PDF审校/<文件名>/ 复制到 pdf/
mkdir -p "output/document-review/$FILENAME/pdf"
cp -r "output/PDF审校/$FILENAME/"* "output/document-review/$FILENAME/pdf/" 2>/dev/null || true

# 生成 reports/ 中的汇总报告
mkdir -p "output/document-review/$FILENAME/reports"
# 从 pdf/ 中复制问题报告
cp "output/document-review/$FILENAME/pdf/问题报告/"* "output/document-review/$FILENAME/reports/" 2>/dev/null || true
```

#### 路由PPT（从旧路径复制）

```bash
# PPT：复制
mkdir -p "output/document-review/$FILENAME/ppt"
cp -r "output/PPT审校问题报告/"* "output/document-review/$FILENAME/ppt/问题报告/" 2>/dev/null || true
cp -r "output/PPT审校修改报告/"* "output/document-review/$FILENAME/ppt/修改报告/" 2>/dev/null || true
cp -r "output/PPT审校修改版/"* "output/document-review/$FILENAME/ppt/修改版/" 2>/dev/null || true
cp -r "output/PPT审校最终归档/"* "output/document-review/$FILENAME/ppt/归档/" 2>/dev/null || true
```

#### 路由MD/DOCX

输出已在 `md/` 和 `reports/` 中，无需额外复制。

### 3.3 生成文档审校总览

`output/document-review/<文件名>/reports/文档审校总览.md`：

```markdown
# 文档审校总览

## 基本信息

| 项目 | 内容 |
|------|------|
| 文件名 | ... |
| 格式 | PDF / PPT / MD |
| 审校模式 | 全量 / 轻量 |
| 审校日期 | ... |

## 审校统计

| 类型 | 数量 |
|------|------|
| A类（确定错误） | ... |
| B类（建议优化） | ... |
| C类（引文/译名） | ... |
| D类（待确认） | ... |
| **合计** | ... |

## 交付清单

| 文件 | 路径 |
|------|------|
| 完整问题报告 | reports/完整问题报告.md |
| B类确认包 | human-confirm/2-B类问题确认.md |
| C类确认包 | human-confirm/3-C类问题确认.md |
| D类确认包 | human-confirm/4-D类问题确认.md |
```

---

## Step 4 — 生成统一人工确认包

输出到 `output/document-review/<文件名>/human-confirm/`。

根据引擎输出（PDF/PPT/MD各引擎已有自己的问题列表），按统一格式生成4份确认文件。

### 4.1 生成策略

| 来源 | 映射到统一确认包 |
|------|----------------|
| PDF审校的 B类问题 → | human-confirm/2-B类问题确认.md |
| PDF审校的 D类问题 → | human-confirm/4-D类问题确认.md |
| PDF审校的 A类未完成 → | human-confirm/1-剩余问题总表.md |
| PDF审校的 引文/译名问题 → | human-confirm/3-C类问题确认.md |
| PPT审校的问题清单 → | 重新分类映射到 A/B/C/D |
| MD审校的问题清单 → | 直接按格式输出 |

### 4.2 文件1：剩余问题总表.md

```markdown
# 剩余问题总表

## 基本信息

| 项目 | 内容 |
|------|------|
| 文件 | ... |
| 问题总数 | ... |
| 其中A类确定错误 | ... |
| B类待人工确认 | ... |
| C类引文/译名 | ... |
| D类待学术确认 | ... |

## 一、A类确定错误（N条）

### A1 错字/漏字/拼写

| 编号 | 位置 | 原文 | 建议修改 | 风险 |
|:----:|:----:|------|----------|:----:|

...
```

### 4.3 文件2：B类问题确认.md（radio单选）

```markdown
# B类问题确认

> 共N条。请逐条确认是否建议修改，在 radio 选项中二选一。

---

### B1-01 第X页/段

- **问题**：...
- **建议修改**：...
- **修改理由**：...
- **是否建议修改**：[  ] 建议修改   [  ] 保持原文
- **确认意见**：___________________
```

### 4.4 文件3：C类问题确认.md（引文/译名专项）

```markdown
# C类引文/译名确认

> 共N条。请逐条确认引文/译名修改建议。

---

### C1-01 第X页

- **类型**：引文 | 译名
- **原文**：...
- **建议修改**：...
- **依据**：...
- **确认结果**：[  ] 确认采纳   [  ] 需作者确认   [  ] 保持原文
- **确认意见**：___________________
```

### 4.5 文件4：D类问题确认.md（checkbox多选）

```markdown
# D类问题确认

> 共N条。每条包含多个检查项，请逐项独立勾选确认。

---

### D1-01 第X页

- **问题**：...
- **原文**：...
- **需要确认**：
  - [  ] 检查项1
  - [  ] 检查项2
- **建议**：...
- **确认意见**：___________________
```

---

## Step 5 — 等待人工确认

### 5.1 Web 8230 推送

将确认包内容推送到确认系统：

```
POST http://localhost:8230/api/confirm
Content-Type: application/json

{
  "project": "<文件名>",
  "output_dir": "output/document-review/<文件名>/",
  "b_confirm": "2-B类问题确认.md 内容",
  "c_confirm": "3-C类问题确认.md 内容（引文/译名）",
  "d_confirm": "4-D类问题确认.md 内容",
  "status_url": "http://localhost:8230/api/status/<文件名>"
}
```

### 5.2 轮询确认状态

```
GET http://localhost:8230/api/status/<文件名>
```

轮询间隔：每30秒检查一次。轮询超时：30分钟。

响应状态：`pending` → `confirmed` | `timeout`

### 5.3 回写确认结果

Web 系统返回后，将确认结果追加写入 `human-confirm/` 对应文件。

### 5.4 降级方案（8230不可达）

```
curl --connect-timeout 3 http://localhost:8230/api/confirm ...
```

如果连接失败（3秒超时），切换到本地模式：

1. 输出确认包到 `human-confirm/`
2. 提示用户：

```
⚠️ Web确认系统（localhost:8230）不可用。
请直接在以下文件中编辑完成确认：
  output/document-review/<文件名>/human-confirm/
    ├── 2-B类问题确认.md    ← 在 [  ] 中填入 Y(建议修改) 或 N(保持原文)
    ├── 3-C类问题确认.md    ← 在 [  ] 中填入 Y(采纳) / A(作者确认) / N(保持)
    └── 4-D类问题确认.md    ← 在 [  ] 中填入 Y(确认)

完成后，再次运行：
  /document-review --apply-confirm output/document-review/<文件名>
```

---

## Step 6 — 最终报告

当人工确认完成后（Web 8230 返回或本地 `--apply-confirm`），更新：

```markdown
# 文档审校最终报告

## 审校统计

| 类型 | 总数 | 已确认 | 未确认 |
|------|:----:|:-----:|:------:|
| A类 | ... | ... | ... |
| B类 | ... | ... | ... |
| C类 | ... | ... | ... |
| D类 | ... | ... | ... |

## 处理建议

- 确认采纳的修改 → 直接应用
- 需作者确认的修改 → 转交作者
- 保持原文的条目 → 无需处理
```

输出到：`output/document-review/<文件名>/reports/最终报告.md`

---

## 异常处理

| 情况 | 处理方式 |
|------|---------|
| 文件不存在 | 报错退出，提示检查路径 |
| 不支持的文件类型 | 报错退出，列出支持的扩展名 |
| 空文件 | 报错退出，提示文件为空 |
| PDF无法解析 | 尝试OCR路径，如果OCR也无结果，提示扫描件需人工处理 |
| DOCX依赖缺失 | `pip3 install python-docx` 后重试 |
| DOC转换失败 | 提示需要LibreOffice |
| PPT LibreOffice缺失 | `brew install libreoffice` 或手动用PowerPoint打开 |
| Web 8230不可达 | 降级为本地确认模式 |
| Agent审校无发现问题 | 输出空报告并备注"未发现明显问题" |

---

## 多文件扩展（路由规则）

当检测到输入为多个文件路径时：

1. 对每个文件独立执行 Step 0 → Step 6
2. 每个文件独立目录：`output/document-review/<文件名1>/`、`output/document-review/<文件名2>/`
3. 文件之间完全隔离，不共享审校上下文
4. 生成汇总总览：`output/document-review/多文件审校总览.md`

```bash
# 多文件调用方式
/document-review "file1.pdf" "file2.pptx"
# 或
/document-review *.pdf
```

或通过 glob 扩展：
```
/document-review ./20260614/*.pdf
```

---

## 兼容性保证

| 项目 | 保证 |
|------|------|
| `/pdf-review` 命令 | 完全独立，`/document-review` 不修改其任何内容 |
| `ppt-quality-review` Skill | 完全独立，`/document-review` 不修改其任何内容 |
| `output/PDF审校/` 目录 | 保留，旧命令仍输出到此路径 |
| `output/PPT审校*/` 目录 | 保留，旧命令仍输出到此路径 |
| 旧命令直接调用 | `/pdf-review` 和 `/ppt-quality-review` 可绕过 `/document-review` 直接使用 |
| 问题JSON格式 | 统一使用 `{page, original, suggestion, reason, type, category}` 结构 |
