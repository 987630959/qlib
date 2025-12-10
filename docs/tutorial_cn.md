# Qlib 量化投资框架使用教程

## 目录

- [1. 简介](#1-简介)
- [2. 安装配置](#2-安装配置)
- [3. 数据准备](#3-数据准备)
- [4. 快速开始](#4-快速开始)
- [5. 核心概念](#5-核心概念)
- [6. 模型训练](#6-模型训练)
- [7. 回测与分析](#7-回测与分析)
- [8. 高级用法](#8-高级用法)
- [9. 常见问题](#9-常见问题)

---

## 1. 简介

### 1.1 什么是 Qlib？

**Qlib** 是由微软开发的开源 **AI 量化投资平台**，旨在帮助研究人员和从业者构建、测试和部署基于机器学习的量化交易策略。

### 1.2 核心特性

| 特性 | 描述 |
|------|------|
| 🚀 高性能数据处理 | 复杂数据处理仅需 7.4 秒（对比传统方案 184 秒以上） |
| 🤖 丰富的模型库 | 内置 30+ 种 SOTA 模型（LightGBM、LSTM、Transformer 等） |
| 📊 完整工作流 | 数据 → 模型 → 回测 → 分析 一站式解决方案 |
| 🔄 在线服务 | 支持模型自动滚动更新和在线预测 |
| 🎮 强化学习 | 内置订单执行和投资组合管理的 RL 框架 |

### 1.3 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    Qlib 整体架构                              │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │
│  │  数据层  │→│  模型层  │→│  策略层  │→│  回测层  │         │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘         │
│       ↓            ↓            ↓            ↓               │
│  ┌─────────────────────────────────────────────────┐        │
│  │              工作流管理 (MLflow)                  │        │
│  └─────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 安装配置

### 2.1 环境要求

- **Python 版本**: 3.8, 3.9, 3.10, 3.11, 3.12
- **操作系统**: Linux, macOS, Windows

### 2.2 安装方式

#### 方式一：通过 pip 安装（推荐）

```bash
pip install pyqlib
```

#### 方式二：从源码安装

```bash
# 安装依赖
pip install numpy
pip install --upgrade cython

# 克隆仓库
git clone https://github.com/microsoft/qlib.git
cd qlib

# 安装（选择一种）
pip install .                    # 标准安装
pip install -e .[dev]           # 开发模式安装
```

### 2.3 可选依赖

根据需求安装额外依赖：

```bash
# 强化学习支持
pip install pyqlib[rl]

# 分析工具
pip install pyqlib[analysis]

# 完整安装
pip install pyqlib[dev,rl,analysis]
```

### 2.4 验证安装

```python
import qlib
print(qlib.__version__)
```

---

## 3. 数据准备

### 3.1 数据来源

由于官方数据源已停用，推荐使用以下替代方案：

| 数据源 | 市场 | 说明 |
|--------|------|------|
| [investment_data](https://github.com/chenditc/investment_data) | 中国 A 股 | 社区维护的数据集 |
| Yahoo Finance | 美股 | 使用内置爬虫获取 |
| Baostock | 中国 A 股 | 开源数据源 |

### 3.2 下载中国 A 股数据

使用社区数据：

```bash
# 下载数据
wget https://github.com/chenditc/investment_data/releases/download/20230101/qlib_bin.tar.gz

# 解压到 qlib 数据目录
mkdir -p ~/.qlib/qlib_data/cn_data
tar -zxvf qlib_bin.tar.gz -C ~/.qlib/qlib_data/cn_data
```

### 3.3 下载美股数据

使用内置的 Yahoo Finance 爬虫：

```bash
# 进入数据收集脚本目录
cd scripts/data_collector/yahoo/

# 下载数据
python collector.py download_data \
    --source_dir ~/.qlib/stock_data/source \
    --start 2010-01-01 \
    --end 2023-12-31 \
    --delay 1 \
    --interval 1d
```

### 3.4 数据格式转换

将原始数据转换为 Qlib 格式：

```bash
python scripts/dump_bin.py dump_all \
    --csv_path ~/.qlib/stock_data/source \
    --qlib_dir ~/.qlib/qlib_data/us_data \
    --include_fields open,close,high,low,volume \
    --date_field_name date
```

---

## 4. 快速开始

### 4.1 初始化 Qlib

```python
import qlib
from qlib.constant import REG_CN, REG_US

# 初始化（中国市场）
qlib.init(
    provider_uri="~/.qlib/qlib_data/cn_data",
    region=REG_CN
)

# 或初始化（美国市场）
# qlib.init(
#     provider_uri="~/.qlib/qlib_data/us_data",
#     region=REG_US
# )
```

### 4.2 数据访问示例

```python
from qlib.data import D

# 获取交易日历
calendar = D.calendar(start_time='2020-01-01', end_time='2023-12-31', freq='day')
print(f"交易日数量: {len(calendar)}")

# 获取股票池
instruments = D.instruments('csi300')  # 沪深300成分股
print(f"股票数量: {len(D.list_instruments(instruments))}")

# 获取行情数据
data = D.features(
    instruments=['SH600000', 'SH600519'],  # 浦发银行、贵州茅台
    fields=['$close', '$volume', '$open', '$high', '$low'],
    start_time='2023-01-01',
    end_time='2023-12-31',
    freq='day'
)
print(data.head(10))
```

### 4.3 使用 YAML 配置运行工作流

最简单的方式是使用预定义的 YAML 配置：

```bash
# 运行 LightGBM 模型
qrun examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml
```

### 4.4 使用 Python 代码运行工作流

```python
import qlib
from qlib.constant import REG_CN
from qlib.utils import init_instance_by_config
from qlib.workflow import R
from qlib.workflow.record_temp import SignalRecord, PortAnaRecord, SigAnaRecord
from qlib.contrib.data.handler import Alpha158

# 初始化
qlib.init(provider_uri="~/.qlib/qlib_data/cn_data", region=REG_CN)

# 定义市场和时间范围
market = "csi300"
benchmark = "SH000300"

# 数据处理器配置
data_handler_config = {
    "start_time": "2008-01-01",
    "end_time": "2020-08-01",
    "fit_start_time": "2008-01-01",
    "fit_end_time": "2014-12-31",
    "instruments": market,
}

# 模型配置
model_config = {
    "class": "LGBModel",
    "module_path": "qlib.contrib.model.gbdt",
    "kwargs": {
        "loss": "mse",
        "colsample_bytree": 0.8879,
        "learning_rate": 0.0421,
        "subsample": 0.8789,
        "lambda_l1": 205.6999,
        "lambda_l2": 580.9768,
        "max_depth": 8,
        "num_leaves": 210,
        "num_threads": 20,
    },
}

# 数据集配置
dataset_config = {
    "class": "DatasetH",
    "module_path": "qlib.data.dataset",
    "kwargs": {
        "handler": {
            "class": "Alpha158",
            "module_path": "qlib.contrib.data.handler",
            "kwargs": data_handler_config,
        },
        "segments": {
            "train": ("2008-01-01", "2014-12-31"),
            "valid": ("2015-01-01", "2016-12-31"),
            "test": ("2017-01-01", "2020-08-01"),
        },
    },
}

# 创建模型和数据集
model = init_instance_by_config(model_config)
dataset = init_instance_by_config(dataset_config)

# 运行实验
with R.start(experiment_name="lightgbm_tutorial"):
    # 训练模型
    model.fit(dataset)

    # 生成预测信号
    recorder = R.get_recorder()
    sr = SignalRecord(model, dataset, recorder)
    sr.generate()

    # 信号分析
    sar = SigAnaRecord(recorder)
    sar.generate()

    # 投资组合分析
    par = PortAnaRecord(recorder, config={"strategy": {"class": "TopkDropoutStrategy"}})
    par.generate()

print("实验完成！")
```

---

## 5. 核心概念

### 5.1 数据层 (Data Layer)

#### 数据 API (`D`)

`D` 是 Qlib 的核心数据访问接口：

```python
from qlib.data import D

# 基础数据字段
# $close - 收盘价
# $open  - 开盘价
# $high  - 最高价
# $low   - 最低价
# $volume - 成交量
# $vwap  - 成交量加权平均价

# 使用表达式计算衍生特征
fields = [
    "$close",                           # 原始收盘价
    "Ref($close, 1)",                   # 昨日收盘价
    "Mean($close, 5)",                  # 5日均价
    "Std($close, 20)",                  # 20日标准差
    "($close - Ref($close, 1)) / Ref($close, 1)",  # 日收益率
    "Rank($close / Ref($close, 5))",    # 5日收益率排名
]

data = D.features(['SH600000'], fields, start_time='2023-01-01', end_time='2023-12-31')
```

#### 常用表达式运算符

| 运算符 | 说明 | 示例 |
|--------|------|------|
| `Ref(x, n)` | 滞后 n 期 | `Ref($close, 1)` |
| `Mean(x, n)` | n 期均值 | `Mean($close, 5)` |
| `Std(x, n)` | n 期标准差 | `Std($close, 20)` |
| `Max(x, n)` | n 期最大值 | `Max($high, 10)` |
| `Min(x, n)` | n 期最小值 | `Min($low, 10)` |
| `Rank(x)` | 截面排名 | `Rank($close)` |
| `Corr(x, y, n)` | n 期相关系数 | `Corr($close, $volume, 10)` |
| `Sum(x, n)` | n 期求和 | `Sum($volume, 5)` |
| `Delta(x, n)` | n 期差值 | `Delta($close, 5)` |

### 5.2 数据处理器 (Data Handler)

Qlib 提供两种预定义的特征集：

#### Alpha158

158 个手工设计的因子，适合树模型：

```python
from qlib.contrib.data.handler import Alpha158

handler = Alpha158(
    instruments="csi300",
    start_time="2020-01-01",
    end_time="2023-12-31",
)
```

#### Alpha360

360 个原始 OHLCV 特征，适合深度学习：

```python
from qlib.contrib.data.handler import Alpha360

handler = Alpha360(
    instruments="csi300",
    start_time="2020-01-01",
    end_time="2023-12-31",
)
```

### 5.3 数据集 (Dataset)

```python
from qlib.data.dataset import DatasetH

dataset = DatasetH(
    handler=handler,
    segments={
        "train": ("2015-01-01", "2018-12-31"),
        "valid": ("2019-01-01", "2019-12-31"),
        "test": ("2020-01-01", "2020-12-31"),
    }
)

# 获取各数据集
train_data = dataset.prepare("train")
valid_data = dataset.prepare("valid")
test_data = dataset.prepare("test")
```

---

## 6. 模型训练

### 6.1 内置模型一览

| 模型类型 | 模型名称 | 适用场景 |
|----------|----------|----------|
| **树模型** | LightGBM, XGBoost, CatBoost | 表格数据，Alpha158 |
| **神经网络** | MLP, TabNet | 通用 |
| **时序模型** | LSTM, GRU, TCN, ALSTM | 时序特征，Alpha360 |
| **注意力模型** | Transformer, GATs, Localformer | 复杂时序依赖 |
| **高级模型** | TRA, HIST, IGMTF | 最新研究成果 |

### 6.2 使用 LightGBM

```python
from qlib.contrib.model.gbdt import LGBModel

model = LGBModel(
    loss="mse",
    learning_rate=0.05,
    max_depth=8,
    num_leaves=200,
    num_threads=10,
    early_stopping_rounds=50,
)

# 训练
model.fit(dataset)

# 预测
predictions = model.predict(dataset)
```

### 6.3 使用 LSTM

```python
from qlib.contrib.model.pytorch_lstm import LSTM

model = LSTM(
    d_feat=360,       # 输入特征维度
    hidden_size=64,
    num_layers=2,
    dropout=0.0,
    n_epochs=200,
    lr=0.001,
    batch_size=2000,
    GPU=0,
)

model.fit(dataset)
predictions = model.predict(dataset)
```

### 6.4 使用 Transformer

```python
from qlib.contrib.model.pytorch_transformer import Transformer

model = Transformer(
    d_feat=6,          # 特征维度
    d_model=64,        # 模型维度
    nhead=2,           # 注意力头数
    num_layers=2,
    dropout=0.0,
    n_epochs=100,
    lr=0.0001,
    batch_size=2048,
    GPU=0,
)

model.fit(dataset)
```

### 6.5 自定义模型

```python
from qlib.model.base import Model

class MyCustomModel(Model):
    def __init__(self, **kwargs):
        super().__init__()
        # 初始化模型
        self.model = None

    def fit(self, dataset, **kwargs):
        """训练模型"""
        train_data = dataset.prepare("train")
        x_train, y_train = train_data["feature"], train_data["label"]
        # 训练逻辑
        pass

    def predict(self, dataset, **kwargs):
        """预测"""
        test_data = dataset.prepare("test")
        x_test = test_data["feature"]
        # 预测逻辑
        return predictions

    def finetune(self, dataset, **kwargs):
        """微调（可选）"""
        pass
```

---

## 7. 回测与分析

### 7.1 策略配置

```python
from qlib.contrib.strategy import TopkDropoutStrategy

strategy_config = {
    "class": "TopkDropoutStrategy",
    "module_path": "qlib.contrib.strategy",
    "kwargs": {
        "signal": None,  # 由 recorder 自动填充
        "topk": 50,      # 持有前50只股票
        "n_drop": 5,     # 每次最多调仓5只
        "method_sell": "bottom",
        "method_buy": "top",
    },
}
```

### 7.2 回测执行器配置

```python
executor_config = {
    "class": "SimulatorExecutor",
    "module_path": "qlib.backtest.executor",
    "kwargs": {
        "time_per_step": "day",
        "generate_portfolio_metrics": True,
    },
}
```

### 7.3 完整回测示例

```python
import qlib
from qlib.constant import REG_CN
from qlib.contrib.evaluate import backtest_daily
from qlib.contrib.strategy import TopkDropoutStrategy

qlib.init(provider_uri="~/.qlib/qlib_data/cn_data", region=REG_CN)

# 假设已有预测信号 pred
# pred = model.predict(dataset)

# 运行回测
portfolio_metric, indicator = backtest_daily(
    start_time="2020-01-01",
    end_time="2020-12-31",
    pred=predictions,  # 模型预测
    strategy=TopkDropoutStrategy(
        topk=50,
        n_drop=5,
    ),
    account=1000000,  # 初始资金 100 万
    benchmark="SH000300",  # 基准：沪深300
    exchange_kwargs={
        "freq": "day",
        "limit_threshold": 0.095,  # 涨跌停限制
        "deal_price": "close",
    },
)

# 输出结果
print(portfolio_metric)
```

### 7.4 分析指标

#### 信号分析指标

| 指标 | 说明 | 期望 |
|------|------|------|
| **IC** | 信息系数（预测与收益相关性） | > 0.03 |
| **ICIR** | IC 的信息比率 | > 0.3 |
| **Rank IC** | 排名 IC | > 0.05 |
| **Rank ICIR** | 排名 IC 信息比率 | > 0.5 |

#### 组合分析指标

| 指标 | 说明 |
|------|------|
| **年化收益率** | 策略的年化收益 |
| **夏普比率** | 风险调整后收益 |
| **最大回撤** | 最大亏损幅度 |
| **信息比率** | 相对基准的超额收益 |
| **换手率** | 组合换手频率 |

### 7.5 可视化分析

```python
from qlib.contrib.report import analysis_position, analysis_model

# 分析持仓
analysis_position.report_graph(recorder)

# 分析模型预测
analysis_model.model_performance_graph(recorder)
```

---

## 8. 高级用法

### 8.1 实验管理 (MLflow)

```python
from qlib.workflow import R

# 创建实验
with R.start(experiment_name="my_experiment", recorder_name="run_001"):
    # 记录参数
    R.log_params(learning_rate=0.01, batch_size=256)

    # 记录指标
    R.log_metrics(ic=0.05, icir=0.4)

    # 保存模型
    R.save_objects(**{"model.pkl": model})

    # 获取 recorder ID
    recorder = R.get_recorder()
    print(f"Recorder ID: {recorder.id}")

# 查看实验结果
experiments = R.list_experiments()
for exp in experiments:
    print(f"实验: {exp.name}")
    for rec in exp.list_recorders():
        print(f"  - {rec.id}: IC={rec.load_object('ic')}")
```

### 8.2 模型滚动更新

```python
from qlib.workflow.online.strategy import RollingStrategy

rolling_config = {
    "class": "RollingStrategy",
    "kwargs": {
        "h_path": "path/to/handler",
        "model": model_config,
        "dataset": dataset_config,
        "rolling_period": 20,  # 每20个交易日更新
    }
}
```

### 8.3 强化学习订单执行

```python
from qlib.rl.order_execution import SingleAssetOrderExecution
from qlib.rl.trainer import Trainer

# 配置环境
env_config = {
    "class": "SingleAssetOrderExecution",
    "kwargs": {
        "order_dir": "buy",
        "horizon": 30,  # 30分钟执行窗口
    }
}

# 训练 RL 策略
trainer = Trainer(
    env=env_config,
    policy="PPO",
    train_episodes=1000,
)
trainer.train()
```

### 8.4 元学习 (Meta Learning)

```python
from qlib.contrib.meta.incremental import MetaIncrementalModel

# 配置元学习模型
meta_model = MetaIncrementalModel(
    base_model=model_config,
    meta_epochs=10,
    adaptation_steps=5,
)

# 训练
meta_model.fit(dataset)

# 快速适应新任务
meta_model.adapt(new_dataset)
```

### 8.5 自定义数据处理器

```python
from qlib.data.dataset.handler import DataHandlerLP

class MyHandler(DataHandlerLP):
    def __init__(self, instruments, start_time, end_time, **kwargs):
        # 定义自己的特征
        data_loader = {
            "class": "QlibDataLoader",
            "kwargs": {
                "freq": "day",
                "config": {
                    "feature": self.get_feature_config(),
                    "label": self.get_label_config(),
                },
            },
        }
        super().__init__(
            instruments=instruments,
            start_time=start_time,
            end_time=end_time,
            data_loader=data_loader,
            **kwargs
        )

    def get_feature_config(self):
        """自定义特征"""
        return [
            ["$close/Ref($close,1)-1", "daily_return"],
            ["Mean($close,5)/Mean($close,20)-1", "ma_ratio"],
            ["Std($close,20)", "volatility"],
            # 添加更多特征...
        ]

    def get_label_config(self):
        """标签：未来收益率"""
        return [
            ["Ref($close,-2)/Ref($close,-1)-1", "LABEL0"]
        ]
```

---

## 9. 常见问题

### 9.1 数据相关

**Q: 如何获取最新的数据？**

A: 使用内置的数据收集器定期更新：

```bash
cd scripts/data_collector/yahoo/
python collector.py update_data --qlib_dir ~/.qlib/qlib_data/us_data
```

**Q: 如何使用自己的数据？**

A: 参考 `scripts/dump_bin.py` 将 CSV 数据转换为 Qlib 格式：

```bash
python scripts/dump_bin.py dump_all \
    --csv_path /path/to/your/csv \
    --qlib_dir ~/.qlib/qlib_data/custom \
    --date_field_name date
```

### 9.2 模型相关

**Q: GPU 训练报错怎么办？**

A: 确保安装了正确版本的 PyTorch：

```bash
# CUDA 11.8
pip install torch --index-url https://download.pytorch.org/whl/cu118
```

**Q: 模型过拟合怎么办？**

A: 尝试以下方法：
1. 增加 `early_stopping_rounds`
2. 减少 `max_depth` 或 `num_leaves`
3. 增加正则化参数 `lambda_l1`, `lambda_l2`
4. 使用更长的验证集

### 9.3 回测相关

**Q: 回测结果与实盘差异大？**

A: 注意以下细节：
1. 设置合理的滑点 (`deal_price`)
2. 考虑涨跌停限制 (`limit_threshold`)
3. 避免未来函数（用 `Ref` 延迟标签）
4. 考虑交易成本

**Q: 如何模拟真实交易成本？**

```python
exchange_kwargs = {
    "freq": "day",
    "deal_price": "vwap",  # 使用 VWAP 成交
    "limit_threshold": 0.095,
    "open_cost": 0.0003,   # 买入手续费
    "close_cost": 0.0013,  # 卖出手续费（含印花税）
    "min_cost": 5,         # 最低手续费
}
```

### 9.4 性能优化

**Q: 数据加载太慢？**

A: 使用缓存机制：

```python
qlib.init(
    provider_uri="~/.qlib/qlib_data/cn_data",
    region=REG_CN,
    expression_cache="DiskExpressionCache",  # 启用表达式缓存
    dataset_cache="DiskDatasetCache",        # 启用数据集缓存
)
```

**Q: 如何并行训练多个模型？**

A: 使用 `run_all_model.py`：

```bash
python examples/run_all_model.py \
    --models LightGBM XGBoost CatBoost \
    --n_jobs 4
```

---

## 附录

### A. 项目结构

```
qlib/
├── qlib/                 # 主代码包
│   ├── data/            # 数据基础设施
│   ├── model/           # 模型框架
│   ├── contrib/         # 贡献模块（30+模型、策略等）
│   ├── backtest/        # 回测引擎
│   ├── workflow/        # 工作流管理
│   └── rl/              # 强化学习框架
├── examples/            # 示例代码
│   ├── benchmarks/      # 26 种模型基准
│   └── tutorial/        # Jupyter 教程
├── scripts/             # 数据收集脚本
└── docs/                # 文档
```

### B. 推荐学习路径

1. **入门**: 运行 `qrun examples/benchmarks/LightGBM/workflow_config_lightgbm_Alpha158.yaml`
2. **进阶**: 阅读 `examples/workflow_by_code.py` 理解代码方式
3. **深入**: 学习 `examples/tutorial/` 下的 Jupyter Notebook
4. **高级**: 探索 `examples/benchmarks_dynamic/` 了解市场适应技术

### C. 社区资源

- **GitHub**: https://github.com/microsoft/qlib
- **文档**: https://qlib.readthedocs.io/
- **社区数据**: https://github.com/chenditc/investment_data
- **论文**: [Qlib: An AI-oriented Quantitative Investment Platform](https://arxiv.org/abs/2009.11189)

---

> **提示**: 本教程基于 Qlib 最新版本编写。如遇问题，请参考官方文档或在 GitHub 上提交 Issue。

**祝你量化投资之旅顺利！** 📈
