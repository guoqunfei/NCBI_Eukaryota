# NCBI Eukaryota Genome Assembly Summary Pipeline

> 自动化真核生物基因组 Assembly 汇总统计 Pipeline，每日定时从 NCBI 下载最新基因组数据并生成结构化 Excel 报告。

---

## 项目概述

本项目通过整合 **NCBI Taxonomy**、**NCBI Datasets** 和 **TaxonKit** 等工具，实现真核生物（Eukaryota）基因组 Assembly 数据的自动化采集、分类学注释和格式化汇总。支持本地手动执行和 GitHub Actions 云端自动运行两种模式。

### 核心功能

| 功能 | 说明 |
|------|------|
| 分类学树构建 | 基于 NCBI Taxonomy 数据库，提取 Eukaryota（TaxID: 2759）下所有物种的分类谱系 |
| 基因组数据采集 | 通过 NCBI Datasets CLI 并行下载多个类群的基因组 Assembly 摘要 |
| 数据清洗与去重 | 自动识别单倍型（haplotype），按物种去重，保留最优 Assembly |
| 结构化 Excel 输出 | 生成带格式的双 Sheet Excel 文件，包含原始数据和物种去重数据 |
| 宏观分类统计 | 按门（Phylum）/纲（Class）汇总物种数、科数、属数等统计指标 |
| 自动化运行 | GitHub Actions 每日定时执行，支持邮件通知和结果自动归档 |

---

## 技术栈

- **Shell (Bash)**：Pipeline 主控脚本
- **Python 3**：数据处理、Excel 生成、超时下载 wrapper
- **TaxonKit**：NCBI Taxonomy 数据解析与分类谱系构建
- **NCBI Datasets CLI**：基因组 Assembly 数据查询与下载
- **jq**：JSON 数据解析与 TSV 转换
- **GitHub Actions**：CI/CD 自动化调度

### Python 依赖

```
pandas >= 1.5
openpyxl >= 3.0
```

### 系统依赖

```
taxonkit >= 0.20.0
ncbi-datasets-cli >= 16.38.1
jq >= 1.6
wget / curl
```

---

## 项目结构

```
NCBI_Eukaryota/
├── .github/
│   └── workflows/
│       ├── ncbi-daily.yml       # 主工作流：每日定时执行 pipeline
│       └── main.yml             # 辅助工作流（可选）
│
├── run_pipeline_final.sh        # 核心 Pipeline 脚本（主入口）
├── DEPLOY_GUIDE.md             # GitHub Actions 部署与配置指南
│
├── Protists.list               # 原生生物门（Phylum）列表，用于分类归类
├── Viridiplantae.list          # 植物界门列表，用于分类归类
│
├── fetch_with_timeout.py       # Python wrapper：带超时保护的 datasets 下载工具
├── get_Eukaryota_Taxonomy_Summary.py   # 宏观分类统计报表生成
├── generate_excel.py           # 格式化 Excel 生成器（双 Sheet）
│
├── auto_push.sh                # 自动推送脚本（辅助工具）
└── push_workflow.sh            # Git 工作流推送辅助脚本
```

### 运行时生成的文件

```
pipeline_YYYYMMDD_HHMMSS.log   # 运行日志
Eukaryota_nodes.txt            # Eukaryota 下所有节点的 TaxID 列表
Eukaryota_species_taxids.txt   # 仅物种/亚物种级别的 TaxID
Eukaryota_taxonomy.tsv         # 物种分类谱系表（界/门/纲/目/科/属/种）
Eukaryota_assemblies.jsonl     # NCBI 原始 JSON Lines 数据（合并后）
Eukaryota_assemblies_stats.tsv # Assembly 统计原始 TSV
Eukaryota_tax_class.txt        # 分类映射中间文件

# 最终输出
Eukaryota_assemblies_summary.xlsx        # 基因组 Assembly 汇总（双 Sheet）
NCBI_Eukaryota_Taxonomy_Summary.xlsx    # 门/纲宏观统计表
NCBI_Eukaryota_Taxonomy_Summary.txt      # 同上，制表符分隔格式
```

---

## Pipeline 流程详解

