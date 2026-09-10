# 核心模块划分与关键伪代码

## 设计目标

本设计面向一个遗留 Python CSV 处理脚本：旧脚本每天读取单只股票数据，计算移动平均线，并生成下游报表依赖的旧输出。新需求要求支持多股票批处理、按行业输出、处理停牌缺失交易日，同时不能破坏旧输出格式。

设计原则：

- 解析、清洗、计算、行业聚合和输出写入解耦。
- 旧输出路径默认保持兼容，新能力走新增路径或显式开关。
- 重构按小切片推进，每个切片都能用旧行为基线验证。
- 兼容适配层集中处理字段顺序、日期格式、数值精度、空值表示和排序。

## 模块划分

| 模块 | 职责 | 输入 | 输出 | 兼容性关注点 |
| --- | --- | --- | --- | --- |
| `config` | 解析命令行参数、配置文件和运行模式 | CLI 参数、配置文件、环境变量 | 标准化配置对象 | 默认参数必须保持旧脚本行为 |
| `io.reader` | 读取 CSV 并保留原始字段 | CSV 文件路径、编码、分隔符 | 原始记录列表 | 编码、列名大小写、日期格式不能被静默改变 |
| `schema` | 校验必需字段和类型 | 原始记录 | 校验结果、错误明细 | 对旧脚本可容忍的问题保持同等容忍度 |
| `cleaning` | 去重、空字段处理、停牌缺失交易日标记 | 校验后的记录 | 清洗后记录、诊断信息 | 不随意补价格；缺失交易日优先标记，不伪造行情 |
| `calendar` | 对齐交易日，区分停牌和缺数据 | 清洗后记录、交易日历 | 对齐后的时间序列 | 交易日历版本需要可追踪 |
| `ma_calculator` | 按股票计算移动平均线 | 单股票时间序列、窗口参数 | 股票级指标记录 | 窗口不足、空值和精度与旧输出一致 |
| `industry_aggregator` | 按行业聚合股票级结果 | 股票级指标、行业映射 | 行业级指标记录 | 聚合口径需说明，例如等权平均或按市值加权 |
| `io.writer` | 输出旧格式和新格式文件 | 指标记录、输出契约 | CSV 输出、诊断输出 | 旧输出字段顺序、文件名和排序保持不变 |
| `compat` | 旧输出适配层 | 新内部模型、旧契约 | 旧格式记录 | 所有旧字段由这里统一映射，避免散落在业务逻辑中 |

## 数据模型

```python
StockBar = {
    "trade_date": date,
    "stock_code": str,
    "stock_name": str,
    "industry": str,
    "close": Decimal | None,
    "volume": int | None,
    "source_row_id": str,
}

StockMetric = {
    "trade_date": date,
    "stock_code": str,
    "stock_name": str,
    "industry": str,
    "close": Decimal | None,
    "ma5": Decimal | None,
    "data_status": "normal" | "suspended" | "missing" | "invalid",
}

IndustryMetric = {
    "trade_date": date,
    "industry": str,
    "stock_count": int,
    "valid_stock_count": int,
    "avg_close": Decimal | None,
    "avg_ma5": Decimal | None,
}
```

## 关键伪代码

### 主流程

```python
def run(argv):
    config = load_config(argv)
    raw_rows = read_csv(config.input_path, encoding=config.encoding)
    schema_report = validate_schema(raw_rows, required_columns=config.required_columns)
    cleaned_rows, diagnostics = clean_rows(raw_rows, config.cleaning_policy)
    aligned_rows = align_trading_days(cleaned_rows, config.trade_calendar)

    stock_metrics = []
    for stock_code, rows in group_by_stock(aligned_rows):
        stock_metrics.extend(calculate_moving_average(rows, windows=config.ma_windows))

    if config.enable_industry_output:
        industry_metrics = aggregate_by_industry(stock_metrics, method=config.industry_method)
        write_industry_output(industry_metrics, config.industry_output_path)

    if config.legacy_mode:
        legacy_rows = adapt_to_legacy_output(stock_metrics, config.legacy_contract)
        write_legacy_output(legacy_rows, config.legacy_output_path)

    if config.write_diagnostics:
        write_diagnostics(schema_report, diagnostics, config.diagnostics_path)
```

