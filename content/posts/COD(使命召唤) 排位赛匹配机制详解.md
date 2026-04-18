---
title: COD(使命召唤)-排位赛匹配机制详解
date: 2025-04-21 09:40:45
tags: ["前后端", "其他技术"]
---

## 前言

当使奴太久了，从cod8 - cod21，前几天卖了PS5，打算彻底脱坑，再也不玩了

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%8E%92%E4%BD%8D%E8%B5%9B%E5%8C%B9%E9%85%8D%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/image-7.png)


之前被折磨的太多了，现在真的不想玩了，动视的匹配机制真的是个迷

有时候打一把，狂杀50人+，第二把露头就死，第三把杀人个位数，总感觉煞笔动视控制了什么

对面杀我和杀孙子一样

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%8E%92%E4%BD%8D%E8%B5%9B%E5%8C%B9%E9%85%8D%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/image-6.png)

这几天读了一下动视的技术白皮书，搞懂了一些东西，那么，便有此记录

纪念一下cod的这几年，和花出去的好几千块钱，over。

## 1. COD的段位和匹配机制

### 1.1 COD的段位等级

|   ​​段位名称​​    |         ​​玩家分布​​         |       ​​玩家分布​​        |
| :---------------: | :--------------------------: | :-----------------------: |
|   青铜 (Bronze)   |       新手/低活跃玩家        |      所有人起点段位       |
|   白银 (Silver)   |        入门级竞技玩家        |    允许与青铜/黄金组队    |
|    黄金 (Gold)    |        主流玩家集中段        | 可跨3段组队（青铜至铂金） |
|  铂金 (Platinum)  |        实力前30%玩家         |   组队跨度收缩（银-钻）   |
|  钻石 (Diamond)   |   顶尖玩家守门员（前10%）    | 仅能与黄金/铂金/钻石组队  |
|  深红 (Crimson)   |     职业级预备役（前3%）     |     仅限钻石/深红组队     |
| 虹彩 (Iridescent) | 全服Top 500（含Top 250榜单） |   ​​实际匹配按钻石+处理   |

**重要说明**

1. Top 250玩家在匹配系统中视为钻石+段位（非独立段位），因人数过少需合并匹配池
2. 段位转换严格依赖**​​SR数值**门槛​​（如黄金→铂金需2500 SR）

### 1.2 段位的隔离机制

#### ​组队硬约束​​策略

​1. ​跨段检测​​

系统实时计算队伍"段位差"（Rank Disparity）
例：钻石（差值为0） + 黄金（差值2） → 整队段位差=2
​​
2. 动态组队锁​​

钻石玩家邀请黄金队友 → 队伍​​立即禁止​​邀请白银玩家
虹彩玩家组队 → 全员必须为深红/虹彩

#### 匹配池分层策略

1. 优先同段位匹配

系统首先寻找同段位玩家（如铂金vs铂金）

2. 超时降级机制

```
搜索开始 --> 仅匹配同段位 --> 超时 --> 放宽至±1段 --> 超时--> 放宽至±2段 --> 超时 --> 启用跨区匹配+高延迟容忍
```

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%8E%92%E4%BD%8D%E8%B5%9B%E5%8C%B9%E9%85%8D%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/image-1.png)

1. 高级段位容易触发降级：铂金以上玩家常遇跨段匹配

#### TOP250处理

1. 单独匹配池

TOP250玩家自动进入候选池

2. 匹配更新机制

TOP250玩家匹配时视为**钻石+**段位，最低会匹配到钻石玩家

### 1.3 段位和Raw Skill（真实战力）的关系

在玩游戏的时候，我们经常会进行掉分操作，例如自雷或者自杀，或者不停的频繁去送

让系统对我们的技术评级降低，这样就能匹配到比较菜的玩家，大杀四方（我没这么干过就是了）

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%8E%92%E4%BD%8D%E8%B5%9B%E5%8C%B9%E9%85%8D%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/image.png)

根据白皮书的描述，可以看这张图，其实动视内部有个机制，是来判定你的真实战力（Raw Skill）的

首先先说结论

段位 **不等于** 游戏的真实实力

看这张图，

​​1. 高段位含金量高​​：深红(Crimson)以上玩家100%集中在技能前20%（0.8-0.9十分位）
​2. ​低段位鱼龙混杂​​：青铜(Bronze)中混有​​高技能玩家​​（0.7+技能分占比超5%）
​​3. 系统强制干预痕迹​​：黄金(Gold)段聚集大量​​被系统压制的中高技能玩家​​（0.5-0.7占比达13%）


