---
name: data-cleansing
description: Designs and executes data cleaning pipelines for tabular datasets (CSV, Excel, Parquet, JSON, logs), including profiling, schema inference, missing value handling, outlier treatment, type conversion, normalization, deduplication, and basic validation. Use when the user asks to clean, preprocess, wrangle, or prepare data for analysis or machine learning.
---

# Data Cleansing

## Purpose

This skill helps design and implement **data cleansing workflows for tabular data** (CSV, Excel, Parquet, JSON-lines, logs, database exports), especially when preparing data for:

- Exploratory data analysis (EDA)
- Dashboarding / BI
- Machine learning training / inference
- Reporting and downstream pipelines

Assume the agent already knows general data science and programming concepts; focus on **workflow, checks, and decision logic**, not explaining basic pandas/SQL syntax.

---

## When to Apply This Skill

Use this skill when:

- The user mentions **“数据清洗 / 清洗数据 / 数据预处理 / data cleaning / data wrangling / data preprocessing / feature engineering”**
- The user wants to **“整理脏数据 / 处理缺失值 / 处理异常值 / 去重 / 统一字段 / 修正格式 / 规范数据”**
- The user provides **CSV/Excel/Parquet/JSON/log** files that are noisy or inconsistent
- The user is blocked on **模型训练 / 指标计算 / 报表生成** because of low-quality data
- The user asks for **reusable清洗脚本、pipeline、工具函数或 notebook 模板**

If the task is only about **visualization** and not about **improving data quality**, do not over-apply this skill; just perform minimal necessary cleaning.

---

## Overall Workflow (High Level)

When performing data cleansing, follow this standard loop:

1. **Clarify goal & constraints**
2. **Profile the raw data**
3. **Design a cleaning plan (checklist)**
4. **Implement cleaning steps in code / SQL**
5. **Validate results against expectations**
6. **Package as reusable pipeline if useful**

You can copy this checklist into the conversation and tick items as you go.

```markdown
数据清洗 Checklist:
- [ ] 明确业务目标和下游使用场景
- [ ] 初步数据概况（行数、列数、数据类型、缺失率）
- [ ] 字段级质量检查（取值分布、异常值、非法值、枚举集合）
- [ ] 缺失值处理策略制定与实现
- [ ] 异常值 / 极端值检测与处理
- [ ] 类型转换与格式统一（时间、类别、数值、布尔）
- [ ] 去重与主键/唯一约束检查
- [ ] 业务规则校验（范围、逻辑约束、跨字段约束）
- [ ] 保存清洗结果（文件/表）与记录清洗规则
- [ ] 如有必要，封装为可复用 pipeline / 函数
```

---

## Step 1: Clarify Goal & Constraints

Before changing data, briefly infer or ask (if absolutely necessary) about:

- **下游用途**:
  - 仅用于描述统计 / 看报表 → 可以允许适度损失数据
  - 用于训练模型 → 更加严格，避免引入偏差
- **可接受的信息损失**:
  - 可以删除多少行？
  - 是否可以丢弃部分字段？
- **关键字段**:
  - 哪些列是业务上“必须可靠”的（ID、时间、金额、状态等）
  - 哪些列可以视作辅助信息，对下游影响较小

Document assumptions in the response:

> 假设该数据主要用于 ×× 分析，对单条记录精度要求中等，但整体分布必须真实。

---

## Step 2: Profile the Raw Data

For pandas-style workflows, prefer a small, standard profiling set:

- **结构与基本信息**:
  - `df.shape`, `df.dtypes`, `df.head()`
- **缺失情况**:
  - `df.isna().mean().sort_values(ascending=False)`
- **重复情况**:
  - `df.duplicated().mean()`
- **字段级分布**:
  - 数值列：`df[col].describe([0.01, 0.99])`
  - 类别列：`df[col].value_counts(dropna=False).head(20)`

Encourage the user to run a small snippet or provide summary tables instead of pasting entire raw datasets.

If the data is in SQL, use:

- `COUNT(*)`, `COUNT(DISTINCT col)` 
- `MIN/MAX/AVG`, `PERCENTILE_CONT(0.5/0.9/0.99)` 
- `GROUP BY` + `COUNT(*)` for top categories

---

## Step 3: Design a Cleaning Plan

Translate profiling insights into a **concrete, ordered plan**:

- **缺失值处理**:
  - 严重缺失且业务价值不大 → 列删除
  - 少量缺失且可推断 → 填充（均值/中位数/众数/前向填充/业务常量）
  - 对关键字段 → 考虑剔除行或回补外部数据
- **异常值 / 极端值**:
  - 使用 IQR 或 z-score 检测
  - 结合业务阈值（如年龄 0–120，金额非负等）
  - 选择截断（capping）、修正（替换为边界）或删除行
- **类型与格式统一**:
  - 时间：统一为 timezone-aware `datetime` 或标准字符串格式
  - 分类：统一大小写、去除空格、合并同义值
  - 数值：去除千分位符号、货币符号，确保列为数值类型
- **去重策略**:
  - 定义主键或“业务主键”（如 [user_id, date]）
  - 明确保留规则：首条、末条、根据某字段最大/最新
