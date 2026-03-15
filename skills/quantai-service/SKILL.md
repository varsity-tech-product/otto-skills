---
name: quantai-service
description: >
  Crypto 单因子量化研究服务 Skill。当用户说"写一个因子"、"研究因子"、"量化打工"、
  "提交因子"、"因子回测"时加载此 Skill。
  Agent 负责编写因子插件代码并通过 HTTP 接口与服务器交互，
  服务器负责所有数据处理和计算，Agent 本地无需任何市场数据。
---

# QuantAI Service — Agent 操作手册

## 服务地址

```
BASE_URL = http://47.129.240.216:8000
```

> 启动前用 `curl ${BASE_URL}/health` 确认服务在线，若连接失败立刻告知用户。

---

## ⛔ 严禁行为

1. **禁止在本地运行任何 quant-factor-loop 脚本**（`run_workflow.py`、`step*.py` 等），所有计算都在服务器上。
2. **禁止下载或存储 parquet 数据文件**，不要尝试访问服务器上的数据文件。
3. **禁止修改 `/home/ec2-user/quant-factor-loop/` 下的任何文件**，该目录只属于服务器内部。
4. **轮询时禁止无限等待**：单个 Job 最长等待 30 分钟，超时后告知用户并停止。
5. **禁止跳过 retest 步骤**：Step 11 失败时必须修复 strategy.cs 后 retest，不能直接宣告失败。

---

## Agent 本地约定路径

每次开始新任务时，将以下路径记录到工作变量中：

```
~/.quant_agent/
├── current_job_id.txt              ← 最新 job_id（快速查阅）
└── jobs/
    └── {job_id}/
        ├── plugin.py               ← 提交时上传的因子插件（阶段2完成后保存）
        ├── strategy.cs             ← Step 4a 生成的 C# 策略（strategy_cs_ready=true 后下载）
        ├── factor_card.json        ← Step 16 因子档案卡（任务 done 后下载）
        └── step11/
            ├── equity_curves.png   ← Step 11 权益曲线图（任务 done 后下载）
            └── trade_log.csv       ← Step 11 逐笔交易记录（任务 done 后下载）
```

---

## 完整工作流程（必须按顺序执行）

### 阶段 0：确认任务

从用户描述中提取：

| 信息 | 说明 | 示例 |
|------|------|------|
| 因子逻辑 | 用自然语言描述信号如何产生 | "RSI低于30时做多" |
| 核心参数 | 窗口期、阈值等超参 | `rsi_period=14, oversold=30` |
| `factor_type` | 因子类型标识（snake_case，全局唯一） | `rsi_oversold_bounce` |
| `factor_name` | 因子名称（含主要参数值） | `rsi_14_ob30` |

若用户描述不清晰，主动补问这几项，确认后再写代码。

---

### 阶段 1：编写因子插件 plugin.py

插件文件包含两部分，**必须同时实现，逻辑必须完全一致**：

1. **`FACTOR_SECTIONS`**：C# 代码片段，服务器用它生成 `strategy.cs`
2. **`build_signal()`**：Python 函数，服务器用它做超参网格搜索

#### 插件文件完整模板

