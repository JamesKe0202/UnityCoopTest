# IRIS 调候系统 & 世界状态 v0.1 技术设计说明

## 0. 目标概述

本项目的核心是一个 **“情感气候系统 + 城市响应”**：

- 整个城市只有一个 **全局 IRIS（5 维情感状态）**，所有 NPC 的情绪表现共享这组状态。
- 全局 IRIS 不直接显示给玩家，而是通过：
  - 天气 / 光线 / 云量 / 可见度 / 温度 / 日月相位 / 植被 / 地表地貌  
  - 城市电子屏与发光物（广告、公告、情感商品价格）  
  - 城市交通运行节奏（公交频率、车流顺畅度）  
  - 背景音乐曲目（BGM 模式）  
  - 城市空间的开启/关闭（有些空间只有特定 IRIS 状态下才存在）  
  - 城市/社区活动与节日  
  - NPC 出现密度与移动速度  
  来“说话”。
- 影响 IRIS 的主要来源：
  - 主角的 **特殊 IRIS 面板**（可以主动微调世界情绪，有代价）  
  - 玩家在游戏世界中的关键交互（对话选择、行为）  
  - 任务完成情况（主线/支线关键节点）  
  - 玩家能力 / 技能对世界的主动调控。

当前阶段（Level 0 multi-agent）：

- **所有 NPC 在逻辑上被视为一个“情感群体”**：  
  他们共享同一个 IRIS，但在空间中以多个实体出现（密度、速度受 IRIS 指导）。
- “多智能体”只在 **世界模型层面** 表现为“城市整体情感状态”，暂不逐一追踪单个 NPC 的个人 IRIS。

---

## 1. 核心数据结构概览

### 1.1 全局 IRIS 状态（5 维）

五个维度定义如下：

1. `intimacyDrive`（亲密关系渴望度）  
   - 高：强烈渴望亲密、愿意靠近与倾诉  
   - 低：追求情感极简、避免牵扯  

2. `romance`（浪漫度）  
   - 高：仪式感、审美、节日与“戏剧性”表达多  
   - 低：关系更工具化、功能化、少装饰  

3. `stimulation`（刺激度）  
   - 高：追求新鲜感、强刺激事件、夜生活  
   - 低：节奏慢、低感官负荷  

4. `trust`（信任度）  
   - 高：信任他人 / 机构 / 系统，整体安全感强  
   - 低：怀疑、防御、偏向合约 / 对冲机制  

5. `fairness`（公平度）  
   - 高：资源与情感交换被感知为大致公平  
   - 低：强烈被剥削感、不公、补偿性 / 报复性行为多  

#### C# 结构示例

    [System.Serializable]
    public class IrisState
    {
        // 建议范围为 [0, 1]
        public float intimacyDrive;  // 亲密关系渴望度
        public float romance;        // 浪漫度
        public float stimulation;    // 刺激度
        public float trust;          // 信任度
        public float fairness;       // 公平度

        public IrisState(float intimacyDrive = 0.5f, float romance = 0.5f,
                         float stimulation = 0.5f, float trust = 0.5f, float fairness = 0.5f)
        {
            this.intimacyDrive = intimacyDrive;
            this.romance       = romance;
            this.stimulation   = stimulation;
            this.trust         = trust;
            this.fairness      = fairness;
            Clamp01();
        }

        public void Clamp01()
        {
            intimacyDrive = Mathf.Clamp01(intimacyDrive);
            romance       = Mathf.Clamp01(romance);
            stimulation   = Mathf.Clamp01(stimulation);
            trust         = Mathf.Clamp01(trust);
            fairness      = Mathf.Clamp01(fairness);
        }

        // ===== 派生情绪-气候指数（供世界规则调用） =====

        // 情绪温度：整体“暖不暖”
        public float GetEmotionalTemperature()
        {
            // 亲密渴望 + 浪漫 + 一点刺激
            return Mathf.Clamp01(0.4f * intimacyDrive + 0.4f * romance + 0.2f * stimulation);
        }

        // 混沌度：整体“乱不乱”
        public float GetChaosLevel()
        {
            // 刺激高 + 信任低 + 公平低 → 混沌高
            return Mathf.Clamp01(
                0.5f * stimulation +
                0.25f * (1f - trust) +
                0.25f * (1f - fairness)
            );
        }

        // 安全感：稳定与可预期程度
        public float GetSecurityLevel()
        {
            // 信任 + 公平 + 低刺激
            return Mathf.Clamp01(
                0.45f * trust +
                0.35f * fairness +
                0.2f  * (1f - stimulation)
            );
        }

        // 浪漫张力：浪漫 *（情绪温度 + 混沌）
        public float GetRomanticIntensity()
        {
            float temp  = GetEmotionalTemperature();
            float chaos = GetChaosLevel();
            return Mathf.Clamp01(romance * (0.5f * temp + 0.5f * chaos));
        }
    }

