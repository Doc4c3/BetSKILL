# 竞彩足球混合过关优化工作流

## 概述

本 Skill 定义了一套完整的竞彩足球混合过关投注方案生成流水线，涵盖从情报收集到最终推荐的端到端流程。核心方法论为：

> **候选池 → 主观概率 → 组合枚举 → 约束优化 → 表格化呈现**

---

## 第一阶段：信息收集

### 1.1 工具环境

```bash
pip install shin soccerdata
```

- `shin`：Shin(1992) 赔率去水算法，输入赔率→输出去 overround 后的真实隐含概率
- `soccerdata`：一站式拉取 FBref（xG/射门/控球）、ClubElo（实力评分）、WhoScored（比赛统计）

### 1.2 OCR 赔率提取范式

> ⚠️ DeepSeek V4 Pro 不支持原生视觉，`ReadMediaFile` 不可用。以下为适配方案。

#### 主路径：用户粘贴模板（最可靠）

用户用微信/系统 OCR 提取截图文本后，按**固定模板**粘贴。一行一场，之间空行分隔：

```
场次编号 主队 VS 客队
胜平负: 主胜赔率 平赔率 主负赔率
让球(+N或-N): 主胜赔率 平赔率 主负赔率
比分: 1:0=赔率, 2:0=赔率, 2:1=赔率, 0:0=赔率, 1:1=赔率, 0:1=赔率, 1:2=赔率, 0:2=赔率, ...
```

**粘贴提示词**（直接复制发给用户）：
> 请用微信截图 → 长按提取文字 → 全选复制粘贴给我。格式如下，一场一段：
> ```
> 061 挪威 VS 法国
> 胜平负: 4.65 3.90 1.52
> 让球(+1): 2.18 3.45 2.63
> 比分: 1:0=16.00, 2:0=28.00, 2:1=12.50, 0:0=16.00, 1:1=6.90, 0:1=8.30, 1:2=6.50, 0:2=8.25, ...
> ```

#### 可选自动化：Tesseract CLI

```bash
# Windows 需预装 Tesseract-OCR (https://github.com/UB-Mannheim/tesseract/wiki)
tesseract screenshot.jpg stdout -l chi_sim+eng
```
输出走解析引擎层处理。

#### 解析引擎

无论来源，最终文本统一走以下 Python 解析器：

```python
import re

def parse_odds_text(text: str) -> dict:
    """
    输入：模板格式文本
    输出：{match_id: {home, away, odds_1x2, handicap, scores}}
    """
    matches = {}
    blocks = text.strip().split('\n\n')
    
    for block in blocks:
        header = re.match(r'(\d+)\s+(.+?)\s+VS\s+(.+)', block)
        if not header: continue
        mid, home, away = header.groups()
        
        m_1x2 = re.search(r'胜平负[：:]\s*([\d.]+)\s+([\d.]+)\s+([\d.]+)', block)
        m_hc = re.search(r'让球\(([+-]\d+)\)[：:]\s*([\d.]+)\s+([\d.]+)\s+([\d.]+)', block)
        scores_raw = re.findall(r'(\d+):(\d+)=(\d+\.?\d*)', block)
        
        matches[mid] = {
            'home': home.strip(),
            'away': away.strip(),
            'odds_1x2': [float(x) for x in m_1x2.groups()] if m_1x2 else None,
            'handicap': {
                'line': int(m_hc.group(1)),
                'odds': [float(x) for x in m_hc.groups()[1:]]
            } if m_hc else None,
            'scores': {f'{h}:{a}': float(o) for h, a, o in scores_raw}
        }
    return matches
```

#### OCR 验证规则

| 检查 | 规则 | 失败处理 |
|------|------|----------|
| 赔率范围 | 1.01 < odds < 1000 | 标记异常，请求用户复核 |
| booksum | Σ(1/odds) ∈ [1.05, 1.30] | 超标=OCR 漏字/多字 |
| 比分完整性 | 至少含 0:0, 1:0, 0:1, 1:1, 2:0, 0:2 | 缺失=截图不完整 |
| 让球方向 | (±1) 或 (±2)，与对阵强弱一致 | 方向反=请用户确认 |
| 场次编号 | 连续、不重复 | 跳号=缺截图 |