```python
import pandas as pd
import numpy as np
from typing import Any, Dict

FACTOR_TYPE = "<factor_type>"   # 与提交时的 factor_type 参数保持一致

FACTOR_DEFAULT_PARAMS = {
    "param1": <default_int>,    # 所有超参及默认值，key 用 snake_case
}

FACTOR_SECTIONS = {
    # ── 注释类（人类可读） ───────────────────────────────────────────────
    "__FACTOR_DESCRIPTION__": "因子的中文描述",
    "__FACTOR_FORMULA__":     "信号公式（注释用）",
    "__FACTOR_TYPE__":        "<factor_type>",

    # ── C# 类字段声明（每行末尾必须有 \n） ──────────────────────────────
    "__FACTOR_PARAM_FIELDS__": (
        "        private int _param1;\n"
        # 每个字段一行，注意 8 个空格缩进
    ),

    # ── C# 构造函数初始化（每行末尾必须有 \n） ───────────────────────────
    "__FACTOR_INIT__": (
        '            _param1 = GetIntParameter("param1", <default>);\n'
        # key 用连字符（"param-one"），对应 Python 端 key 用下划线（"param_one"）
    ),

    # ── 初始化日志（每行末尾必须有 \n） ─────────────────────────────────
    "__FACTOR_LOG__": (
        '            Log($"[INIT] param1={_param1}");\n'
    ),

    # ── 滑动窗口大小（合法 C# 整数表达式，不加引号） ─────────────────────
    "__PRICE_WINDOW_EXPR__": "_param1 + 1",

    # ── 额外数据列 Buffer 声明（每行末尾必须有 \n，仅用 close 时填 ""） ──
    # FactorCsvBar 字段类型（决定 Enqueue 时是否需要强转）：
    #   decimal : Open / High / Low / Close / Volume
    #             → Enqueue 时必须写 (double)bar.Volume 等，否则 CS1503 编译报错
    #   double  : TakerBuyVolume / TakerSellVolume / TakerBuyQuoteVolume /
    #             TakerSellQuoteVolume / TakerBuyTrades / TakerSellTrades / QuoteVolume
    #             → 直接 Enqueue，无需转型
    "__EXTRA_BUF_FIELDS__": "",   # 示例：'        private readonly Queue<double> _volBuf = new Queue<double>();\n'

    # ── 额外列每 bar 入队（每行末尾必须有 \n，不用时填 ""） ─────────────
    # decimal 字段示例：'            _volBuf.Enqueue((double)bar.Volume);\n'
    # double  字段示例：'            _takerBuyBuf.Enqueue(bar.TakerBuyVolume);\n'
    "__EXTRA_BUF_ENQUEUE__": "",

    # ── 额外列超窗口出队（每行末尾必须有 \n，不用时填 ""） ──────────────
    "__EXTRA_BUF_DEQUEUE__": "",  # 示例：'            if (_volBuf.Count > requiredBars) _volBuf.Dequeue();\n'

    # ── 额外列转数组供计算体使用（每行末尾必须有 \n，不用时填 ""） ────────
    "__EXTRA_BUF_TOARRAY__": "",  # 示例：'            var volumes = _volBuf.ToArray();\n'

    # ── C# 信号计算主体 ──────────────────────────────────────────────────
    # 始终可用：prices[]（close，从旧到新）
    # 若声明了 __EXTRA_BUF_TOARRAY__，对应数组也在此可用
    # 必须给 rawSignal 赋值（正=看多，负=看空）并 return true
    # 数据不足时 return false（不要 throw）
    "__FACTOR_COMPUTE_BODY__": """
            // C# 计算逻辑
            var n = prices.Length;
            if (n < _param1) return false;
            // ... 计算 ...
            rawSignal = <signal_value>;
            return true;
""",
}


def build_signal(
    close: pd.DataFrame,
    params: Dict[str, Any],
    # 声明因子用到的列（框架自动注入，与 close 地位相同）：
    # open, high, low, volume, quote_volume,
    # taker_buy_volume, taker_sell_volume,
    # taker_buy_quote_volume, taker_sell_quote_volume,
    # taker_buy_trades, taker_sell_trades
    **_kwargs,
) -> pd.DataFrame:
    """
    close  : pd.DataFrame，index=UTC DatetimeIndex，columns=币种代码
    params : dict，key 与 FACTOR_DEFAULT_PARAMS 一致
    返回   : 与 close 同形状的 DataFrame，正=看多，负=看空，NaN=无信号
    逻辑必须与 FACTOR_SECTIONS.__FACTOR_COMPUTE_BODY__ 完全一致
    """
    param1 = int(params.get("param1", <default>))
    # ... Python 实现 ...
    return signal.reindex_like(close)
```

#### C# 代码约束（违反会导致 Step 11 编译失败）

