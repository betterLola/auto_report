# 天府市民云报表自动化系统 (Auto-Report)

[![Python Version](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

天府市民云工作日报、周报自动化生成系统。支持两种模式：**专项报告**（日报/周报，硬编码业务逻辑）和**通用模板引擎**（任意 Word 模板 + CSV 配置，无需写代码）。

数据获取基础见 https://github.com/betterLola/data-foundation

---

## 核心功能

- **日报自动化**：生成昨日运行情况 Word 报表，涵盖 DAU、核心服务人次、新增/累计注册用户。
- **周报自动化**：对比"本周期（上周五-本周四）"与"上周期"数据，生成带走势图、涨跌榜和搜索词分析的深度周报。
- **通用模板引擎**：在任意 Word 文档中写 `{{变量名}}`，配一个 CSV 表格指定数据库字段，即可自动生成报告，无需写 SQL 或代码。
- **数据自动回填**：智能检查近 7 天数据缺失，自动通过 API 或爬虫补全。

---

## 快速开始

### 1. 环境准备

```bash
cd auto_report
pip install -r requirements.txt
```

### 2. 配置 `.env`

在根目录创建 `.env` 文件：

```ini
# 数据库
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=您的密码
DB_NAME=daily

# 友盟 API（日报/周报专用）
UMENG_API_KEY=您的Key
UMENG_API_SECURITY=您的Secret

# 路径（可选，有默认值）
SERVICE_MAPPING_PATH=C:/path/to/是否为服务.xlsx
OUTPUT_DIR=C:/path/to/报表产出
```

---

## 使用指南

### 模式一：通用模板引擎（傻瓜模式）

适合业务人员自行新增报告，**无需写 SQL 或代码**，只需三步：

#### 第一步：制作 Word 模板

在 Word 文档中，把需要自动填入数据的位置换成 `{{变量名}}`：

```
{{report_date}} 工作日报

当日平台日活 {{platform_dau}}，环比 {{dau_growth}}。
累计注册用户 {{total_users}}，较昨日新增 {{new_users}}。
```

保存为 `.docx`，放入 `templates/` 目录。

#### 第二步：扫描模板，生成 CSV 骨架

```bash
python report_engine.py --scan templates/my_report.docx
```

自动识别所有占位符，生成 `templates/my_report.csv`。

#### 第三步：用 Excel 打开 CSV，填写数据库字段

| 占位符 | 数据库字段 | 格式 | 固定值 | 备注 |
|--------|-----------|------|--------|------|
| report_date | | 日期 | | 自动填昨日日期 |
| platform_dau | platform_daily_metrics.platform_dau | 万 | | 平台日活 |
| dau_growth | platform_daily_metrics.dau_growth | 百分比 | | 环比增幅 |
| total_users | platform_daily_metrics.total_register_users | 万 | | 累计注册 |
| new_users | platform_daily_metrics.new_register_users | 整数 | | 新增注册 |
| dept_name | | | 城运中心 | 固定文字 |

**列说明：**

| 列 | 填写规则 |
|----|---------|
| 数据库字段 | 填 `表名.字段名`（如 `platform_daily_metrics.platform_dau`）；或只填字段名，系统自动找表 |
| 格式 | `万` / `百分比` / `整数` / `日期` / `原始` |
| 固定值 | 填了直接用该文字，不查数据库；优先于数据库字段 |

保存 CSV，运行：

```bash
python report_engine.py
```

报告输出到 `报表产出/` 目录。

---

#### 其他引擎命令

```bash
# 生成单个报告
python report_engine.py --template templates/my_report.docx

# 列出所有可用模板及配置状态
python report_engine.py --list

# 批量生成模板目录下所有报告
python report_engine.py
```

---

### 模式二：专项报告脚本

#### 日报

```bash
python generate_daily_report.py
```

生成昨日（T-1）工作日报，输出到 `报表产出/` 目录。

#### 周报

```bash
# 步骤 1：同步搜索词数据（search_detail 表为空时执行）
python search_detail_import.py

# 步骤 2：生成周报
python weekly_report_generator.py
```

#### 历史数据补全

```bash
python data_backfilling.py
```

检查并自动修复近 7 天的数据缺口。

---

## 文件结构

```
auto_report/
├── report_engine.py              通用模板引擎（CSV 傻瓜模式 + JSON 高级模式）
├── generate_daily_report.py      日报专项脚本
├── weekly_report_generator.py    周报专项脚本
├── search_detail_import.py       搜索词数据同步
├── data_backfilling.py           历史数据补全
├── requirements.txt              Python 依赖
├── .env                          本地配置（不提交）
└── templates/                    模板目录
    ├── README.md                 模板使用说明
    ├── my_report.docx            Word 模板（用 {{变量名}} 标记占位符）
    ├── my_report.csv             CSV 配置（--scan 自动生成，用户填写字段）
    ├── daily_report.json         日报 JSON 配置示例
    └── weekly_report.json        周报 JSON 配置示例
```

---

## 统计规则说明

### 日报

- **周期**：昨日 00:00–24:00
- **核心服务**：仅统计"服务映射表"中标记为`是否为服务=1`的项目
- **增幅前五**：对比上周同期，昨日使用量 > 100 次的服务降序排列

### 周报

- **本周期**：上周五至本周四（共 7 天）
- **走势描述**：自动识别"震荡、单调上升/下降、先升后降、先降后升"五种趋势
- **各端备注**：有端口下降时，自动列出下降端均值及日均减少量

---

## 更新日志

### 2026-04-13

**工作日报增幅排序逻辑优化 (`generate_daily_report.py`)**
- **增幅排行门槛限制**：修改了“昨日增幅前五位服务”的计算规则。现在，**只有当日总服务次数大于 100** 的服务事项才会被纳入增幅排名和计算。此举有效过滤了因基数过小（如从 1 变 10）导致的高比例“虚假”涨幅，使报表更具业务参考价值。
- **数据库配置校对**：核对并确保了 `DB_CONFIG` 生产环境配置（`localhost:3306` / `daily` 库）的一致性。


### 2026-03-25

- **通用模板引擎升级**：新增 CSV 配置模式，业务人员无需写 SQL，只需在 Excel 里填写数据库字段名即可完成报告配置。
- **`--scan` 命令**：自动扫描 Word 模板中的所有 `{{占位符}}`，生成待填写的 CSV 骨架。
- **字段自动发现**：CSV 中只填字段名（不填表名）时，系统自动查询 `INFORMATION_SCHEMA` 定位所在表。
- **中文格式支持**：格式列支持中文填写（万/百分比/整数/日期/原始），降低使用门槛。
- **配置优先级**：`.csv` 优先于 `.json`，新旧配置格式完全兼容。

### 2026-03-20

- **周报话术升级**：优化日活上升/下降的自动描述逻辑，支持根据各端表现自动切换"增加/减少"措辞。
- **数值规范化**：统一所有报表中的"万"单位表述，保持四舍五入两位小数。

### 2026-03-16

- **新增自动回填模块**：实现 `data_backfilling.py`，支持近 7 天 Umeng DAU、核心服务、搜索词及爬虫数据的全自动化补全。
- **主任务调度优化**：调整 `main.py` 逻辑，在每日采集后自动触发回填与全库去重。

### 2026-03-13

- **search_detail_import.py**：修正表字段名 `service_amount` → `search_amount`，`service_name` → `search_name`。
- **周报智能分析**：引入 `build_platform_decline_text()`，自动识别全部上升/部分下降等场景并生成话术。
- **周报趋势可视化**：新增 `analyze_dau_trend()` 判定走势形态，`generate_dau_chart()` 生成本周期 vs 上周期对比折线图并嵌入 Word。
- **依赖更新**：新增 `matplotlib` 支持（含 SimHei 中文字体配置）。

### 2026-03-13

- **新增周报体系**：实现 `search_detail_import.py` 与 `weekly_report_generator.py`。

### 2026-03-11

- **日报格式严控**：重构 `generate_daily_report.py`，确保标题与副标题使用 `<w:br/>` 换行，完美契合城运中心红头文件格式。

### 2026-03-04

- **首次代码撰写**：基于 data-foundation 项目打造的数据库，产出 `generate_daily_report.py`。