---

### 1.2 调候原型（8 个 ClimateArchetype）

调候不是只有 8 种，而是一个 **连续空间**。  
先定义 8 个“地理-文化原型”，作为气候 preset 的 **基准点**：

    public enum ClimateArchetype
    {
        HawaiiSummer,       // 夏威夷夏天
        NorwayWinter,       // 挪威冬天
        ArcticPolarNight,   // 北极极夜
        SaharaDesert,       // 撒哈拉荒漠
        ChinaRedAutumn,     // 中国红叶秋
        MonsoonMetropolis,  // 季风大都会（雨季港口/东亚城市）
        NeonTyphoonNight,   // 霓虹台风夜（赛博+暴雨）
        HighlandClearSky    // 高原澄空（日照强、空气极清）
    }

实际世界状态不会“只在 8 个档位上跳”，而是：

1. 用 IRIS 计算情绪指标（温度 / 混沌 / 安全 / 浪漫张力）。
2. 基于这些指标选出一个 **主导原型（dominant archetype）**。
3. 在原型 preset 上做 **连续微调与随机扰动**，形成丰富的“同一原型内部变化”。

---

### 1.3 植被 & 地表类型

    public enum VegetationType
    {
        Tropical,       // 热带阔叶 / 棕榈
        Coniferous,     // 针叶林
        Tundra,         // 苔原 / 稀疏低矮植被
        DesertSparse,   // 荒漠稀疏灌木
        DeciduousRed,   // 落叶红叶林
        UrbanGreen,     // 城市绿植（行道树、绿化带）
        MossyWet,       // 雨季苔藓、潮湿植物
        HighlandGrass   // 高原草甸
    }

    public enum SurfaceType
    {
        Sand,
        Rock,
        Snow,
        Ice,
        WetConcrete,
        DryConcrete,
        LeafLitter,
        Grass,
        Mixed
    }

---

### 1.4 环境状态 EnvironmentState

    [System.Serializable]
    public class EnvironmentState
    {
        // 调候原型 + 内部变化参数
        public ClimateArchetype climateArchetype;
        public float archetypeBlend;   // 0-1，在原型 preset 内偏移/混合程度（简单版本可固定为 0.5）

        // 植被与地表
        public VegetationType vegetation;
        public SurfaceType surface;

        // 天空 & 天气
        public float cloudiness;      // 云量 0-1
        public float lightIntensity;  // 光照强度 0-1
        public float visibility;      // 可见度 0-1
        public float temperature;     // 体感温度 0-1（0=极冷，1=极热）
        public float sunHeight;       // 0-1，太阳高度 / 存在感
        public float moonPhase;       // 0-1，新月→满月

        // 城市屏幕 / 发光物
        public float adIntensity;       // 广告屏亮度 / 数量
        public float adCommercialRatio; // 产品广告 vs 公益/公告 的比例 0-1
        public float publicAlertLevel;  // 公共警报/系统提示频率 0-1

        // 交通
        public float busFrequency;      // 公交班次频率 0-1（相对值）
        public float trafficFlow;       // 车流密度 0-1
        public float flowOrderliness;   // 车流有序程度 0-1

        // NPC 场景分布
        public float npcDensity;        // 同屏 NPC 密度 0-1
        public float npcSpeed;          // 移动速度 0-1
        public float npcPathNoise;      // 行走路径随机性 0-1
    }

