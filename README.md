# 🎲 BetSKILL — 竞彩足球混合过关智能优化引擎

> **Candidate Pool → Subjective Probability → Combinatorial Enumeration → Constrained Optimization → Tabular Presentation**

一个基于数学建模的竞彩足球混合过关投注方案生成器。从赔率逆向推导真实概率，融合结构化数据与实时情报，通过组合枚举 + 约束优化输出最优串关方案。

---

## 🧠 核心理念

博彩公司用 overround（抽水）隐藏真实概率。BetSKILL 做三件事：

1. **去水** — Shin (1992) 算法逆向还原公平概率
2. **量化** — Elo 评级 + xG 数据替代主观拍脑袋
3. **优化** — 全量枚举所有 K 关组合，按约束条件选最优

```
竞彩赔率 → Shin 去水 → 结构化信号(Elo/xG) × 文本信号(Tavily) → 候选池 → 枚举 → 优化 → 推荐单
```

---

## 📦 依赖

```bash
pip install shin soccerdata
```

| 包 | 作用 | 许可 |
|----|------|------|
| [`shin`](https://github.com/mberk/shin) | 赔率去水，Shin (1992) 隐含概率模型 | MIT |
| [`soccerdata`](https://github.com/probberechts/soccerdata) | FBref + ClubElo + WhoScored 结构化数据 | MIT |

文本情报通过 Tavily MCP 实时搜索（SportsMole / RotoWire / WhoScored 等）。

---

## 🚀 快速开始

### 1. 输入赔率

从竞彩截图手动 OCR 赔率文本，格式如下：

```
061 挪威 VS 法国
胜平负: 4.65 / 3.90 / 1.52
让球(+1): 2.18 / 3.45 / 2.63
比分: 1:0=16.00, 2:0=28.00, 1:1=6.90, 0:1=8.30, 1:2=6.50, ...
```

### 2. 运行分析流水线

```python
from shin import calculate_implied_probabilities
import soccerdata as sd

# Step 1: 赔率去水
fair = calculate_implied_probabilities([4.65, 3.90, 1.52])
# → [0.195, 0.232, 0.573]  # 已去除 overround

# Step 2: 结构化数据
elo = sd.ClubElo().read_team_ratings()
elo_signal = 1.0 + (elo_home - elo_away) / 200

fbref = sd.FBref(leagues="World Cup", seasons=2026)
stats = fbref.read_team_season_stats()
xG_signal = 1.0 + (xG_home - xG_away) * 0.5

# Step 3: 合成概率
final_prob = fair[i] * elo_signal * xG_signal * text_signal
```

### 3. 枚举优化（Node.js）

```javascript
// 对 N 场, 每场 M 个候选 → 笛卡尔积枚举所有 K 关组合
for (const combo of combinations(matches, K)) {
  for (const selection of cartesianProduct(combo)) {
    const sp = selection.reduce((a, x) => a * x.odds, 1);
    const prob = selection.reduce((a, x) => a * x.probability, 1);
    results.push({ sp, prob, selection });
  }
}
// 保稳单: SP ∈ [15, 50], 最大化命中概率
// 刺激单: 单场概率 ≥ 8% 且整体 ≥ 0.03%, 最大化 SP
```

---

## 📊 输出示例

### 🛡️ 保稳单

| # | 场次 | 选项 | 赔率 | 逻辑 |
|---|------|------|------|------|
| 1 | 061 挪威(+1) VS 法国 | 主胜 | 2.18 | 法国轮换+挪威有球 |
| 2 | 064 乌拉圭(+1) VS 西班牙 | 主胜 | 2.70 | 西班牙终结软 |
| 3 | 065 埃及 VS 伊朗 | 主胜 | 2.32 | 萨拉赫状态火 |
| 4 | 066 新西兰(+2) VS 比利时 | 主胜 | 2.39 | 比利时两场0运动战进球 |

> **SP = 32.64** | 命中概率 ≈ 3.38% | 2 元 → 65 元

### 🚀 刺激单

| # | 场次 | 选项 | 赔率 | 逻辑 |
|---|------|------|------|------|
| 1 | 062 塞内加尔 VS 伊拉克 | 比分 2:0 | 5.50 | 共识比分 |
| 2 | 063 佛得角 VS 沙特 | 比分 1:1 | 5.50 | 报告首选 |
| 3 | 064 乌拉圭 VS 西班牙 | 比分 0:1 | 6.25 | 西班牙小胜 |
| 4 | 065 埃及 VS 伊朗 | 比分 1:0 | 6.50 | 一球封喉 |

> **SP = 1,229** | 命中概率 ≈ 0.053% | 2 元 → 2,458 元

---

## 🧬 方法论

### 概率公式

```
主观概率 = Shin(赔率) × 结构化信号 × 文本信号 × 共识权重
                                    ↑              ↑
                              Elo差 + xG差     Tavily ≥2源交叉
```

### 交叉验证防投毒

| 规则 | 检查 |
|------|------|
| 1 | Shin 去水 vs 加法去水 → 差 > 3% → 噪声报警 |
| 2 | ClubElo vs 竞彩隐含 → 差 > 15% → 市场数据分歧 |
| 3 | Tavily 至少 ≥2 独立来源 → 单源 → 信号中性化 |

### 多人共识拟合

当多个玩家独立选单时，自动构建共识矩阵并加权：

```
3/3 玩家一致 → 概率 ×1.30
2/3 玩家一致 → 概率 ×1.15
1/3 玩家单独 → 概率 ×1.00（无加成）
```

---

## 🗂 项目结构

```
bet/
├── SKILL.md              # 完整工作流定义（AI Agent 可直接读取执行）
├── README.md             # 本文件
├── .gitignore            # 排除截图/会话数据
└── 2026626/all/          # 赔率数据（已 gitignore）
```

---

## 🔗 相关项目

| 项目 | 作用 | 与本项目的关联 |
|------|------|:--:|
| [`mberk/shin`](https://github.com/mberk/shin) | 赔率去水 | 概率计算第一步 |
| [`probberechts/soccerdata`](https://github.com/probberechts/soccerdata) | 结构化体育数据 | 量化信号源 |
| [`martineastwood/penaltyblog`](https://github.com/martineastwood/penaltyblog) | 泊松比分模型 | 计划 v1.3 引入 |
| [`JaseZiv/worldfootballR`](https://github.com/JaseZiv/worldfootballR) | R 版数据爬虫 | 备选方案 |

---

## ⚠️ 免责声明

本项目仅供娱乐和学术研究。**不构成任何投注建议**。竞彩有风险，请理性购彩。所有概率均为数学模型估计，不代表实际赛果。

---

## 📄 许可

MIT License
