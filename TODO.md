# TODO — 概率论与数理统计学习助手

> 最后更新：2026-07-07 | 当前版本：v1.0

---

## 部署架构：本地 + ima 知识库 双轨并行

```
┌────────────────────────────────────────────────────┐
│                  SKILL.md (教学引擎)                 │
│                                                     │
│  出题流程 Step C-D：查找例题 + 展示配图              │
│          │                                          │
│          ├── 轨道 A（本地）：Grep textbook-examples  │
│          │      → present_files(本地 PNG)           │
│          │                                          │
│          └── 轨道 B（ima）：search_knowledge         │
│                 → fetch_media_content → present     │
│                                                     │
│  优先级：ima 可用则优先 ima，否则回退本地            │
└────────────────────────────────────────────────────┘
```

---

## 🔴 P0 — 阻塞项

### P0-1: 教材内容入库（本地 + ima 双轨）

**目标**：使 Skill 出题时有真实的例题和配图可用。支持本地文件系统和 ima 知识库两种后端，优先 ima，回退本地。

#### 子任务 P0-1A: 本地轨道

| 步骤 | 谁做 | 内容 |
|------|------|------|
| 1 | 用户 | 翻看 PDF，记录例题编号→页码映射表 |
| 2 | 脚本 | 批量渲染每页为 PNG，存入 `textbook_images/` |
| 3 | 脚本 | 填充 `textbook-examples.md`（每道题含图片文件名） |
| 4 | 脚本 | 运行 `grep` 验证所有引用的图片均存在 |

**预计状态**：❌ 未开始（等待映射表）

#### 子任务 P0-1B: ima 轨道

| 步骤 | 谁做 | 内容 |
|------|------|------|
| 1 | 用户 | 在 ima 客户端创建知识库（如"概率论教材"） |
| 2 | 用户 | 上传 `curriculum.md`、`textbook-examples.md`、`problem-solving-method.md` |
| 3 | 用户 | 上传 PDF 原文（ima 自动解析文本层） |
| 4 | 用户 | 上传教材配图，按章节分文件夹 |
| 5 | 脚本 | 获取 `knowledge_base_id`，写入 SKILL.md 配置 |
| 6 | 脚本 | 修改 SKILL.md 出题流程：添加 ima `search_knowledge` 优先路径 |

**预计状态**：❌ 未开始（等待用户创建知识库）

---

### P0-2: SKILL.md 出题流程双轨改造

**描述**：在当前 Grep 出题流程之前插入 ima 语义搜索优先路径。改动集中在 SKILL.md 的第九节（教材例题使用规范）。

**改动点**：
```
当前 Step 2: Grep 搜索 references/textbook-examples.md
         ↓
改造 Step 2: 先 try search_knowledge(ima)
            ├─ 命中 → 用 fetch_media_content 拿图片
            └─ 未命中/ima 不可用 → 回退 Grep 本地文件
```

**预计状态**：❌ 未开始（依赖 P0-1B 有 knowledge_base_id）

---

## 🟡 P1 — 重要增强

### P1-1: SKILL.md 图片路径预解析修正

**描述**：当前路径写的是 `circuit-interactive-tutor`，需改为 `probability-statistics-tutor`。同时适配 ima 模式下的路径跳过逻辑。

**预计状态**：❌ 未开始

---

### P1-2: ima 知识库扩展资料添加

**描述**：在上传基础教材文件后，逐步添加：
- 考研真题（按章节分类）
- 常见错题集
- 网页科普文章（贝叶斯公式、中心极限定理等）

**预计状态**：❌ 未开始（可渐进式积累）

---

### P1-3: curriculum.md 第 11-12 章微步骤细化

**描述**：当前 11-12 章为选学章节，知识点描述较简略。

**预计状态**：❌ 未开始

---

## 🟢 P2 — 优化改进

### P2-1: 部署文档（已完成 ✅）

**描述**：`DEPLOY.md` 已创建，包含双轨部署步骤、环境变量、分支策略、冲突处理。

**预计状态**：✅ 已完成

---

### P2-2: 添加 .gitattributes

**描述**：统一行尾规则，消除 Windows CRLF 警告。

**预计状态**：❌ 未开始

---

### P2-3: GitHub Release v1.0

**描述**：创建 Release 标签，附带版本说明。

**预计状态**：❌ 未开始

---

### P2-4: CONTRIBUTING.md

**描述**：贡献指南，说明章节格式、例题格式、图片命名规范等。

**预计状态**：❌ 未开始

---

## 完成记录

| 日期 | 完成项 | 说明 |
|------|--------|------|
| 2026-07-07 | SKILL.md | 1040 行，全功能教学系统 |
| 2026-07-07 | curriculum.md | 850 行，189 个知识点含微步骤 |
| 2026-07-07 | problem-solving-method.md | 297 行，STEP-Prove 双轨法 |
| 2026-07-07 | textbook-examples.md | 265 行，模板版 |
| 2026-07-07 | README.md | 项目说明、安装、使用、课程表 |
| 2026-07-07 | LICENSE | MIT |
| 2026-07-07 | .gitignore | PDF/backup/缓存排除 |
| 2026-07-07 | DEPLOY.md | 本地+ima 双轨部署方案 |
| 2026-07-07 | GitHub 仓库 | sqbsayori/probability-statistics-tutor |