我们继续分析这张图

在我玩游戏初期，经常能碰到一局杀几十个人的大神，甚至还有混战码头100多杀的大牛，这点我一开始就非常疑惑

因为本身我的KD只有0.9，或者1，这种大牛我能匹配到真的非常逆天

其实这张图就说明了很多东西

| 技能分 | 青铜段占比 |   ​​实际分布段位   |
| :----: | :--------: | :----------------: |
|  0.7   |     0      | 黄金(2%)/白银(5%)  |
|  0.6   |     0      | 黄金(3%)/白银(5%)  |
|  0.5   |     1%     | ​​青铜最高技能分​​ |

这里可以看出来，动视其实是**不允许技能分高于0.7**的玩家到青铜段炸鱼的

那么动视是如何让0.7+的战力玩家不能在青铜呆着呢？

白皮书上也说了

1. 加速晋级（战力校准机制）

如果Raw Skill ≥0.7的时候，那么就触发机制

```python
if current_rank == "Bronze" and raw_skill >= 0.7:
    SR_gain *= 2.5  # SR获取效率翻倍
    matchmaking_pool = "Gold/Platinum"  # 强制匹配更高段位
```

白皮书上描述：

​​1. 0.7技能玩家在青铜段 单场胜利SR可超100点（普通玩家20-30点）
1. 5连胜内必升入白银


2. 越段战斗（跨阶段战斗）

在某个阶段内，给你匹配高段位玩家，根据你的raw_skill数值来判定你匹配的真实段位玩家

有些时候会炸鱼，可能是你的匹配，把你匹到高段位玩家的池子了

有一种经典的情况：

**我上一把杀的嘎嘎爽，下一把马上成哈皮了**

不是动视针对你 → 是系统检测到Raw Skill≥0.7

绝壁你被划到高段位池子了，至于为什么这样做

**青铜段存在大量高玩 → 新手退游**

游戏属实就没法玩了，人全跑光了

所以一般会安排一大堆傻人机哈哈哈哈。


### 1.4 玩家成长和段位关系

从上面的分析可以看出来

1. 段位晋升≠线性成长

黄金→铂金：需突破"SR收益衰减墙"（胜场SR从50→20）
钻石→虹彩：强制"地狱检验"（仅匹配前10%玩家）

2. 系统强制干预场景

|       现象       |           触发条件            | ​​玩家感受 |
| :--------------: | :---------------------------: | :--------: |
|  连胜后突然连败  |      当前SR > 目标SR+15%      |  妈的搞我  |
| 青铜局遇大神狂杀 | 高潜力玩家检测（Raw Skill>0.7 |  妈的搞我  |
|  钻石段匹配超时  |      段位差规则降级失败       | 妈的鬼服了 |

## 2. COD的SR（Skill Rating）机制

SR我们已经在前面提过了

这逼玩意儿简单来说，就是控制你什么时候该升到什么段位，或者是你该获取多少经验值的一种动态干预算法

是不是有点子抽象，换个简单的语句来描述

这表面是段位系统，实则是动视的 **使奴驯化装置**。

### 2.1 动视内部的段位操作

首先，动视会给你进行一个评估

根据你每局的杀敌数/死亡数，算出一个真实的隐藏分数（Raw Skill），这是你无法看见的

这里的计算公式是

```python
raw_skill = (击杀×0.25 + 助攻×0.15 + 目标分×0.45 + 占点×0.15 - 死亡×0.3) / √(对局时长)
```

​​1. 目标分权重最高(0.45)​​（控制点位/炸弹模式核心指标）
​2. ​对局时长开平方​​（防止刷时长影响）


假设你现在的Raw Skill=0.6

然后根据你的真实水平（Raw Skill），动视会算出一个 ​​你**应该**在的段位​（Target SR）​（比如黄金需要2500分）

动视内部有个数值，我称之为：**行为调整因子**，这里是分析玩家行为数据的核心数据

这个数值的主要影响因素有：

| 影响因素          | 原描述 | 修正公式（编程级精准）               |
| ----------------- | ------ | ------------------------------------ |
| 高活跃度(日>4h)   | 负向   | -0.15 * log2(当日游戏小时数)         |
| 近期付费(7天消费) | 正向   | min(0.1, 消费金额($)/2000)           |
| 弃游风险          | 负向   | -0.3 * (1 - min(1, 当日活跃/周平均)) |