```
┌─────────────────────────────────────────────────────────────────┐
│  0. 环境检查                                                     │
│     └── 验证依赖：taxonkit, datasets, jq, python3, pandas       │
├─────────────────────────────────────────────────────────────────┤
│  1. 生成辅助脚本                                                 │
│     ├── fetch_with_timeout.py    (超时保护下载)                 │
│     ├── get_Eukaryota_Taxonomy_Summary.py  (宏观统计)            │
│     └── generate_excel.py         (Excel 生成)                  │
├─────────────────────────────────────────────────────────────────┤
│  2. 获取 Eukaryota 物种 TaxID                                    │
│     └── taxonkit list --ids 2759  (提取真核生物全部子类群)      │
├─────────────────────────────────────────────────────────────────┤
│  3. 提取物种级 TaxID                                             │
│     └── 过滤 [species] 和 [subspecies] 节点                      │
├─────────────────────────────────────────────────────────────────┤
│  4. 构建分类 lineage                                             │
│     └── taxonkit lineage → reformat  (生成 界/门/纲/目/科/属/种)│
├─────────────────────────────────────────────────────────────────┤
│  5. 分类汇总                                                     │
│     └── 按门/纲统计物种数、科数、属数，生成宏观报表              │
├─────────────────────────────────────────────────────────────────┤
│  6. 并行下载 NCBI 基因组摘要                                     │
│     └── 覆盖 10 个主要类群，每个 300s 超时，自动跳过已存在文件   │
├─────────────────────────────────────────────────────────────────┤
│  7. 合并 JSONL                                                   │
│     └── 合并多个类群数据为单一 Eukaryota_assemblies.jsonl       │
├─────────────────────────────────────────────────────────────────┤
│  8. 提取 Assembly 统计                                           │
│     └── jq 解析 JSON → TSV (Assembly Accession, 基因组大小等)   │
├─────────────────────────────────────────────────────────────────┤
│  9. 生成格式化 Excel（双 Sheet）                                  │
│     ├── Sheet 1: All_Raw          (全部 Assembly)               │
│     └── Sheet 2: Species_Unique   (按物种去重后，保留最优 Assembly)│
└─────────────────────────────────────────────────────────────────┘
```

---

## 本地运行

### 前置依赖安装

```bash
# 1. 安装 taxonkit
wget https://github.com/shenwei356/taxonkit/releases/download/v0.20.0/taxonkit_linux_amd64.tar.gz
tar -xzf taxonkit_linux_amd64.tar.gz
sudo mv taxonkit /usr/local/bin/

# 2. 安装 NCBI datasets CLI
wget https://ftp.ncbi.nlm.nih.gov/pub/datasets/command-line/LATEST/linux-amd64/datasets
sudo chmod +x datasets
sudo mv datasets /usr/local/bin/

# 3. 安装 Python 依赖
pip install pandas openpyxl

# 4. 安装系统依赖 (Ubuntu/Debian)
sudo apt-get install jq wget curl
```

### 下载 NCBI Taxonomy 数据库

```bash
mkdir -p ~/.taxonkit
cd ~/.taxonkit
wget https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/taxdump.tar.gz
tar -xzf taxdump.tar.gz
```

### 运行 Pipeline

```bash
bash run_pipeline_final.sh
```

运行完成后，当前目录会生成：
- `Eukaryota_assemblies_summary.xlsx` — 基因组 Assembly 汇总表
- `NCBI_Eukaryota_Taxonomy_Summary.xlsx` — 宏观分类统计表
- `pipeline_YYYYMMDD_HHMMSS.log` — 运行日志

---

## GitHub Actions 自动运行

项目已配置 `.github/workflows/ncbi-daily.yml`，支持全自动云端运行。

### 触发方式

| 触发方式 | 说明 |
|----------|------|
| 定时触发 | 每天 UTC 02:00（北京时间 10:00）自动执行 |
| Push 触发 | 推送到 `main` 分支时自动执行 |
| 手动触发 | 通过 `workflow_dispatch` 在 Actions 页面手动触发 |

### 配置 GitHub Secrets（邮件通知）

在仓库 **Settings → Secrets and variables → Actions** 中添加：

| Secret 名称 | 说明 | 示例 |
|-------------|------|------|
| `SMTP_SERVER` | SMTP 服务器 | `smtp.gmail.com` |
| `SMTP_PORT` | SMTP 端口 | `587` |
| `SMTP_USER` | SMTP 用户名 | `yourname@gmail.com` |
| `SMTP_PASSWORD` | SMTP 密码/应用密码 | `xxxxxxxxxxxxxxxx` |
| `EMAIL_TO` | 收件人地址 | `user@example.com` |
| `EMAIL_FROM` | 发件人地址 | `yourname@gmail.com` |