---

### 1.5 声音：简化 BGM 模式

    public enum BgmMode
    {
        CalmWarm,      // 平静温暖
        CalmCold,      // 平静冷感
        Tense,         // 紧张压迫
        Melancholy,    // 忧郁/回忆
        HyperActive,   // 过度兴奋 / 夜场
        Empty          // 近乎无音 / 极度稀薄
    }

    [System.Serializable]
    public class SoundState
    {
        public BgmMode currentBgmMode;
    }

---

### 1.6 世界整体状态 WorldState

    [System.Serializable]
    public class WorldState
    {
        public IrisState iris;             // 全局 IRIS
        public EnvironmentState env;       // 环境 & 城市状态
        public SoundState sound;           // BGM 模式

        // 空间与活动（示意）
        public List<string> openZones;     // 当前开放的特殊空间 ID
        public List<string> activeEvents;  // 正在进行的城市/社区活动 ID
    }

---

## 2. 调候系统：IRIS → 气候原型 → 具体环境参数

### 2.1 步骤总览

1. 从 `IrisState` 计算情绪-气候指标：  
   - `emotionalTemperature`（情绪温度）  
   - `chaosLevel`（混沌度）  
   - `securityLevel`（安全感）  
   - `romanticIntensity`（浪漫张力）

2. 基于这些指标选出一个 **主导 ClimateArchetype**。
3. 从该 archetype 读取一组 **基准 preset**（植被 / 地表 / 云量 / 光照等）。
4. 在 preset 上，根据 IRIS 指标做 **连续微调**，再加少量随机扰动/时间因子，得到最终 `EnvironmentState` 参数。

---

### 2.2 选择 ClimateArchetype（简单版本）

    public static class ClimateSelector
    {
        public static ClimateArchetype SelectArchetype(IrisState iris)
        {
            float temp     = iris.GetEmotionalTemperature();
            float chaos    = iris.GetChaosLevel();
            float secure   = iris.GetSecurityLevel();
            float romantic = iris.GetRomanticIntensity();

            // 以下是粗略规则，可后期用 ScriptableObject 调参

            if (temp > 0.7f && romantic > 0.6f && chaos < 0.5f)
                return ClimateArchetype.HawaiiSummer;

            if (temp < 0.3f && secure > 0.7f && romantic < 0.5f)
                return ClimateArchetype.NorwayWinter;

            if (temp < 0.25f && romantic < 0.3f && chaos > 0.6f)
                return ClimateArchetype.ArcticPolarNight;

            if (temp > 0.7f && secure < 0.4f && iris.fairness < 0.4f)
                return ClimateArchetype.SaharaDesert;

            if (romantic > 0.65f && temp > 0.4f && temp < 0.7f)
                return ClimateArchetype.ChinaRedAutumn;

            if (temp > 0.5f && chaos > 0.5f && iris.stimulation > 0.5f)
                return ClimateArchetype.MonsoonMetropolis;

            if (chaos > 0.75f && iris.stimulation > 0.7f)
                return ClimateArchetype.NeonTyphoonNight;

            if (secure > 0.7f && temp > 0.4f && temp < 0.7f)
                return ClimateArchetype.HighlandClearSky;

            // 默认兜底
            return ClimateArchetype.MonsoonMetropolis;
        }
    }

---