### 解析与校验解耦

```python
def read_csv(path, encoding):
    rows = csv_read(path, encoding=encoding)
    return [preserve_original_columns(row) for row in rows]

def validate_schema(rows, required_columns):
    report = ValidationReport()
    for index, row in enumerate(rows):
        for column in required_columns:
            if column not in row:
                report.add_error(index, column, "missing_column")
        report.add_warning_if_extra_columns(row)
    return report
```

### 停牌和缺失数据处理

```python
def align_trading_days(rows, trade_calendar):
    result = []
    by_stock = group_by_stock(rows)
    for stock_code, stock_rows in by_stock.items():
        row_by_date = index_by_trade_date(stock_rows)
        for trade_date in trade_calendar:
            if trade_date in row_by_date:
                result.append(mark_status(row_by_date[trade_date], "normal"))
            elif is_known_suspended(stock_code, trade_date):
                result.append(empty_bar(stock_code, trade_date, status="suspended"))
            else:
                result.append(empty_bar(stock_code, trade_date, status="missing"))
    return result
```

### 移动平均线计算

```python
def calculate_moving_average(rows, windows):
    rows = sort_by_trade_date(rows)
    metrics = []
    for i, row in enumerate(rows):
        metric = base_metric(row)
        for window in windows:
            window_rows = rows[max(0, i - window + 1): i + 1]
            closes = [r["close"] for r in window_rows]
            if len(closes) < window or any(value is None for value in closes):
                metric[f"ma{window}"] = None
            else:
                metric[f"ma{window}"] = round_decimal(sum(closes) / window, places=2)
        metrics.append(metric)
    return metrics
```

### 行业聚合

```python
def aggregate_by_industry(stock_metrics, method="equal_weight"):
    result = []
    for (trade_date, industry), rows in group_by_date_and_industry(stock_metrics):
        valid_rows = [r for r in rows if r["data_status"] == "normal" and r["ma5"] is not None]
        result.append({
            "trade_date": trade_date,
            "industry": industry,
            "stock_count": len(rows),
            "valid_stock_count": len(valid_rows),
            "avg_close": mean([r["close"] for r in valid_rows]),
            "avg_ma5": mean([r["ma5"] for r in valid_rows]),
        })
    return sort_by_date_industry(result)
```

### 旧输出兼容适配

```python
def adapt_to_legacy_output(stock_metrics, contract):
    rows = filter_legacy_scope(stock_metrics, stock_code=contract.default_stock_code)
    rows = sort_by_trade_date(rows)
    return [
        {
            "date": format_date(row["trade_date"], contract.date_format),
            "close": format_decimal(row["close"], contract.decimal_places),
            "ma5": format_decimal(row["ma5"], contract.decimal_places),
        }
        for row in rows
    ]
```

## 重构切片

| 切片 | 变更 | 验收 |
| --- | --- | --- |
| 1 | 抽出 CSV 读取函数，不改输出 | 旧样例输出字节级或逐字段一致 |
| 2 | 抽出移动平均线计算函数 | 单股票 MA 结果一致 |
| 3 | 增加内部标准数据模型 | 旧输出由适配层生成且契约不变 |
| 4 | 增加多股票分组处理 | 旧单股票默认路径不变，新路径输出股票维度 |
| 5 | 增加停牌和缺失交易日标记 | 不伪造价格，诊断输出可解释 |
| 6 | 增加行业聚合输出 | 行业输出为新增文件，不影响旧报表 |
| 7 | 增加灰度开关和双写 | 可一键关闭新输出，旧输出继续产生 |

## 兼容边界

旧输出默认不变：

- 文件名和目录不变。
- 字段名、字段顺序不变。
- 日期格式不变。
- 数值精度和空值表示不变。
- 默认只输出旧报表依赖的单股票结果。

新能力通过显式配置开启：

- `--multi-stock`
- `--industry-output`
- `--diagnostics-output`
- `--shadow-run`