### 1.3 赛前情报搜索（双通道）

**通道 A：Tavily 文本搜索**（现有）

对每一场比赛，使用 `mcp__tavily__tavily_search` 搜索：

```
"{主队} vs {客队} World Cup 2026 preview prediction odds team news form tactics"
```
- `max_results: 5-8`, `search_depth: "advanced"`
- 优先信任：WhoScored, RotoWire, SportsMole, RacingPost, Goal.com, Yahoo Sports
- **必须引用 ≥2 个独立来源**，仅 1 源或来源方向矛盾 → 情报因子中性化(=1.0)

**通道 B：结构化数据**（新增）

使用 `soccerdata` 拉取量化指标（一行代码）：

```python
import soccerdata as sd
# FBref 球队赛季数据：xG, 射门, 控球率, 传球成功率
fbref = sd.FBref(leagues="World Cup", seasons=2026)
stats = fbref.read_team_season_stats()
# ClubElo 实力基线
elo = sd.ClubElo()
ratings = elo.read_team_ratings()
```

**情报因子对应关系**：

| 情报维度 | 文本通道（Tavily） | 结构化通道（soccerdata） | 量化方式 |
|----------|---------------------|--------------------------|----------|
| 小组形势 | 积分榜、出线条件 | — | 文本摘要 |
| 近期状态 | 近5场战绩描述 | `Goals`, `xG`, `xGA` | 近5场 xG 差 → 攻击力偏移 |
| 战术风格 | 阵型、进攻方式 | `Passes`, `Pressures`, `Aerial Duels` | 传球数/压迫数/争顶率 |
| 综合判断 | 机构预测比分 | `ClubElo` 评分 | Elo差÷200 → 理论胜率偏移 |
| 关键伤停 | 核心球员缺阵报道 | WhoScored 预览（soccerdata 封装） | 文本为主，结构化辅助 |
| 爆冷风险 | 轮换可能、战意 | — | 文本摘要 |

### 1.4 子 Agent 任务（可选，用于深度分析）

当需要更深入的分析时，使用 `Agent` 启动 explore 子代理：

```
Agent(
  subagent_type="explore",
  prompt="搜索并汇总 {主队} vs {客队} 的赛前情报，包括：小组形势、伤停、战术风格、近期状态、机构预测。输出结构化表格。",
  description="{主队}vs{客队}"
)
```

**注意**：子代理可能因网络超时失败，此时应回退到直接 Tavily 搜索。

---

## 第二阶段：构建候选池与主观概率

### 2.1 候选池构建

从赔率截图中提取所有可用选项，分类为：

- **胜平负**：三向结果
- **让球胜平负**：带让球数的三向结果
- **比分**：所有比分及赔率

### 2.2 主观概率赋值（三步管线）

**核心公式**：

```
主观概率 = Shin去水概率 × 结构化信号 × 文本信号
```

**Step 1：赔率去水（替换朴素 1/odds）**

```python
from shin import calculate_implied_probabilities
# 输入竞彩胜平负赔率
fair_probs = calculate_implied_probabilities([主胜赔, 平赔, 客负赔])
# → [0.373, 0.280, 0.347]  # 已去除 overround
```

Shin 方法自动去除 bookmaker 的 overround（抽水），比 `1/赔率 ÷ booksum` 更严谨。
Shin 同时输出 `z` 值（内幕交易者比例估计）：
- `z < 0.07` → 市场效率可接受，赔率可用
- `z > 0.10` → 极端不确定性，标记为高风险场次（回测：2026-06-26 六场全 z>0.02 但方向全对，阈值需放宽）

**Step 2：结构化信号（量化）**

