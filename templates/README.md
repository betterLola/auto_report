# 报告模板目录使用说明

## 两种使用方式

### 方式一：傻瓜模式（推荐，无需写 SQL）

1. 把 Word 模板（`.docx`）放入本目录，在需要填数据的位置写 `{{变量名}}`
2. 运行扫描命令，自动生成 CSV 配置骨架：
   ```bash
   python report_engine.py --scan templates/my_report.docx
   ```
3. 用 **Excel** 打开生成的 `my_report.csv`，在 `数据库字段` 列填写字段名：

   | 占位符 | 数据库字段 | 格式 | 固定值 | 备注 |
   |--------|-----------|------|--------|------|
   | platform_dau | platform_daily_metrics.platform_dau | 万 | | 平台日活 |
   | dau_growth | platform_daily_metrics.dau_growth | 百分比 | | 日活增幅 |
   | report_date | | 日期 | | 自动填昨日日期 |
   | custom_text | | | 这是固定文字 | 固定内容 |

4. 保存 CSV，运行引擎：
   ```bash
   python report_engine.py
   ```

---

### 方式二：高级模式（JSON 配置，支持自定义 SQL）

与模板同名，扩展名改为 `.json`：

```json
{
  "report_name": "我的报告",
  "output_dir": "报表产出",
  "output_filename": "我的报告_{date}.docx",
  "variables": {
    "platform_dau": {
      "type": "sql",
      "query": "SELECT platform_dau FROM platform_daily_metrics WHERE stat_date = '{yesterday}'",
      "format": "wan"
    },
    "custom_text": {
      "type": "literal",
      "value": "固定文字内容"
    }
  }
}
```

---

## CSV 列说明

| 列名 | 说明 |
|------|------|
| 占位符 | 对应模板里的 `{{变量名}}` |
| 数据库字段 | `表名.字段名`（如 `platform_daily_metrics.platform_dau`）；或只填字段名（系统自动找表） |
| 格式 | 见下表 |
| 固定值 | 若填写则直接使用该文字，不查数据库 |
| 备注 | 仅供人工阅读，引擎忽略 |

## 格式选项

| 格式 | 效果 |
|------|------|
| `万` | `123456` → `12.35万` |
| `百分比` | `0.0523` → `+5.23%` |
| `整数` | `123456` → `123,456` |
| `日期` | `2026-03-18` → `3月18日` |
| `原始` | 原始值（默认） |

> 也支持英文格式名：`wan` / `pct` / `int` / `date_cn` / `raw`

## 内置日期变量

以下占位符无需填写数据库字段，系统自动处理：

| 占位符 | 说明 |
|--------|------|
| `{{yesterday}}` | 昨日（YYYY-MM-DD） |
| `{{today}}` | 今日 |
| `{{last_week}}` | 上周同日 |
| `{{date}}` | 当前时间戳（用于文件名） |

## 运行命令

```bash
# 第一步：扫描模板，生成 CSV 骨架
python report_engine.py --scan templates/my_report.docx

# 第二步：生成报告（自动识别同名 .csv 或 .json）
python report_engine.py --template templates/my_report.docx

# 批量生成模板目录中所有报告
python report_engine.py

# 列出可用模板及配置状态
python report_engine.py --list
```