- **业务规则约束**:
  - 比如：`start_time <= end_time`，`quantity >= 0`，`status` 在允许集合内

Represent the plan as a short checklist before writing code.

---

## Step 4: Implement Cleaning (Code/SQL Patterns)

### 4.1 General Principles

- Prefer **可读性强、步骤清晰**的代码 over clever one-liners
- 保留原始数据副本或增加标记列，如 `is_outlier`, `imputed_amount`
- 对重要变换写简单断言 / 检查（如行数变化、缺失率变化）

### 4.2 pandas Workflow Template

Use this as a starting template for pandas-based cleaning:

```python
import pandas as pd

def load_raw(path: str) -> pd.DataFrame:
    # 根据后缀选择 read_csv / read_excel / read_parquet 等
    return pd.read_csv(path)

def profile(df: pd.DataFrame) -> None:
    print("shape:", df.shape)
    print(df.dtypes)
    print("missing ratio:")
    print(df.isna().mean().sort_values(ascending=False).head(20))

def clean(df: pd.DataFrame) -> pd.DataFrame:
    df = df.copy()

    # 示例：去除完全空行
    df = df.dropna(how="all")

    # 示例：处理关键字段缺失
    # df = df.dropna(subset=["user_id", "event_time"])

    # 示例：类型转换
    # df["event_time"] = pd.to_datetime(df["event_time"], errors="coerce")

    # 示例：去重
    # df = df.sort_values("event_time").drop_duplicates(subset=["user_id", "event_time"], keep="last")

    # TODO: 按具体业务补充清洗规则
    return df

def main(raw_path: str, cleaned_path: str) -> None:
    df_raw = load_raw(raw_path)
    profile(df_raw)
    df_clean = clean(df_raw)
    print("cleaned shape:", df_clean.shape)
    df_clean.to_csv(cleaned_path, index=False)

if __name__ == "__main__":
    main("raw.csv", "cleaned.csv")
```

When asked, adapt this template to the user’s schema and requirements.

### 4.3 SQL Workflow Template

For database-side cleaning, structure queries as CTEs:

```sql
WITH raw AS (
    SELECT * FROM source_table
),
typed AS (
    SELECT
        CAST(user_id AS BIGINT) AS user_id,
        CAST(event_time AS TIMESTAMP) AS event_time,
        amount::NUMERIC(18, 2) AS amount,
        status
    FROM raw
),
filtered AS (
    SELECT *
    FROM typed
    WHERE amount >= 0
      AND event_time IS NOT NULL
),
deduped AS (
    SELECT DISTINCT *
    FROM filtered
)
SELECT *
FROM deduped;
```

Always comment critical business rules in SQL when the user plans to maintain these queries long term.

---

## Step 5: Validation & Quality Checks

After cleaning, always run **sanity checks**:

- **行数变化**:
  - 是否在预期范围内（例如：删除 5–15% 的行是合理的？）
- **缺失率变化**:
  - 关键字段缺失率应显著下降或为 0
- **数值范围**:
  - 检查最值是否仍然合理（如年龄、金额、比率等）
- **分布漂移**:
  - 对比清洗前后某些关键字段的分布（直方图/箱线图/分位数）

Example validation snippet (pandas):

```python
def validate(df_raw, df_clean):
    print("raw rows:", len(df_raw), "clean rows:", len(df_clean))
    # 例如，行数不应减少超过 50%
    assert len(df_clean) >= 0.5 * len(df_raw), "Too many rows were dropped"
```

If validation fails, clearly explain likely reasons and propose adjustments (weaker filters, different missing value strategy, etc.).

---

## Reusable Pipelines & Modularization

When the user expects to **重复使用**清洗逻辑：

- 将清洗过程拆成 **小函数** / **小模块**（按主题：缺失值处理、类型转换、业务规则校验等）
- 对每个函数写 **简短 docstring**，说明：
  - 该步骤意图
  - 输入/输出约定
- 若项目较大，考虑使用：
  - Python 包结构（`data/`, `src/`, `notebooks/`）
  - `pyproject.toml` 或 `requirements.txt` 记录依赖

Encourage the user to place generic cleaning utilities in a shared module (e.g. `data_cleaning.py`) rather than scattering similar code across notebooks.

---

## Examples of How to Respond (Patterns)

### Example 1: 用户给出 CSV 并说“帮我做基础数据清洗”

**Recommended response structure**:

1. 简要说明你将做哪些事情（缺失值、异常值、类型转换、去重）
2. 给出一段完整但易改造的 pandas 脚本
3. 在脚本中留“TODO: 根据业务补充规则”的标记
4. 提醒用户先在样例数据上测试，再用在全量上

### Example 2: 用户说“模型效果很差，怀疑是数据太脏”

优先做：

- 帮忙设计一个 **数据质量检查清单**（更侧重发现问题）
- 提供针对标签列与关键特征的质量检查代码
- 根据检查结果再建议相应清洗操作，而不是盲目清洗

---

## Summary

When this skill is active, prioritize:

- **先理解业务目标，再动数据**
- **先分析和度量问题，再做清洗**
- **每一步清洗都可解释、可复现、可回滚**

Keep responses concise but structured, and whenever possible, provide **可直接运行且容易修改**的代码模板，而不是零散片段。