```python
# Elo 差值 → 理论胜率偏移
elo_home = elo.read_team_ratings()["home_team"]
elo_away = elo.read_team_ratings()["away_team"]
elo_signal = 1.0 + (elo_home - elo_away) / 200  # 例: +100 → ×1.05

# xG 差值 → 攻击力偏移
xG_signal = 1.0 + (xG_home_mean - xG_away_mean) * 0.5  # 近5场均值差

结构化信号 = elo_signal × xG_signal
```

**Step 3：文本信号（Tavily）**

```
文本信号 = 战意因子 × 伤停因子 × 舆论因子
  战意因子: 1.0 ± 0.2  (出线形势)
  伤停因子: 1.0 ± 0.25 (核心缺阵；TOP2球员同时缺阵→取极值±0.25)
  舆论因子: 1.0 ± 0.1  (媒体预测方向)
  均值回归因子: 1.0 ± 0.15 (连续N场低于xG→反弹修正；如比利时2场0进球→上调攻击力预期)
```

**关键规则**：当结构化信号与文本信号矛盾时（如文本说「状态火热」但 xG 差<0.2），**降权文本信号 ×0.5**，以结构化数据为准。

### 2.2.1 交叉验证防投毒（3 规则）

每条概率赋值前，必须通过以下检查：

```
规则 1: Shin去水 vs 加法去水 — 差>3% → 标记「赔率噪声大」
规则 2: ClubElo胜率 vs 竞彩隐含胜率(去水后) — 差>15% → 标记「市场vs数据分歧」
规则 3: Tavily 至少引用 ≥2 个独立来源 — 单源/来源矛盾 → 文本信号中性化(=1.0)
```

**防投毒逻辑**：任何单一信源异常都会被另外 1-2 个信源交叉暴露。

**情报修正因子参考（保留兼容）**：

| 因子 | 上调（×1.1~1.3） | 下调（×0.7~0.9） |
|------|------|------|
| 战意 | 必须赢才能出线 | 已出线可能轮换 |
| 状态 | 近期连胜/火力全开 | 连续不胜/进攻哑火 |
| 伤停 | 对手核心缺阵 | 本方核心缺阵 |
| 盘口 | 机构让球偏保守 | 机构让球偏激进 |
| 爆冷 | 弱队有逼平强队历史 | 强弱过于悬殊 |

**概率精度**：保留到 1%（如 0.14），不追求过度精确。

### 2.3 候选池数据结构

```javascript
const picks = {
  matchId: [
    ["场次 市场 选项名称", 赔率, 主观概率],
    // 每个match至少包含：
    // - 让球选项（+1/+2方向）
    // - 胜平负选项
    // - 最可能1-3个比分
    // - 排除概率<5%的极端选项
  ]
};
```

**排除规则**：
- 概率 < 5% 的极端比分（如 5:0, 4:2 等）
- 明显与现实矛盾的选择（如弱队赢强队多个球）
- 但保留「低概率仍可能」的选项（如冷门平局、一球小胜）

---

## 第二阶段·附加：多玩家预测结果拟合

当有多个玩家的独立选单需要合并时，在原候选池基础上叠加**共识权重**。

### 2A.1 数据结构

```
玩家1: { 场次ID: [选项1, 选项2...] }
玩家2: { 场次ID: [选项1, 选项2...] }
...
```

### 2A.2 共识矩阵

对每场比赛，统计所有玩家的选择并集：

| 场次 | 选项 | P1 | P2 | P3 | 共识数 | 赔率 |
|------|------|----|----|----|--------|------|
| 055 | +1 平 | ✓ |   | ✓ | 2 | 3.45 |
| 055 | +1 主负 |   | ✓ |   | 1 | 2.35 |
| 056 | +2 平 | ✓ | ✓ |   | 2 | 3.85 |
| 056 | +2 主胜 |   |   | ✓ | 1 | 2.40 |

### 2A.3 共识权重因子

```
调整后概率 = 原主观概率 × 共识权重因子

共识权重因子：
  3/3 玩家一致  → ×1.45
  2/3 玩家一致  → ×1.15
  1/3 玩家单独  → ×1.00（无加成）
```