这里的Target SR计算公式是

```python
base_SR = min(4500, Raw_Skill × 4500)
behavior_effect = behavior_factor × 0.25 × base_SR/1000
Target_SR = base_SR + behavior_effect
```


写一个伪代码

```python
# 计算Raw Skill（每局更新）
def calc_raw_skill(match_data):
    kills = match_data.kills
    assists = match_data.assists
    objectives = match_data.objective_points  # 炸弹/点位控制分
    zone_time = match_data.zone_seconds       # 占点秒数
    deaths = match_data.deaths
    duration = match_data.duration_seconds / 60  # 转为分钟
    
    return (kills*0.25 + assists*0.15 + objectives*0.45 + zone_time*0.15 - deaths*0.3) / math.sqrt(duration)

# 计算目标SR（每日更新）
def calc_target_SR(raw_skill, behavior_data):
    base_SR = min(4500, raw_skill * 4500)  # 基础映射，上限4500
    
    # 计算行为影响值（每1000分±25%）
    bh_factor = calc_behavior_factor(behavior_data)
    behavior_effect = bh_factor * 0.25 * (base_SR / 1000)
    
    return min(5500, max(0, base_SR + behavior_effect))  # 全段位边界保护
```

### 2.2 动视关于输赢的操作

SR系统本质是 ​​动态难度调节器​​——它决定你何时能赢，何时必须输。

#### 连续胜

```python
def on_win_streak(streak_count):
    if streak_count >= 3:  # 触发三连胜协议
        # 1. 匹配系统启动"段位跃升测试"
        matchmaking.set_opponent_rank(current_rank + 2)  # 匹配高2段车队
        
        # 2. SR收益压制系统激活
        win_reward = max(10, base_SR * 0.4)  # 基础分打4折
        
        # 3. 表现分封顶机制
        performance_bonus = min(5, raw_performance)  # 杀50人只算5分
        
        return win_reward + performance_bonus
```

1. 当你三连胜时，下局必遇 ​​钻石车队打黄金局​​（白皮书第5.2.3节称其为“confidence-based calibration”）
2. 就算你力挽狂澜赢了
```python
原可获25基础分+15表现分=40分  
现强制压制为(25 * 0.4)+5=15分（砍掉62.5%）  
```

所以你赢了好几把之后，你是一定会输的，你打不到那前100，你输的概率都是非常大的。


#### 连续输

```python
def on_lose_streak(streak_count, player_type):
    if streak_count >= 3:  # 三连败触发救济
        if player_type == "NEWBIE":  # 新手保护
            spawn_bot_lobby()  # 生成人机局
            return 30  # 固定+30分（希望药剂）
        else:  # 老油条处置
            reduce_next_opponent_skill(40)  # 下局对手强度-40%
            return -10  # 扣分减半（从-18→-10）
    elif player_type == "WHALE":  # 付费玩家特殊关怀
        return abs(base_SR) * 0.3  # 扣分再打7折
```

白皮书上也说了：

Loss streak interventions boost retention by 22%


| 玩家类型 | 三连败救济方案          |
| -------- | ----------------------- |
| 萌新     | 人机局​​ → +30分        |
| 老玩家   | 对手强度-40% & 扣分减半 |
| 充值玩家 | 扣分再打7折             |

### 2.3 赛季控制的操作

#### 赛季重置

​1. ​软性段位重置 (Soft Rank Reset)​
将所有玩家的可视段位重置至初始等级（如青铜），但保留隐藏实力分（Raw Skill）的90%

2.​​差异化匹配处理 (Tiered Matchmaking Treatment)​
根据玩家历史实力划分匹配池，高竞争力玩家仍与原段位玩家对战

​​3.加速晋升协议 (Accelerated Promotion Protocol)​
高竞争力玩家在重置后获得额外SR增益系数

动视白皮书描述（Sec 7.2）：​​
```
"Rank resets apply cosmetic demotions while preserving core skill metrics, 
enabling high-skill players to rapidly return to appropriate tiers."
（段位重置仅应用表面降级，同时保留核心技能指标，使高水平玩家能快速返回适当段位）
```

#### 控制晋级

1.​​动态难度校准 (Dynamic Difficulty Calibration)​
根据玩家实时状态调整晋升赛对手强度