> **Gmail 用户注意**：需使用[应用专用密码](https://myaccount.google.com/apppasswords)，而非登录密码。

### 运行结果

Pipeline 成功后：
1. **Artifact 上传**：运行结果保留 90 天
2. **自动 Git 提交**：结果文件自动提交到 `results/YYYY-MM-DD/` 目录
3. **邮件通知**：发送成功/失败邮件，附带结果附件和运行日志链接

---

## 输出文件说明

### Eukaryota_assemblies_summary.xlsx

| Sheet | 说明 | 行数 |
|-------|------|------|
| All_Raw | 所有 Assembly 记录（含重复单倍型） | ~全部 |
| Species_Unique | 按物种去重后（每个物种保留 1 条最优 Assembly） | ~去重后 |

**关键字段**：
- `Assembly Accession` — Assembly 登录号
- `Species` / `Taxonomic ID` — 物种名和 TaxID
- `Subkingdom` / `Phylum` / `Class` / `Order` / `Family` / `Genus` — 分类层级
- `Assembly Level` — 基因组完成度 (Complete / Chromosome / Scaffold / Contig)
- `RefSeq Category` — RefSeq 分类 (Reference Genome / Representative Genome)
- `Haplotype` — 单倍型推断 (hap1/hap2/mat/pat/alt/pri/unphased)
- `Assembly Stats Total Sequence Length` — 总序列长度 (bp)
- `Assembly Stats Contig N50` / `Scaffold N50` — 连续性指标
- `Assembly Stats GC Percent` — GC 含量
- `Assembly Release Date` — 发布日期
- `Assembly Sequencing Tech` — 测序技术

### NCBI_Eukaryota_Taxonomy_Summary.xlsx

按 **门(Phylum)** 和 **纲(Class)** 汇总的宏观统计表：
- 总目数、总科数、总属数、总种数
- 按总种数降序排列

---

## 技术亮点

### 1. 超时保护与进程管理

`fetch_with_timeout.py` 使用 `Popen + killpg` 实现下载超时自动终止，避免 datasets CLI 因网络问题导致 CI 超时：

```python
proc = subprocess.Popen(cmd, stdout=f, stderr=subprocess.PIPE, start_new_session=True)
# ...
os.killpg(os.getpgid(proc.pid), signal.SIGKILL)
```

### 2. 断点续跑

Pipeline 支持增量运行，已下载的 `.jsonl` 文件会自动跳过，避免重复下载：

```bash
if [[ -s "$outfile" ]]; then
    log "  跳过 ${taxid}（已存在）"
    continue
fi
```

### 3. 单倍型识别与去重

`generate_excel.py` 自动识别 Assembly Name 中的单倍型标记（hap1/hap2/mat/pat/alt/pri），并按物种去重，优先保留 Reference Genome 和 N50 最高的记录：

```python
raw['__is_ref'] = (raw['RefSeq Category'].str.strip().str.lower() == 'reference genome').astype(int)
raw['__n50'] = pd.to_numeric(raw['Assembly Stats Contig N50 (bp)'], errors='coerce').fillna(-1)
raw['__key'] = raw['Taxonomic ID'] + '|' + raw['Haplotype']
rmdup = raw.sort_values(by=['__is_ref', '__n50'], ascending=[False, False]) \
           .drop_duplicates(subset='__key', keep='first')
```

### 4. TaxonKit 数据库缓存

GitHub Actions 使用 `actions/cache@v4` 缓存 `~/.taxonkit` 目录（约 1GB），避免每次运行都重新下载 NCBI Taxonomy 数据库。

---

## 常见问题

| 问题 | 解决方案 |
|------|----------|
| `taxonkit: xopen: no content` | 检查 `~/.taxonkit/` 下 `names.dmp`、`nodes.dmp` 等文件是否完整，尝试重新下载 taxdump |
| `datasets` 下载超时 | 正常现象，脚本内置 300s 超时保护，超时后会自动跳过并继续执行 |
| Excel 未生成 | 检查 `pandas` 和 `openpyxl` 是否安装成功 |
| 邮件未收到 | 检查 GitHub Secrets 配置是否正确，SMTP 端口是否为 587 |

---

## 作者

- **Maintainer**: [guoqunfei](https://github.com/guoqunfei)
- **License**: MIT

---

## 相关链接

- [NCBI Datasets Documentation](https://www.ncbi.nlm.nih.gov/datasets/docs/v2/)
- [TaxonKit GitHub](https://github.com/shenwei356/taxonkit)
- [NCBI Taxonomy FTP](https://ftp.ncbi.nlm.nih.gov/pub/taxonomy/)