**注意**：共识因子只调整概率，不改变赔率。赔率保持原始竞彩数据。

### 2A.4 合并策略

**策略一：高概率优先合并（保稳票）**

对每场，在玩家并集中优先选择：
1. **赔率最低的选项**（隐含概率最高），优先级最高
2. 若多个选项赔率接近，选**共识数最高**的
3. 形成候选池后，正常走枚举→约束优化

对应代码逻辑：
```javascript
// 只筛选 ≥2 人共识的选项进入保稳候选池
const safePool = rawPicks[matchId].filter(x => x.consensus >= 2);
// 若某场无 ≥2 共识选项，沿用全量候选池
if (safePool.length === 0) safePool = rawPicks[matchId];
```

**策略二：高赔率次级合并（刺激票）**

对每场的「非保稳选项」中：
1. 优先选择**赔率最高**但有**至少2人共识**的选项
2. 若无2人共识选项，选赔率最高但**信息来源最可靠**的
3. 允许少量1人选项，前提是对应玩家历史命中率较高，或有特定情报支撑

对应代码逻辑：
```javascript
// 刺激候选池：取每个match共识≥2的高赔选项 + 保稳池未用的剩余高赔
const highOddsPool = rawPicks[matchId].filter(x => 
  x.consensus >= 2 && x.odds >= 3.0
);
// 如果某场没有满足的，回退到1人共识+赔率最高
if (highOddsPool.length === 0) {
  highOddsPool = rawPicks[matchId]
    .filter(x => x.odds >= 3.0)
    .sort((a,b) => b.odds - a.odds)
    .slice(0, 1);
}
```

### 2A.5 四人及以上扩展

| 玩家数 | 3/3 + | 2/3 + | 信号判定 |
|--------|--------|--------|----------|
| 3人 | 3/3 → ×1.45 | 2/3 → ×1.15 | 2/3=可靠 |
| 4人 | 4/4 → ×1.50 | 3/4 → ×1.20 | 3/4=可靠 |
| 5人 | 5/5 → ×1.50 | 3+/5 → ×1.15 | 3+/5=可靠 |
| N人 | N/N → ×1.50 | ≥ceil(N/2) → ×1.10 | 过半=可靠 |

### 2A.6 多人拟合的完整算法

```
输入：N个玩家的选单（每场1-2个选项），M场比赛
输出：保稳候选池、刺激候选池

1. 按场次归并玩家选择 → 共识矩阵
2. 为每个选项标注共识数
3. 计算原始主观概率 = 1/赔率 × 情报修正因子
4. 应用共识权重：prob' = prob × 共识因子
5. 保稳池 = 每场共识≥ceil(N/2)的选项
6. 刺激池 = 每场赔率≥3.0且共识≥2的选项，不足则取单人高赔
7. 两个池分别进入枚举→约束优化流水线
```

### 2A.7 实际案例（来自经典会话）

```
三人选单合并：
  P1: 055 +1平3.45  P2: 055 +1主负2.35  P3: 055 +1平3.50
  → 平共识2/3, 主负共识1/3
  → 保稳选主负（赔率更低2.35），刺激选平（赔率更高且2人共识）

  P1/P2: 057 +2主负1.80  P3: 057平3.90+客胜2.05
  → 主负/客胜方向 3/3共识
  → 保稳=客胜1.80，刺激=平3.90（P3的冷门高赔选择）
```

---

## 第三阶段：组合枚举与约束优化

### 3.1 枚举算法

对 N 场比赛、每场 M 个选项：

```javascript
// 选 K 场比赛的组合
for (const matchCombination of combinations(allMatches, K)) {
  // 笛卡尔积：每场选一个选项
  for (const selection of cartesianProduct(matchCombination)) {
    const sp = product(selection.map(s => s.odds));
    const prob = product(selection.map(s => s.probability));
    results.push({ sp, prob, selection });
  }
}
```

