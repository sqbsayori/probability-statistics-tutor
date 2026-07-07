# 部署方案 — 本地 + ima 知识库 双轨并行

> 版本：v1.0 | 日期：2026-07-07

---

## 一、架构总览

```
┌───────────────────────────────────────────────────────┐
│                    SKILL.md                           │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │           出题流程（第九节）                      │ │
│  │                                                   │ │
│  │  Step 1: 确定知识点 kp_X.Y.Z                     │ │
│  │  Step 2: 查找例题                                │ │
│  │    ├─ [优先] search_knowledge(ima) ← 轨道 B      │ │
│  │    │    └─ 命中 → fetch_media_content → present  │ │
│  │    └─ [回退] Grep textbook-examples.md ← 轨道 A  │ │
│  │         └─ 命中 → present_files(local PNG)       │ │
│  │  Step 3: 展示配图 + 出题                         │ │
│  └─────────────────────────────────────────────────┘ │
│                                                       │
│  环境变量（SKILL.md 头部注释区）:                     │
│    IMA_KB_ID = "xxx"        # ima 知识库 ID          │
│    LOCAL_IMG_DIR = "./references/textbook_images"     │
│    SEARCH_MODE = "ima-first" # ima-first | local-only │
└───────────────────────────────────────────────────────┘
```

---

## 二、分支策略

```
master (v1.0 稳定版)
  ├── feat/ima-integration     ← ima 双轨改造开发分支
  │     ├── 修改 SKILL.md 出题流程
  │     ├── 添加 IMA_KB_ID 配置
  │     └── 测试 → MR → merge to master
  │
  └── feat/textbook-content    ← 教材内容填充分支
        ├── P0-1A: 本地图片提取
        ├── P0-1B: ima 知识库上传
        └── 测试 → MR → merge to master
```

| 分支 | 用途 | 合并策略 |
|------|------|---------|
| `master` | 稳定发布版 | 仅通过 MR 合入 |
| `feat/ima-integration` | ima 检索逻辑改造 | squash merge |
| `feat/textbook-content` | 例题+图片填充 | squash merge |

---

## 三、轨道 A：本地部署

### 3.1 前置条件

- 项目已 `git clone` 到本地
- Python 3.11+ 已安装 `pymupdf`

### 3.2 步骤

**Step A1 — 获取例题映射表**（人工，约 30 分钟）

翻看 `概率论与数理统计.pdf`，按以下格式记录：

```
例1-1 → 第3页
例1-4 → 第8页
例1-9 → 第15页
例2-4 → 第42页
...
```

保存为 `references/mapping.txt`。

**Step A2 — 批量渲染图片**（脚本，约 5 分钟）

```python
# scripts/extract_images.py
import fitz, json

with open("references/mapping.txt") as f:
    # 解析映射表
    ...

doc = fitz.open("概率论与数理统计.pdf")
for example_id, page_num in mapping.items():
    pix = doc[page_num - 1].get_pixmap(dpi=150)
    pix.save(f"references/textbook_images/{example_id}.png")
doc.close()
```

**Step A3 — 填充 textbook-examples.md**（脚本，约 5 分钟）

将映射表转化为标准例题格式，确保每道例题的"配套图片"字段正确指向生成的 PNG。

**Step A4 — 验证完整性**（脚本，约 1 分钟）

```bash
# 检查所有引用的图片是否存在
grep -oP 'ch\d{2}_image\d+\.png' references/textbook-examples.md \
  | sort -u | while read img; do
    [ -f "references/textbook_images/$img" ] || echo "❌ 缺失: $img"
  done
```

### 3.3 本地模式配置

SKILL.md 中无需额外配置——Grep + present_files 默认走本地路径。

---

## 四、轨道 B：ima 知识库部署

### 4.1 前置条件

- ima 客户端已登录
- 已创建知识库（名称随意，如"概率论教材资源"）

### 4.2 步骤

**Step B1 — 获取 knowledge_base_id**（在 Skill 启动时自动获取）

```
调用 get_knowledge_base_list → 找到名为"概率论教材资源"的知识库 → 记录其 ID
```

将此 ID 记录为环境变量 `IMA_KB_ID`，写入 SKILL.md 头部。

**Step B2 — 上传教学内容**（在 ima 客户端操作）

| 文件 | 上传为 | 用途 |
|------|--------|------|
| `curriculum.md` | MARKDOWN | 知识点体系，按章节语义搜索 |
| `textbook-examples.md` | MARKDOWN | 例题题库，按知识点匹配 |
| `problem-solving-method.md` | MARKDOWN | 解题方法论，学生可检索 |
| `概率论与数理统计.pdf` | PDF | 教材全文，ima 自动解析文本层 |
| `textbook_images/*.png` | IMG | 按章节分文件夹，如 `ch01/`、`ch02/` |

**Step B3 — 修改 SKILL.md 出题流程**

在第九节 Step 2 之前插入 ima 优先检索逻辑：

```
Step 2: 查找例题
  a) [ima 模式] 调用 search_knowledge(
       knowledge_base_id=IMA_KB_ID,
       query="<当前知识点名称 + 关键词>",
       filters={media_type: ["MARKDOWN", "PDF"]}
     )
     → 若返回结果，取最匹配的例题条目
     → 若例题有配图，调用 fetch_media_content 获取
     → present_files 展示
  b) [回退模式] 若 ima 不可用或无结果：
     → 回退到 Grep textbook-examples.md
     → present_files 展示本地 PNG
```

