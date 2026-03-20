# 天府市民云报表自动化系统 (Auto-Report)

[![Python Version](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

天府市民云工作日报、周报自动化生成系统。通过 Umeng API 和 自动化爬虫获取多端（App、小程序、智能前端）数据，自动进行聚合计算，并生成符合城运中心格式要求的 Word 文档。

数据获取基础见https://github.com/betterLola/data-foundation

---

## 🚀 核心功能

-   **日报自动化**：每日清晨自动生成昨日运行情况 Word 报表，涵盖 DAU、核心服务人次、新增/累计注册。
-   **周报自动化**：每周五自动对比“本周期（上周五-本周四）”与“上周期”数据，生成带走势图、涨跌榜和搜索词分析的深度周报。
-   **数据自动回填**：智能检查近 7 天数据缺失情况，自动通过 API 或爬虫补全漏洞，确保报表连续性。
-   **多源数据整合**：打通友盟 API（App/小程序）、智慧前端、内网注册系统等多方数据源。

---

## 🛠️ 快速开始

### 1. 环境准备
确保您的环境为 Python 3.11+。

```bash
# 克隆项目 (或解压)
cd auto_report

# 安装依赖
pip install -r requirements.txt
```

### 2. 配置说明
项目使用 `.env` 文件管理敏感信息。请在根目录创建 `.env` 并填写：

```ini
# 数据库配置
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=您的密码
DB_NAME=daily

# 友盟 API 配置
UMENG_API_KEY=您的Key
UMENG_API_SECURITY=您的Secret

# 路径配置
SERVICE_MAPPING_PATH=C:/path/to/是否为服务.xlsx
OUTPUT_DIR=C:/path/to/报表产出
```

---

## 📖 使用指南

### 日报生成
生成昨日（T-1）的工作日报，输出到 `报表产出` 目录。
```bash
python generate_daily_report.py
```

### 周报生成
生成周报需要两个步骤：
1. **同步搜索数据**：
   ```bash
   python search_detail_import.py
   ```
2. **生成 Word 周报**：
   ```bash
   python weekly_report_generator.py
   ```

### 历史补全 (自动回填)
检查并修复近 7 天的数据缺口。
```bash
python data_backfilling.py
```

---

## 📂 文件结构说明

| 文件                           | 说明                                       |
| :--------------------------- | :--------------------------------------- |
| `generate_daily_report.py`   | **日报核心脚本**：数据提取、图表生成、Word 渲染             |
| `weekly_report_generator.py` | **周报核心脚本**：环比分析、趋势图、事项涨跌榜                |
| `data_backfilling.py`        | **补漏助手**：自动检查并回填近 7 天缺失的各端数据（见data-foundation项目） |
| `report_engine.py`           | **底层引擎**：通用的数据聚合逻辑与数据库交互                 |
| `templates/`                 | 存放报表生成的 JSON 模板及样例                       |

---

## 📊 统计规则概要

### 日报 (Daily)
-   **周期**：昨日 00:00 - 24:00。
-   **核心服务**：仅统计“服务映射表”中标记为“是否为服务=1”的项目。
-   **增幅前五**：对比上周同期，昨日使用量 > 100 次的服务进行降序排列。

### 周报 (Weekly)
-   **本周期**：上周五 至 本周四（共 7 天）。
-   **走势描述**：自动识别“震荡、单调上升/下降、先升后降、先降后升”五种趋势。
-   **各端备注**：当存在端口下降时，自动列出下降端的均值及日均减少量。

---

## 📝 更新日志

### 2026-03-20
-   **周报话术升级**：优化了日活上升/下降的自动描述逻辑，支持根据各端表现自动切换“增加/减少”措辞。
-   **数值规范化**：统一所有报表中的“万”单位表述，保持四舍五入两位小数。

### 2026-03-16
-   **新增自动回填模块**：实现 `data_backfilling.py`，支持近 7 天 Umeng DAU、核心服务、搜索词及爬虫数据的全自动化补全。
-   **主任务调度优化**：调整 `main.py` 逻辑，在每日采集后自动触发回填与全库去重。

### 2026-03-13
-   **search_detail_import.py**：修正表字段名 `service_amount` -> `search_amount`，`service_name` -> `search_name`，确保与数据库实际结构一致。
-   **weekly_report_generator.py (智能分析)**：引入 `build_platform_decline_text()`，自动识别全部上升/部分下降等场景并生成对应话术。
-   **weekly_report_generator.py (趋势可视化)**：新增 `analyze_dau_trend()` 判定走势形态，新增 `generate_dau_chart()` 生成本周期 vs 上周期对比折线图并嵌入 Word。
-   **依赖更新**：新增 `matplotlib` 支持（含 SimHei 中文字体配置）。

### 2026-03-13 
-   **新增周报体系**：实现 `search_detail_import.py`（逐日拉取搜索行为数据）与 `weekly_report_generator.py`（自动化周报核心引擎）。

### 2026-03-11
-   **日报格式严控**：重构 `generate_daily_report.py`，确保标题与副标题在同一段落内使用 `<w:br/>` 换行，完美契合城运中心红头文件格式。

### 2026-03-04
-   **首次代码撰写**：基于data-foundation项目打造的数据库，进行自动化报告生成，产出generate_daily_report.py。