### 3.2 约束条件定义

**保稳单（Conservative）**：

| 参数 | 含义 | 典型值 |
|------|------|------|
| SP 下限 | 最低赔率（太低不够刺激） | 15 |
| SP 上限 | 最高赔率（太高不稳） | 50 |
| 串关数 K | 选择的场次数 | 3~4 |
| 优化目标 | **最大化命中概率** | `max(prob) where sp∈[15,50]` |

**刺激单（Aggressive）**：

| 参数 | 含义 | 典型值 |
|------|------|------|
| 单场概率下限 | 每个选项最低主观概率 | 8% |
| 整体概率下限 | 组合最低命中概率 | 0.03%~0.05% |
| 串关数 K | 选择的场次数 | 4 |
| 优化目标 | **最大化 SP** | `max(sp) where prob≥threshold` |

### 3.3 概率阈值对标

| 整体命中概率 | 约等于 | 适用场景 |
|-------------|--------|----------|
| 0.10%（1/1000） | 略激进 | 冲高单下限 |
| 0.05%（1/2000） | 标准 | 对标经典1354倍单 |
| 0.03%（1/3300） | 较激进 | 追求更高SP |
| 0.01%（1/10000） | 极激进 | 一般不推荐 |

### 3.4 边界检测

优化结果需通过以下边界检查：

- ❌ SP 过高（>5000）且命中 <0.01% → 退回，提高概率下限
- ❌ 四个选项全是极端比分（如 3:2×3:1×2:3...）→ 退回，加入非比分选项
- ❌ 保稳单用了两个以上比分选项 → 警示，可能太激进
- ✅ SP 适中（保稳15-50，刺激200-3000）+ 每条有情报支撑 → 通过

---

## 第四阶段：表格化呈现

### 4.1 情报汇总表

```
| 场次 | 对阵 | 小组形势 | 近期状态 | 关键伤停 | 战术风格 | 机构风向 | 综合判断 |
```

### 4.2 赔率矩阵表

```
| 场次 | 时间 | 对阵 | 胜平负 | 让球盘 | 让球主胜/平/主负 |
```

### 4.3 推荐单格式

```
## 🛡️ 保稳单 / 🚀 刺激单

| # | 场次 | 选项 | 赔率 | 逻辑/现实锚点 |
|---|------|------|------|-------------|
| 1 | ... | ... | ... | 一句话情报支撑 |

> SP = 全部乘积 | 命中概率 ≈ X% | 2元 → X元
```

### 4.4 汇总对比表

```
| | 场次 | 玩法结构 | SP | 命中估概 | 2元→ |
```

---

## 第五阶段：推理纠错与动态调整

### 5.1 用户反馈驱动迭代

| 用户反馈 | 调整动作 |
|----------|----------|
| "SP太低" | 提高SP下限，或降低串关数 |
| "SP太高不现实" | 提高整体概率下限 |
| "加入新场次" | 重新构建候选池，重新枚举 |
| "只要N串1" | 固定K值 |
| "不限定玩法" | 候选池混入胜平负+让球+比分 |
| "基于现实" | 增加单场概率下限，排除极端选项 |
| "成功概率最高" | 优化目标从 max(SP) 切换为 max(prob) |

### 5.2 常见纠错

1. **赔率单位混淆**：亚洲盘 vs 竞彩赔率不同，统一使用竞彩赔率
2. **让球方向搞反**：`主队+1` 表示主队受让1球
3. **SP区间过窄无解**：扩大区间或减少串关数
4. **新比赛缺情报**：立即启动 Tavily 并行搜索补充

---

## 第六阶段：提示词优化

### 6.1 启动提示词模板

```
基于以下比赛的赔率截图和情报报告，使用候选池→主观概率→组合枚举→约束优化的方法论，
给出两张推荐票：
1. 保稳单：SP ∈ [15, 50]，最大化命中概率
2. 刺激单：不设SP上限但每条基于现实（单场概率≥8%，整体概率≥0.03%）
```