### 4.3 ima 模式配置

在 SKILL.md 前导注释区声明：

```yaml
# === 环境配置（部署时填写） ===
# IMA_KB_ID: "xxxxxxxx"           # ima 知识库 ID（轨道 B）
# LOCAL_IMG_DIR: "references/textbook_images"  # 本地图片目录（轨道 A）
# SEARCH_MODE: "ima-first"        # ima-first | local-only
```

---

## 五、并行部署冲突处理

### 5.1 图片版本不一致

**场景**：本地 `textbook_images/` 和 ima 知识库中的同一张图内容不同（如 PDF 重新扫描、清晰度不同）。

**处理**：
1. **ima 优先**：Skill 总是先查 ima，返回 ima 中的版本
2. 本地图片仅作为回退，不主动与 ima 对比
3. 更新策略：修改教材图片时，**同时更新两边**，确保最终一致

### 5.2 例题内容不一致

**场景**：`textbook-examples.md` 在本地仓库和 ima 知识库中有不同版本（如修正了某道题的解答）。

**处理**：
1. 本地 `textbook-examples.md` 为**源码**（Git 管理）
2. ima 知识库为**快照**（手动上传同步）
3. 每次修改本地 `.md` 文件后，**commit + push**，然后在 ima 客户端重新上传更新后的文件
4. 同步检查清单：每完成一轮修改后，核实 ima 知识库中的文件日期 ≥ Git 最新 commit 日期

### 5.3 知识库 ID 变更

**场景**：ima 知识库被删除重建，ID 变化。

**处理**：
1. SKILL.md 中的 `IMA_KB_ID` 应支持运行时动态检测（`get_knowledge_base_list` 按名称查找）
2. 不硬编码 ID，而是记录"知识库名称"作为查找依据
3. 首次启动时自动解析名称→ID，缓存到会话上下文中

### 5.4 两轨搜索结果合并

**场景**：ima 返回了 A 例题，本地 Grep 返回了 B 例题，两者不同。

**处理**：
1. 始终以 ima 结果为准（ima 优先策略）
2. 不合并两轨结果——合并可能导致出题重复或冲突
3. 仅在 ima 完全不可用时才启用本地轨道

---

## 六、环境变量速查

| 变量 | 作用 | 示例值 | 来源 |
|------|------|--------|------|
| `IMA_KB_ID` | ima 知识库 ID | `"kb_abc123"` | `get_knowledge_base_list` 动态获取 |
| `IMA_KB_NAME` | 知识库名称（用于查找 ID） | `"概率论教材资源"` | 用户创建时命名 |
| `LOCAL_IMG_DIR` | 本地图片路径 | `"references/textbook_images"` | 项目固定路径 |
| `SEARCH_MODE` | 检索模式 | `"ima-first"` 或 `"local-only"` | SKILL.md 配置 |
| `SKILL_NAME` | skill 标识 | `"probability-statistics-tutor"` | SKILL.md 元数据 |

---

## 七、部署检查清单

### 本地轨道验证

- [ ] `textbook_images/` 下 PNG 文件数量 > 0
- [ ] `grep` 验证所有 `textbook-examples.md` 引用的图片均存在
- [ ] `present_files` 可正常展示一张教材图片

### ima 轨道验证

- [ ] `get_knowledge_base_list` 可返回知识库列表
- [ ] `search_knowledge(query="贝叶斯公式", kb_id=xxx)` 返回相关结果
- [ ] `get_knowledge_list(kb_id=xxx)` 可列出知识库内容
- [ ] `fetch_media_content(media_id=xxx)` 可获取图片

### 双轨切换验证

- [ ] ima 可用时 → 走 ima 路径，不触发 Grep
- [ ] ima 不可用时 → 自动回退本地 Grep，不报错
- [ ] `SEARCH_MODE=local-only` 时 → 强制走本地，不尝试 ima

---

## 八、首次部署顺序

```
1. git clone → 本地仓库就绪
2. 执行 轨道A Step A1-A4 → 本地图片+例题就绪
3. 在 ima 客户端创建知识库 → 上传教材文件
4. 运行 get_knowledge_base_list → 获取 IMA_KB_ID
5. 修改 SKILL.md → 写入 IMA_KB_ID + 双轨检索逻辑
6. git commit + push → 代码层部署完成
7. 重启 WorkBuddy → 加载新版 SKILL.md
8. 验证：启动 Skill → 选第1章 → 确认例题可正常展示
```

---

## 九、回滚方案

| 场景 | 回滚操作 |
|------|---------|
| ima 知识库不可用 | 设置 `SEARCH_MODE=local-only`，仅使用本地轨道 |
| 本地图片损坏/缺失 | 设置 `SEARCH_MODE=local-only` 跳过（ima 优先时无需回退），或重新执行 Step A2 |
| SKILL.md 修改出错 | `git revert` 到上一个稳定 commit |
| 双轨都不可用 | P0 阻塞——需立即修复至少一条轨道 |