2.受控晋升率 (Controlled Promotion Rate)​
系统精确控制不同段位间的晋升成功率（通常53-58%）


3.检验性匹配 (Validation Matching)
晋升关键局分配高强度对手验证玩家实力


```
动视白皮书描述（Sec 5.3）：​​

"Promotion matches trigger validation protocols, 

exposing players to marginally superior opponents to confirm tier readiness."
（晋升赛触发检验协议，使玩家面对略强对手以确认段位准备度）
```

所以你会直接感受到如下的游玩体验

| 玩家视角     | 技术术语       | 真实目的               |
| ------------ | -------------- | ---------------------- |
| 新赛季新开始 | 软性段位重置   | 维护高玩炸鱼特权       |
| 晋级赛遇大神 | 检验性匹配     | 压制自然晋升率         |
| 刚升段就连败 | 段位适应性调节 | 迫使玩家停留当前段位   |
| 赛季初被血虐 | 差异化匹配处理 | 用低段玩家供养高玩留存 |

## 3. 三个重要匹配机制

### 3.1 网络优先机制

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%8E%92%E4%BD%8D%E8%B5%9B%E5%8C%B9%E9%85%8D%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/image-2.png)

1. 排位赛强制 ​​Delta Ping≤50ms​​（红线），而普通模式无限制
2. 当匹配超时后会强制放宽至15ms

等于说玩游戏的人受两种折磨

第一个就是，你会在低分段，受到来自匹配超时大佬的暴击（可能

第二个就是，在高段位，你等待排队的时间只会更长/延迟更大

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%8E%92%E4%BD%8D%E8%B5%9B%E5%8C%B9%E9%85%8D%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/11a266797b5c3.png)

### 3.2 段位/SR双过滤机制

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%8E%92%E4%BD%8D%E8%B5%9B%E5%8C%B9%E9%85%8D%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/image-3.png)

|        坐标轴         |           技术含义           |     白皮书关联机制      |
| :-------------------: | :--------------------------: | :---------------------: |
| ​​横轴 Lobby Max SR​​ | 当局所有玩家中的​​最高SR值​​ | 决定匹配强度（3.2.1节） |
|    ​​纵轴 Count​​     | 遭遇该SR值的​​对局次数占比​​ |  组队惩罚算法（5.3节）  |


1. 如果你是单排玩家，并且你的SR非常高（2000-3000R），你的匹配速度会更快，因为系统会加速校准
2. 如果你是组队玩家，你的SR出现异常高峰（队内有钻石大佬），就会触发跨段惩罚机制

```python
# 单排青铜
if solo_player and bronze_rank:
    lobby_max_sr = target_sr * 1.2  # 系统校准性匹配
    # 结果：集中于2000-3000（黄金段）

# 组队
if partied_with_diamond:
    lobby_max_sr = max(diamond_sr)*1.5  # 惩罚系数
    # 结果：5000-8000异常峰（虹彩段）
    # 当青铜与钻石组队 → 按钻石SR×1.5匹配 → 青铜面对7500SR虹彩战神
```

### 3.3 Raw Skill干预机制

![alt text](https://blog-1256169066.cos.ap-chengdu.myqcloud.com/%E6%8E%92%E4%BD%8D%E8%B5%9B%E5%8C%B9%E9%85%8D%E6%9C%BA%E5%88%B6%E8%AF%A6%E8%A7%A3/image-4.png)

|        坐标轴         |             技术含义             |
| :-------------------: | :------------------------------: |
| ​​横轴 Lobby Max SR​​ | 当局最高SR值 → 代表对局强度等级  |
| ​​纵轴 Probability​​  | 玩家在该强度对局中的​​存活概率​​ |

```python
# 底层玩家封锁（白皮书4.3节）
if player.percentile < 0.25:
    target_sr = min(1500, raw_skill*5000)  # 锁白银
    allow_high_tier = False  # 禁止接触高端局

# 顶尖玩家特权
if player.percentile > 0.95:
    bypass_rank_constraints = True  # 可突破段位隔离
```

这个的核心意思就是，后25%玩家永远在青铜困住，真玩家胜率只会越来越低


## 4. 结尾

动视的白皮书，一共出了三个部分

1. 排位赛匹配机制详解
2. Ping机制详解
3. SBMM（Skill-Based Matchmaking，基于技术水平的匹配机制）

我这段时间，会依次把这些文章读完并且发出来

脱坑cod之后，刹那天地宽

再也不玩了，再也不买了，靠