| 约束 | 说明 |
|------|------|
| `__PRICE_WINDOW_EXPR__` | 必须是纯 C# 整数表达式，不加引号，如 `_window + 1` |
| `rawSignal` | 必须在 `__FACTOR_COMPUTE_BODY__` 中被赋值 |
| 数据不足 | 用 `return false`，不要 `throw` 或 `return true` 而不赋值 |
| 类型 | 所有计算用 `double`，不用 `decimal` 或 `float` |
| 禁止调用 | `Securities[].GetLastData()`、`Portfolio`、`Order`、`SetHoldings` |
| 参数 key | `GetIntParameter("param-name", default)` 用连字符 |
| 每行末尾 | `__FACTOR_PARAM_FIELDS__` / `__FACTOR_INIT__` / `__FACTOR_LOG__` / `__EXTRA_BUF_FIELDS__` / `__EXTRA_BUF_ENQUEUE__` / `__EXTRA_BUF_DEQUEUE__` / `__EXTRA_BUF_TOARRAY__` 每行末尾加 `\n` |
| 不用额外列时 | `__EXTRA_BUF_FIELDS__` / `__EXTRA_BUF_ENQUEUE__` / `__EXTRA_BUF_DEQUEUE__` / `__EXTRA_BUF_TOARRAY__` 填空字符串 `""` |
| 额外列数组长度 | 额外列 buf 使用与 close 相同的 `requiredBars` 窗口大小 |
| **decimal → double 强转** | `Open`/`High`/`Low`/`Close`/`Volume` 是 `decimal`，Enqueue 时必须写 `(double)bar.Volume` 否则报 `CS1503`；`TakerBuy*/TakerSell*/QuoteVolume` 已是 `double`，无需转型 |

#### 参考实现（完整可运行的例子）

<details>
<summary>示例：RSI 超卖反弹因子（rsi_oversold_bounce）</summary>