### 2.3 原型 preset：按 archetype 填充 EnvironmentState 基准值

    public static class EnvironmentPresets
    {
        public static void ApplyClimatePreset(EnvironmentState env)
        {
            switch (env.climateArchetype)
            {
                case ClimateArchetype.HawaiiSummer:
                    env.vegetation     = VegetationType.Tropical;
                    env.surface        = SurfaceType.Sand;
                    env.cloudiness     = 0.3f;
                    env.lightIntensity = 0.9f;
                    env.visibility     = 0.9f;
                    env.temperature    = 0.85f;
                    break;

                case ClimateArchetype.NorwayWinter:
                    env.vegetation     = VegetationType.Coniferous;
                    env.surface        = SurfaceType.Snow;
                    env.cloudiness     = 0.6f;
                    env.lightIntensity = 0.3f;
                    env.visibility     = 0.85f;
                    env.temperature    = 0.2f;
                    break;

                case ClimateArchetype.ArcticPolarNight:
                    env.vegetation     = VegetationType.Tundra;
                    env.surface        = SurfaceType.Ice;
                    env.cloudiness     = 0.7f;
                    env.lightIntensity = 0.1f;
                    env.visibility     = 0.7f;
                    env.temperature    = 0.1f;
                    break;

                case ClimateArchetype.SaharaDesert:
                    env.vegetation     = VegetationType.DesertSparse;
                    env.surface        = SurfaceType.Sand;
                    env.cloudiness     = 0.1f;
                    env.lightIntensity = 1.0f;
                    env.visibility     = 0.8f;
                    env.temperature    = 0.95f;
                    break;

                // TODO: 补全其他 archetype
            }
        }
    }

---

### 2.4 在 preset 基础上做“内部变化”

    public static class EnvironmentAdjuster
    {
        public static void AdjustEnvironment(EnvironmentState env, IrisState iris, float timeOfDay01)
        {
            float temp     = iris.GetEmotionalTemperature();
            float chaos    = iris.GetChaosLevel();
            float secure   = iris.GetSecurityLevel();
            float romantic = iris.GetRomanticIntensity();

            // 云量：浪漫高 → 云更多光柱，混沌高 → 云层运动更剧烈
            env.cloudiness += (1f - romantic) * 0.2f + chaos * 0.1f;

            // 光照：情绪温度越高 → 光强更高
            env.lightIntensity *= 0.8f + temp * 0.3f;

            // 可见度：安全感高 → 空气更清晰
            env.visibility *= 0.6f + secure * 0.4f;

            // 温度微调
            env.temperature = (env.temperature * 0.5f) + (temp * 0.5f);

            // 日月相位：简单用 timeOfDay01 映射（可后续替换为更复杂逻辑）
            env.sunHeight = Mathf.Clamp01(timeOfDay01);
            env.moonPhase = 1f - env.sunHeight;

            // 交通：刺激 & 混沌 → 车流快且乱
            env.trafficFlow     = Mathf.Clamp01(0.5f + iris.stimulation * 0.4f);
            env.flowOrderliness = Mathf.Clamp01(0.7f * secure - 0.3f * chaos);

            // 屏幕广告：浪漫 & 刺激高 → 情感产品广告多；信任低 & 公平低 → 系统警报多
            env.adIntensity       = Mathf.Clamp01(0.5f + romantic * 0.3f + iris.stimulation * 0.2f);
            env.adCommercialRatio = Mathf.Clamp01(0.5f + romantic * 0.3f - (1f - iris.fairness) * 0.2f);
            env.publicAlertLevel  = Mathf.Clamp01(0.3f + chaos * 0.4f + (1f - iris.trust) * 0.3f);

            // NPC 场景分布
            env.npcDensity   = Mathf.Clamp01(0.4f + iris.stimulation * 0.4f);
            env.npcSpeed     = Mathf.Clamp01(0.3f + iris.stimulation * 0.5f);
            env.npcPathNoise = Mathf.Clamp01(0.2f + chaos * 0.6f);
        }
    }

---

## 3. BGM 切换规则

    public static class BgmSelector
    {
        public static BgmMode SelectBgm(IrisState iris, ClimateArchetype climate)
        {
            float temp     = iris.GetEmotionalTemperature();
            float chaos    = iris.GetChaosLevel();
            float secure   = iris.GetSecurityLevel();
            float romantic = iris.GetRomanticIntensity();

            if (chaos > 0.7f && iris.stimulation > 0.6f)
                return BgmMode.HyperActive;

            if (secure > 0.7f && temp > 0.6f)
                return BgmMode.CalmWarm;

            if (secure > 0.7f && temp < 0.4f)
                return BgmMode.CalmCold;

            if (temp < 0.3f && chaos < 0.4f)
                return BgmMode.Empty;

            if (romantic > 0.6f)
                return BgmMode.Melancholy;

            return BgmMode.Tense;
        }
    }