### 6.2 情报搜索模板

```
搜索 "{主队} vs {客队} World Cup 2026 preview prediction odds team news form tactics"
重点提取：小组形势、出线条件、近期战绩、伤停名单、战术风格、机构预测比分
```

### 6.3 用户交互模板

- 首次推荐后主动询问："SP范围合适吗？要不要调整串关数？"
- 追加比赛时："新增X场，候选池已更新，需要我重新跑优化吗？"
- 交付时附风险提示："4串1本身难中，请理性购彩"

---

## 执行清单

### 单人模式（直接分析）
- [ ] `pip install shin soccerdata`
- [ ] 读取赔率截图或手动OCR文本 → 结构化存储
- [ ] 对每场比赛并行 Tavily 搜索（6场=6个并行调用）
- [ ] `soccerdata` 拉取 FBref xG/控球率 + ClubElo 评分
- [ ] Shin 去水：`calculate_implied_probabilities([h,d,a])`
- [ ] 构建候选池：每场3-6个选项（胜平负 + 让球 + 1-3个比分）
- [ ] 赋值主观概率：`Shin去水概率 × 结构化信号 × 文本信号`
- [ ] 交叉验证：Shin对比、Elo对比、Tavily多源 ≥2
- [ ] → 跳至枚举优化

### 多人模式（合并拟合）
- [ ] 读取多个玩家的选单 → 结构化存储
- [ ] 对每场并集玩家的选择 → 共识矩阵
- [ ] 标注共识数（N/3、2/3…）
- [ ] 构建候选池：保稳池=共识≥2的选项、刺激池=共识≥2且赔率≥3的选项
- [ ] 赋值主观概率 = `Shin去水概率 × 结构化信号 × 文本信号 × 共识权重`
- [ ] → 跳至枚举优化

### 通用：枚举 + 优化
- [ ] 枚举所有 K 关组合，计算 SP 和整体命中概率
- [ ] 保稳单：筛选 SP∈[15,50]，取 max(概率)
- [ ] 刺激单：筛选单场概率≥8%且整体≥0.03%，取 max(SP)
- [ ] 边界检测：是否有极端选项、SP是否过分离谱
- [ ] 表格化呈现，每个选项附一句话现实锚点
- [ ] 输出汇总对比表
- [ ] 根据用户反馈迭代调整

---

## 代码工具

**概率前置（Python）**：

```python
# 赔率去水
from shin import calculate_implied_probabilities
fair = calculate_implied_probabilities([h_odds, d_odds, a_odds])
z = fair.z  # 内幕交易比例，<0.07=可用

# 结构化数据
import soccerdata as sd
elo = sd.ClubElo().read_team_ratings()
fbref = sd.FBref(leagues="World Cup", seasons=2026).read_team_season_stats()

# 概率合成
base_prob = fair[i]  # Shin 去水后的第 i 个方向概率
structured_signal = (1 + elo_diff/200) * (1 + xG_diff*0.5)
final_prob = base_prob * structured_signal * text_signal
```

**枚举优化（Node.js）**：

```javascript
// 模板：枚举 + 过滤 + 排序
const results = [];
for (const ms of combos(matchIds, K)) {  // 选K场
  for (const sel of cartesianProduct(ms)) {  // 每场选一个
    const sp = sel.reduce((a,x) => a * x[1], 1);
    const prob = sel.reduce((a,x) => a * x[2], 1);
    results.push({ sp, prob, sel });
  }
}
// 保稳：SP∈[15,50], 按概率降序
const safe = results.filter(r => r.sp >= 15 && r.sp <= 50)
                    .sort((a,b) => b.prob - a.prob);
// 刺激：单场≥8%, 整体≥0.0003, 按SP降序
const aggressive = results.filter(r => 
  r.sel.every(x => x[2] >= 0.08) && r.prob >= 0.0003
).sort((a,b) => b.sp - a.sp);
```

### 多人模式：共识矩阵构建