```python
import pandas as pd
import numpy as np
from typing import Any, Dict

FACTOR_TYPE = "rsi_oversold_bounce"

FACTOR_DEFAULT_PARAMS = {
    "rsi_period": 14,
    "oversold":   30,
    "overbought": 70,
}

FACTOR_SECTIONS = {
    "__FACTOR_DESCRIPTION__": "RSI 超卖反弹：RSI < oversold 做多，RSI > overbought 做空",
    "__FACTOR_FORMULA__":     "RSI < oversold → +(oversold-RSI)/oversold; RSI > overbought → -(RSI-overbought)/(100-overbought)",
    "__FACTOR_TYPE__":        "rsi_oversold_bounce",
    "__FACTOR_PARAM_FIELDS__": (
        "        private int _rsiPeriod;\n"
        "        private double _oversold;\n"
        "        private double _overbought;\n"
        "        private double _prevGainEma;\n"
        "        private double _prevLossEma;\n"
        "        private bool _rsiInitialized;\n"
    ),
    "__FACTOR_INIT__": (
        '            _rsiPeriod = GetIntParameter("rsi-period", 14);\n'
        '            _oversold = GetDoubleParameter("oversold", 30.0);\n'
        '            _overbought = GetDoubleParameter("overbought", 70.0);\n'
        '            _prevGainEma = 0.0;\n'
        '            _prevLossEma = 0.0;\n'
        '            _rsiInitialized = false;\n'
    ),
    "__FACTOR_LOG__": (
        '            Log($"[INIT] rsi_period={_rsiPeriod} oversold={_oversold} overbought={_overbought}");\n'
    ),
    "__PRICE_WINDOW_EXPR__": "_rsiPeriod + 1",
    "__EXTRA_BUF_FIELDS__":   "",
    "__EXTRA_BUF_ENQUEUE__":  "",
    "__EXTRA_BUF_DEQUEUE__":  "",
    "__EXTRA_BUF_TOARRAY__":  "",
    "__FACTOR_COMPUTE_BODY__": """
            var n = prices.Length;
            if (n < _rsiPeriod + 1) return false;

            if (!_rsiInitialized)
            {
                double sumGain = 0.0, sumLoss = 0.0;
                for (int i = 1; i < n; i++)
                {
                    var change = prices[i] - prices[i - 1];
                    if (change > 0) sumGain += change;
                    else sumLoss += Math.Abs(change);
                }
                _prevGainEma = sumGain / _rsiPeriod;
                _prevLossEma = sumLoss / _rsiPeriod;
                _rsiInitialized = true;
            }
            else
            {
                var change = prices[n - 1] - prices[n - 2];
                var gain = change > 0 ? change : 0.0;
                var loss = change < 0 ? Math.Abs(change) : 0.0;
                _prevGainEma = (_prevGainEma * (_rsiPeriod - 1) + gain) / _rsiPeriod;
                _prevLossEma = (_prevLossEma * (_rsiPeriod - 1) + loss) / _rsiPeriod;
            }

            double rsi;
            if (_prevLossEma < 1e-12)
                rsi = 100.0;
            else
            {
                var rs = _prevGainEma / _prevLossEma;
                rsi = 100.0 - 100.0 / (1.0 + rs);
            }

            if (rsi < _oversold)
                rawSignal = (_oversold - rsi) / _oversold;
            else if (rsi > _overbought)
                rawSignal = -(rsi - _overbought) / (100.0 - _overbought);
            else
                rawSignal = 0.0;

            return true;
""",
}


def _compute_rsi_wilder(close: pd.DataFrame, period: int) -> pd.DataFrame:
    delta = close.diff()
    gain  = delta.clip(lower=0.0)
    loss  = (-delta).clip(lower=0.0)
    avg_gain = gain.ewm(com=period - 1, min_periods=period, adjust=False).mean()
    avg_loss = loss.ewm(com=period - 1, min_periods=period, adjust=False).mean()
    rs  = avg_gain / avg_loss.replace(0, np.nan)
    rsi = 100.0 - 100.0 / (1.0 + rs)
    rsi.iloc[:period] = np.nan
    return rsi


def build_signal(close: pd.DataFrame, params: Dict[str, Any], **_) -> pd.DataFrame:
    rsi_period = int(params.get("rsi_period", 14))
    oversold   = float(params.get("oversold",   30.0))
    overbought = float(params.get("overbought", 70.0))

    rsi    = _compute_rsi_wilder(close, rsi_period)
    signal = pd.DataFrame(0.0, index=close.index, columns=close.columns)
    signal[rsi < oversold]   = (oversold   - rsi[rsi < oversold])   / oversold
    signal[rsi > overbought] = -(rsi[rsi > overbought] - overbought) / (100.0 - overbought)
    signal[rsi.isna()]       = np.nan
    return signal.reindex_like(close)
```

</details>

插件写好后保存到一个临时路径供提交用，提交后再按 job_id 归档。

<details>
<summary>示例：主力资金流入因子（taker_buy_ratio_momentum）— 使用 taker 额外列</summary>