引擎侧实现：

- 预置 6 条 loop BGM，分别对应 6 种模式。
- 每当 `BgmMode` 变化时，淡出当前、淡入新模式即可。

---

## 4. 特殊空间与解锁条件（示例）

特殊空间由 **调候 + IRIS + 任务状态** 决定是否出现/开放。  
技术上：`WorldState.openZones` 中包含其 ID，即视为“存在并可进入”。

示例（从中选若干实现）：

| 空间 ID             | 解锁条件（简要）                                                                                  | 功能示意                                           |
|---------------------|---------------------------------------------------------------------------------------------------|----------------------------------------------------|
| HiddenGarden        | archetype = HawaiiSummer / ChinaRedAutumn；intimacyDrive & romance 较高                          | 热带/红叶隐蔽花园，缓慢恢复玩家某些状态             |
| LightfogGallery     | romance 高；NorwayWinter / ChinaRedAutumn                                                        | 悬空光雾回廊，播放关系影像                          |
| EmotionExchangeHall | stimulation 高、fairness 低；NeonTyphoonNight                                                   | 情绪证券交易所，展示“情感指数”与价格               |
| DopamineGreenhouse  | intimacyDrive 高、stimulation 高；HawaiiSummer / MonsoonMetropolis                              | 多巴胺温室，短期 buff，长期有副作用                 |
| PolarObservatory    | trust 高、romance 低；ArcticPolarNight / HighlandClearSky                                       | 极夜观测站，可看全局 IRIS 趋势                      |
| DuneTheatre         | fairness 极低；SaharaDesert                                                                      | 沙丘剧场，反思“集体失败”的空间                      |
| EchoOfAutumn        | romance 高；ChinaRedAutumn                                                                       | 红叶回响站，影响长期关系弧线                        |
| EmotionCryoArchive  | trust 高、stimulation 极低；NorwayWinter / ArcticPolarNight                                      | 情绪冷冻库，可暂时锁定某部分 IRIS 波动              |
| NightMarketBlur     | stimulation 高、trust 低；NeonTyphoonNight / MonsoonMetropolis                                   | 失焦夜市，贩卖高风险情感改造服务                    |
| RootLibrary         | intimacyDrive 低、fairness 高；任意调候（极夜/荒漠期更易出现）                                   | 地下根系图书馆，解锁世界观/模型参数                 |

示意实现：

    public static class ZoneUnlocker
    {
        public static void UpdateOpenZones(WorldState world, QuestState quest)
        {
            world.openZones.Clear();
            var iris    = world.iris;
            var climate = world.env.climateArchetype;

            // 示例：HiddenGarden
            if ((climate == ClimateArchetype.HawaiiSummer || climate == ClimateArchetype.ChinaRedAutumn) &&
                iris.intimacyDrive > 0.6f && iris.romance > 0.6f)
            {
                world.openZones.Add("HiddenGarden");
            }

            // TODO: 其他空间同理添加...
        }
    }

---

## 5. IRIS 的更新来源（接口）

当前实现只需要预留几个更新接口，具体数值由设计侧提供“事件 → ΔIRIS 表”。

### 5.1 主角 IRIS 面板调整

玩家通过一个 UI 面板设置“目标 IRIS 状态”（`targetIris`），  
世界 IRIS 慢慢向 `targetIris` 收敛，并消耗某种资源。

    public static class IrisPanelUpdater
    {
        public static void ApplyPanelTarget(WorldState world, IrisState target, float influenceStrength)
        {
            // influenceStrength 建议 0.0~0.2 之间
            world.iris.intimacyDrive = Mathf.Lerp(world.iris.intimacyDrive, target.intimacyDrive, influenceStrength);
            world.iris.romance       = Mathf.Lerp(world.iris.romance,       target.romance,       influenceStrength);
            world.iris.stimulation   = Mathf.Lerp(world.iris.stimulation,   target.stimulation,   influenceStrength);
            world.iris.trust         = Mathf.Lerp(world.iris.trust,         target.trust,         influenceStrength);
            world.iris.fairness      = Mathf.Lerp(world.iris.fairness,      target.fairness,      influenceStrength);

            world.iris.Clamp01();
        }
    }