```javascript
// 1. 解析每个玩家的选单
const playerPicks = {
  P1: { 55: ["+1平@3.45"], 56: ["+2平@3.88"], ... },
  P2: { 55: ["+1主负@2.35"], 56: ["+2平@3.88"], ... },
  P3: { 55: ["+1平@3.50"], 56: ["+2主胜@2.40"], ... },
};

// 2. 按场次合并，标注共识数
const consensusMatrix = {};
for (const [player, matches] of Object.entries(playerPicks)) {
  for (const [matchId, selections] of Object.entries(matches)) {
    if (!consensusMatrix[matchId]) consensusMatrix[matchId] = {};
    for (const sel of selections) {
      const key = sel.split('@')[0]; // 选项名
      if (!consensusMatrix[matchId][key]) {
        consensusMatrix[matchId][key] = { count: 0, odds: parseFloat(sel.split('@')[1]) };
      }
      consensusMatrix[matchId][key].count++;
    }
  }
}

// 3. 共识权重因子
const CONSENSUS_WEIGHT = { 3: 1.45, 2: 1.15, 1: 1.00 };

// 4. 作为候选池概率的乘数融入
for (const [matchId, options] of Object.entries(consensusMatrix)) {
  for (const [name, data] of Object.entries(options)) {
    data.adjustedProb = (1/data.odds) * CONSENSUS_WEIGHT[data.count];
  }
}
```

### 混合模式：情报 + 共识双重修正

```javascript
// 修订后的主观概率公式（多人模式）
主观概率 = Shin去水概率 × 结构化信号 × 文本信号 × 共识权重因子

// 结构化信号：Elo差值 + xG差值（量化，来自 soccerdata）
// 文本信号：战意/伤停/舆论（Tavily，≥2源交叉）
// 共识权重因子（v1.3: 回测验证 3/3=5/6全中，上调至1.45）
// 基于玩家投票（1/3→×1.0, 2/3→×1.15, 3/3→×1.45）
```

---

> **版本**: v1.3

---

## 回测验证：2026-06-26 六场

| 场次 | 实际 | 单人保稳 | 单人刺激 | 三人6串1 |
|------|------|:--:|:--:|:--:|
| 061 挪威vs法国 | 1-4 | France Win ✅ | 1-2 ❌ | +1主负 ✅ |
| 062 塞内加尔vs伊拉克 | 5-0 | — | 2-0 ❌ | -2主胜 ✅ |
| 063 佛得角vs沙特 | 0-0 | — | — | -1客胜 ✅ |
| 064 乌拉圭vs西班牙 | 0-1 | Uru+1 ❌ | 0-1 ✅ | 主负 ✅ |
| 065 埃及vs伊朗 | 1-1 | Egypt Win ❌ | 1-0 ❌ | 主胜 ❌ |
| 066 新西兰vs比利时 | 1-5 | NZ+2 ❌ | — | +2主负 ✅ |

```
三人合并: 5/6 ✅   单人保稳: 1/4   单人刺激: 1/4
```

### 关键教训

1. **伤停信息是最强信号** — Haaland+Odegaard 轮休直接导致法国 4-1 而非 1-2（因子从 ±0.15 → ±0.25）
2. **三人共识比单人判断准得多** — 三人一致的方向 4/4 全对（3/3 权重上调至 ×1.45）
3. **「进攻乏力」需要均值回归修正** — 比利时 2 场 0 球后轰 5 球，新增「均值回归因子」
4. **Shin z 阈值太严** — 六场 z 全 >0.02 但方向全对，放宽至 z>0.07
5. **比分概率纯主观太粗糙** — 6 场比分仅中 1 场，泊松模型（penaltyblog）优先级提前  
> **适用场景**: 竞彩足球混合过关（世界杯/联赛），支持单人分析和多人合并  
> **依赖**: `shin`（赔率去水）、`soccerdata`（结构化数据）、Tavily MCP（文本情报）  
> **注意**: 所有推荐仅供娱乐参考，理性购彩