```python
import pandas as pd
import numpy as np
from typing import Any, Dict

FACTOR_TYPE = "taker_buy_ratio_momentum"

FACTOR_DEFAULT_PARAMS = {
    "window": 20,
}

FACTOR_SECTIONS = {
    "__FACTOR_DESCRIPTION__": "主力资金流入：taker 主动买量占比的滚动均值偏离中性线 0.5",
    "__FACTOR_FORMULA__":     "buy_ratio = taker_buy_vol / (buy+sell); signal = rolling_mean(buy_ratio, w) - 0.5",
    "__FACTOR_TYPE__":        "taker_buy_ratio_momentum",
    "__FACTOR_PARAM_FIELDS__": (
        "        private int _window;\n"
    ),
    "__FACTOR_INIT__": (
        '            _window = GetIntParameter("window", 20);\n'
    ),
    "__FACTOR_LOG__": (
        '            Log($"[INIT] window={_window}");\n'
    ),
    "__PRICE_WINDOW_EXPR__": "_window",
    # ── 额外列：taker_buy_volume / taker_sell_volume ──────────────────
    "__EXTRA_BUF_FIELDS__": (
        "        private readonly Queue<double> _takerBuyBuf  = new Queue<double>();\n"
        "        private readonly Queue<double> _takerSellBuf = new Queue<double>();\n"
    ),
    "__EXTRA_BUF_ENQUEUE__": (
        "            _takerBuyBuf.Enqueue(bar.TakerBuyVolume);\n"
        "            _takerSellBuf.Enqueue(bar.TakerSellVolume);\n"
    ),
    "__EXTRA_BUF_DEQUEUE__": (
        "            if (_takerBuyBuf.Count  > requiredBars) _takerBuyBuf.Dequeue();\n"
        "            if (_takerSellBuf.Count > requiredBars) _takerSellBuf.Dequeue();\n"
    ),
    "__EXTRA_BUF_TOARRAY__": (
        "            var takerBuys  = _takerBuyBuf.ToArray();\n"
        "            var takerSells = _takerSellBuf.ToArray();\n"
    ),
    "__FACTOR_COMPUTE_BODY__": """
            var n = prices.Length;
            if (n < _window) return false;

            double sumRatio = 0.0;
            for (int i = 0; i < n; i++)
            {
                var total = takerBuys[i] + takerSells[i];
                var ratio = total > 1e-12 ? takerBuys[i] / total : 0.5;
                sumRatio += ratio;
            }
            rawSignal = sumRatio / n - 0.5;
            return true;
""",
}


def build_signal(
    close:            pd.DataFrame,
    params:           Dict[str, Any],
    taker_buy_volume: pd.DataFrame,
    taker_sell_volume: pd.DataFrame,
    **_kwargs,
) -> pd.DataFrame:
    window = int(params.get("window", 20))
    total = taker_buy_volume + taker_sell_volume
    buy_ratio = taker_buy_volume / total.replace(0, float("nan"))
    signal = buy_ratio.rolling(window).mean() - 0.5
    return signal.reindex_like(close)
```

</details>

---

### 阶段 2：提交任务

```bash
# 先把 plugin 写到临时文件
cat > /tmp/current_plugin.py << 'PLUGIN_EOF'
<plugin 内容>
PLUGIN_EOF

curl -s -X POST ${BASE_URL}/jobs/submit \
  -F "factor_kind=custom" \
  -F "factor_type=<factor_type>" \
  -F "factor_name=<factor_name>" \
  -F "params=<JSON字符串，如 {\"rsi_period\":14}>" \
  -F "fwd_period=16" \
  -F "plugin=@/tmp/current_plugin.py"
```

成功返回：

```json
{ "job_id": "job_20260312_153001_f4a2c1", "status": "queued" }
```

拿到 `job_id` 后立即执行以下两步本地归档：

```bash
JOB_ID="job_20260312_153001_f4a2c1"

# 1. 更新 current_job_id.txt
echo "${JOB_ID}" > ~/.quant_agent/current_job_id.txt

# 2. 把 plugin.py 归档到 job 目录
mkdir -p ~/.quant_agent/jobs/${JOB_ID}
cp /tmp/current_plugin.py ~/.quant_agent/jobs/${JOB_ID}/plugin.py
```

> **builtin 因子**（`momentum` / `trend` / `mean_revert`）不需要上传 plugin，
> 改用 `factor_kind=builtin` 并省略 `-F "plugin=..."` 即可（无需归档 plugin.py）。

---

### 阶段 3：轮询进度

每 **15 秒**查询一次，最多等待 **30 分钟**：

```bash
JOB_ID=$(cat ~/.quant_agent/current_job_id.txt)
curl -s ${BASE_URL}/jobs/${JOB_ID}/status
```

返回示例：

```json
{
  "status": "running",
  "current_step": 10,
  "progress": "IS 超参网格搜索"
}
```

#### status 状态说明

| status | 含义 | Agent 行为 |
|--------|------|-----------|
| `queued` | 排队中 | 继续等待 |
| `running` | 执行中 | 告知用户当前步骤，继续等待 |
| `done` | 全部完成 | 进入阶段 5 获取结果 |
| `failed` | 工作流失败 | 读取 `error` 字段，判断是否可修复（进入阶段 4） |
| `retesting` | 正在 retest | 继续轮询 |
| `retest_failed` | retest 也失败 | 查看 retest 日志，再次修复 strategy.cs |

#### strategy_cs_ready 标志