### 5.2 游戏内交互事件

每个关键交互事件有一个 `IrisDelta`（五维加减）；  
触发时调用 `ApplyEventDelta()` 即可。

    [System.Serializable]
    public class IrisDelta
    {
        public float dIntimacyDrive;
        public float dRomance;
        public float dStimulation;
        public float dTrust;
        public float dFairness;
    }

    public static class IrisEventApplier
    {
        public static void ApplyEventDelta(WorldState world, IrisDelta delta)
        {
            world.iris.intimacyDrive += delta.dIntimacyDrive;
            world.iris.romance       += delta.dRomance;
            world.iris.stimulation   += delta.dStimulation;
            world.iris.trust         += delta.dTrust;
            world.iris.fairness      += delta.dFairness;
            world.iris.Clamp01();
        }
    }

### 5.3 任务 / 能力等

- 任务完成：调用 `ApplyEventDelta`，并视设计需求修改某些“平衡点”。
- 能力/技能：可视为特殊 Event，同样通过 `IrisEventApplier` 更新 IRIS。

---

## 6. 更新流程建议（引擎中的一帧 / 一周期）

每当 IRIS 或玩家行为有更新，世界应按以下顺序更新：

1. 根据玩家面板 / 交互 / 任务 → 更新 `world.iris`。  
2. 使用 `ClimateSelector.SelectArchetype(iris)` → 选出 `climateArchetype`。  
3. 使用 `EnvironmentPresets.ApplyClimatePreset(env)` → 填入基准环境。  
4. 使用 `EnvironmentAdjuster.AdjustEnvironment(env, iris, timeOfDay)` → 做内部微调。  
5. 根据 `iris`, `climate` → 选择 `BgmMode`（`BgmSelector.SelectBgm`）。  
6. 根据 `iris`, `env` → 更新 `openZones` / `activeEvents`（`ZoneUnlocker.UpdateOpenZones`）。  
7. 场景层从 `WorldState` 读参数：
   - 天空系统 / 天气系统  
   - 植被/地表材质（通过 prefab / shader 参数）  
   - 屏幕广告样式与亮度  
   - 交通系统与 NPC spawner  
   - BGM 切换  
   - 特殊空间入口的显隐 / 可交互状态  

---

## 7. MVP 实现优先级（建议）

为便于两人小组在短时间内落地 demo，建议优先实现：

### 7.1 必做

1. `IrisState`（5 维）+ 派生指数函数。  
2. `WorldState` + `EnvironmentState` + `SoundState` 数据结构。  
3. `ClimateSelector` + `EnvironmentPresets` + `EnvironmentAdjuster`（可先简单版本）。  
4. `BgmSelector` + 6 条 BGM 模式切换。  
5. 至少 1–2 个特殊空间的解锁逻辑（例如 `HiddenGarden` + `EmotionExchangeHall`）。  
6. Scene 中可视化以下内容：
   - 天空 / 光线（简单 skybox + Directional Light 即可）。  
   - 至少一种植被 / 地表变化（通过切换 prefab / 材质）。  
   - 若干广告屏/灯箱根据 `env.adIntensity` & `env.adCommercialRatio` 变化。  
   - NPC 密度 / 速度变化（用简单 AI 移动）。

### 7.2 可选 / 后续

1. 更多调候原型的精细差异。  
2. 更多特殊空间与对应 UI / 交互。  
3. 无人机灯阵、情绪云图墙、动态斑马线等“情绪城市模块”。  
4. 将所有规则迁移到 ScriptableObject，便于非程序同学调参。  
5. 日志系统（记录 IRIS / ENV / 玩家行为，为 future learned world model 做准备）。