轮询时若返回 `"strategy_cs_ready": true`，立即下载并归档 strategy.cs：

```bash
JOB_ID=$(cat ~/.quant_agent/current_job_id.txt)
mkdir -p ~/.quant_agent/jobs/${JOB_ID}
curl -s ${BASE_URL}/jobs/${JOB_ID}/files/strategy.cs \
  -o ~/.quant_agent/jobs/${JOB_ID}/strategy.cs
```

此操作只需执行一次（下载后不必重复）。

---

### 阶段 4：处理 Step 11 失败（C# 编译 / 运行时错误）

当 `status=failed` 且 `failed_step=11` 时执行此阶段。

#### Step 4-1：查看错误日志

```bash
JOB_ID=$(cat ~/.quant_agent/current_job_id.txt)
curl -s "${BASE_URL}/jobs/${JOB_ID}/logs?tail=80"
```

#### Step 4-2：下载当前 strategy.cs

```bash
mkdir -p ~/.quant_agent/jobs/${JOB_ID}
curl -s ${BASE_URL}/jobs/${JOB_ID}/files/strategy.cs \
  -o ~/.quant_agent/jobs/${JOB_ID}/strategy.cs
```

#### Step 4-3：分析错误并修复

根据日志中的错误信息修改 `~/.quant_agent/jobs/${JOB_ID}/strategy.cs`。

常见错误速查表：

| 错误信息 | 原因 | 修复方式 |
|---------|------|---------|
| `CS0019: Operator '/' cannot be applied to 'double' and 'decimal'` | C# 类型不匹配 | 在除法前加 `(double)` 强转，或检查变量声明类型 |
| `CS0103: The name 'xxx' does not exist` | 变量名拼写错误或作用域不对 | 检查 `__FACTOR_PARAM_FIELDS__` 中的声明 |
| `CS1002: ; expected` | C# 语法错误（缺分号等） | 检查 `__FACTOR_COMPUTE_BODY__` 的每行结尾 |
| `CS0012: Type 'DynamicObject' not referenced` | 调用了 `GetLastData()` | 删除 `Securities[].GetLastData()` 调用，改用传入的 `prices[]` |
| `rawSignal` 始终为 0 | `return true` 前忘记给 `rawSignal` 赋值 | 确保所有代码路径都给 `rawSignal` 赋值 |
| `return false` 导致无信号 | `__PRICE_WINDOW_EXPR__` 比实际需要大 | 调小 `__PRICE_WINDOW_EXPR__` 的值 |
| Lean 运行时 NullReference | 访问了未初始化的字段 | 检查 `__FACTOR_INIT__` 是否遗漏了某个字段初始化 |

**修复原则**：
- 只修改 `__FACTOR_COMPUTE_BODY__` 对应的 C# 代码块区域
- 如果是字段声明或初始化问题，找到 strategy.cs 中对应的 `#region` 块修改
- 修改后在脑内或注释中验证每条代码路径都能给 `rawSignal` 赋值

#### Step 4-4：提交 retest

```bash
curl -s -X POST ${BASE_URL}/jobs/${JOB_ID}/retest \
  -F "strategy_cs=@~/.quant_agent/jobs/${JOB_ID}/strategy.cs"
```

返回 `{ "status": "retesting" }` 后回到**阶段 3**继续轮询。

> retest 可以多次，每次服务器都会保留独立日志（`retest_001.log`、`retest_002.log`…）。
> 若连续 3 次 retest 仍失败，重新审视 plugin.py 的 `FACTOR_SECTIONS` 是否有根本性错误，
> 考虑重写 plugin.py 后重新 POST `/jobs/submit` 开新任务。

---

### 阶段 5：获取结果并下载文件

#### Step 5-1：获取 JSON 结果

```bash
JOB_ID=$(cat ~/.quant_agent/current_job_id.txt)
curl -s ${BASE_URL}/jobs/${JOB_ID}/result
```

返回字段说明：

| 字段 | 含义 | 关注点 |
|------|------|--------|
| `factor_status` | `"pass"` 或 `"fail"` | 最终结论 |
| `best_params` | 超参搜索最优参数 | zscore_window / use_ewma / ewma_span / sigmoid_c |
| `profile.ic_mean` | IC 均值 | > 0.03 是及格线 |
| `profile.icir` | IC 信息比 | > 0.5 较好 |
| `val_summary.median_sharpe` | VAL 段中位 Sharpe | > 1.0 是及格线 |
| `val_summary.win_rate` | 胜率 | > 0.55 较好 |
| `factor_card_txt` | 可读的因子档案卡全文 | 直接展示给用户 |

将 `factor_card_txt` 完整展示给用户，并用一段话总结因子的核心表现。

#### Step 5-2：下载任务产物文件

任务 done 后，下载并归档以下文件到 job 目录：

```bash
JOB_ID=$(cat ~/.quant_agent/current_job_id.txt)
JOB_DIR=~/.quant_agent/jobs/${JOB_ID}
mkdir -p ${JOB_DIR}/step11

# 因子档案卡
curl -s ${BASE_URL}/jobs/${JOB_ID}/files/factor_card.json \
  -o ${JOB_DIR}/factor_card.json

# 权益曲线图
curl -s ${BASE_URL}/jobs/${JOB_ID}/files/equity_curves.png \
  -o ${JOB_DIR}/step11/equity_curves.png

# 逐笔交易记录（完整下载）
curl -s ${BASE_URL}/jobs/${JOB_ID}/files/trade_log.csv \
  -o ${JOB_DIR}/step11/trade_log.csv
```

> strategy.cs 若在阶段3已下载，此处无需重复。若尚未下载，同样在此补下：
> `curl -s ${BASE_URL}/jobs/${JOB_ID}/files/strategy.cs -o ${JOB_DIR}/strategy.cs`

---

## 完整流程示意图

```
用户：「研究一个布林带宽度突破因子」
        │
        ▼
[阶段0] 确认 factor_type=boll_breakout, params={window:20}
        │
        ▼
[阶段1] 写 /tmp/current_plugin.py
        │  包含 FACTOR_SECTIONS（C#片段）+ build_signal()（Python）
        │
        ▼
[阶段2] POST /jobs/submit -F plugin=@/tmp/current_plugin.py
        │  → 拿到 job_id，保存到 current_job_id.txt
        │  → 归档 plugin.py 到 ~/.quant_agent/jobs/{job_id}/plugin.py
        │
        ▼
[阶段3] 每15秒 GET /jobs/{id}/status 轮询
        │
        ├─ strategy_cs_ready=true
        │       └─ GET /files/strategy.cs → 保存到 jobs/{job_id}/strategy.cs
        │
        ├─ status=running ──→ 告知用户当前 Step，继续等待
        │
        ├─ status=failed, failed_step=11
        │       │
        │       ▼
        │  [阶段4] GET logs → GET /files/strategy.cs → 修复 → POST retest
        │       │
        │       └─ 回到阶段3继续轮询
        │
        └─ status=done
                │
                ▼
        [阶段5] GET /result → 展示 factor_card_txt 给用户
                │
                ▼
             下载 factor_card.json / equity_curves.png / trade_log.csv
             → 归档到 ~/.quant_agent/jobs/{job_id}/
```

---

## 其他接口

### 查看 retest 日志

```bash
curl -s "${BASE_URL}/jobs/${JOB_ID}/retest_logs?tail=100"
```

### 健康检查

```bash
curl -s ${BASE_URL}/health
```

返回中 `config_exists: true` 且 `klinedata_exists: true` 表示服务器数据就绪。

---

## 向用户汇报进度的节奏

- 提交任务后立即告知 job_id
- 每 2~3 次轮询后向用户报告一次当前 Step（不必每次都说）
- Step 11 开始时特别说明「正在提交 Lean 云端回测，约需 3~5 分钟」
- 遇到 retest 时告知用户「Step 11 编译失败，正在修复 C# 代码后重试」，不要抛出错误
- 最终结果中重点突出 `factor_status`（pass/fail）、`median_sharpe`、`icir` 三个核心指标
