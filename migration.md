# Claw ↔ MotionLab 数据迁移说明（combat.html 对接页配套）

状态：迁移的是**时间、轨迹、事件**，不是把 Claw 的画法搬进 MotionLab，
也不是把 MotionLab 的火柴人坐标直接当成 Claw 成片坐标。
正式渲染仍由 Claw 仓库的 clawd() 负责；本页的方块 Claw 只是预演示意。

## 一、Claw 那边的事实（读自 claw-animation-base）

- Claw 是方块体：身体 10u 宽 × 6u 高（y −8u…−2u），腿到地面 y 0，
  全高 8u；地面点 (x, y) 在两脚之间；正视图 4 条短腿在 x −4、−2、1、3u，
  手臂 pivot 在 (±4.9u, −4.5u)，臂长仅 2.2u。
- 视图是关键视图（front / q / side / qback / back / highFront / highQ），
  不是自由旋转的 3D 盒子；转身是切关键视图，不是转模型。
- 表演字段（acting.js ACT_BASE）：dx、dy、sq、rot、aL、aR、eyes、mouth、
  lookX、lookY、squint、blush、gloom；配方是“时刻＋姿态增量”的关键帧表，
  act() 在配方之间用 0.16 × pace 做混合，不跳变。
- clawd() 选项注释：dy 负值 = 上；sq 挤压（负值拉伸）；
  rot 绕脚旋转；aL / aR：0 = 平举，+ = 上，− = 下。
- clawdAnchor(x, y, u, o, name) 是现成的锚点接口：
  handR / handL / feet / foot0–3 / eyes，返回经过 dx / dy / sq / rot
  变换后的画面坐标——这是 Claw 侧校验“手到没到”的官方落点，
  不是 MotionLab 的手坐标。

## 二、映射表（精确度分三档：精确 / 换算 / 近似）

| Claw / 词典 | MotionLab 战斗页 | 精确度与说明 |
|---|---|---|
| 配方关键帧时刻（秒） | 相位起止（秒 = 帧 ÷ 24） | 精确：两边都以秒为准绳 |
| 候选 hit_frame（@24） | 相位表算出的命中帧 | 精确：本页命中帧由相位算出，不手写 |
| 事件 impact / hitstop-start / recover-start | 导出 JSON 的 events（按事件名挂接） | 精确：不按帧号硬编码 |
| dx（u） | 主角地面 x 位移（px）= dx × u | 换算：本页方块 Claw 取 u = 11.5px |
| dy（u，负 = 上） | yOff（px，正 = 下）= −dy × u | 换算：符号翻转，别直接抄 |
| sq | 体宽 ×(1 + sq×0.6)、体高 ×(1 − sq) | 公式精确（clawd 的 scale）；本页用 yOff 推导 sq 是近似 |
| rot（弧度，绕脚） | lean（度）= rot × 57.3 | 近似：MotionLab 以骨盆为基准，Claw 以脚为基准，角度可比、支点不同 |
| aR / aL（0 = 平举） | 手方向：a（弧度）= (sh − 90) × π/180，sh 为本页肩角（0 = 下垂） | 方向一致，但 Claw 臂仅 2.2u，手位置以 clawdAnchor 为准，不搬 MotionLab 手坐标 |
| 15 个节点名（pelvis、chest、head、shoulderR/L、elbowR/L、handR/L、hipR/L、kneeR/L、footR/L） | 同名节点 | 名称精确，坐标系不同：MotionLab 是 px，Claw 是 u，需除以 u 并以地面点为原点；v1 导出已注明“Claw 侧按节点名映射” |
| eyes / mouth / lookX / lookY / squint / blush / gloom | 未迁移 | 本页方块脸固定为中性（两道眼缝＋一字嘴），表情仍由 Claw 侧接管 |

## 三、迁移方向

### MotionLab → Claw（预演定稿后）

1. 取本页导出 JSON：phases（起止秒）→ act() 的关键帧时刻；
   每个相位起点写一个 key，姿态增量由 dx / dy / sq 推导。
2. 手方向按上表公式换算成 aL / aR，**到 Claw 侧实测**后再定稿，
   不把换算值当成已验证的配方。
3. 到 Claw 侧渲染后，用 clawdAnchor(handR) 校验手落点：
   比的是方向＋时刻，不比像素相等（两边骨架不同）。

### Claw → MotionLab（已有配方想上预演台）

1. 取 acting.js 配方的关键帧时刻表，原样作为相位表（秒）。
2. dx / dy 按上表换算成 px 位移；sq 按公式换算成体宽体高；
   rot 换算成 lean 度数；aL / aR 反推手方向角。
3. eyes / mouth / 表情不带过来——预演台只看动作逻辑，
   不看表情演技。

## 四、已验证的例子：重拳（heavy-punch）

- 相位（@24）：前摇 5 帧（0.208s）→ 出拳 2 帧 → 命中帧第 7 帧 →
  停顿 6 帧（0.250s）→ 后摇 8 帧 → 回稳。
- 换算到舞台 @60：停顿 0.250s = 15 帧 @60，不是把 6 直接当舞台帧数。
- 手方向：出拳时本页肩角 84° → a = (84 − 90)° = −6° ≈ −0.105 弧度
  （接近平举略向下）；前摇肩角 −118° 的换算值超出常规范围，
  到 Claw 侧实测后再定，不直接写入配方。

## 五、顺带复查：生物力学 / 机械 / 天体三域的现状

结论：三域在 motionlab.html 里是**实装，不是占位**，各有画布、参数、
校验与独立导出，且明确不碰 Claw、不混进 motionlab-0.1：

- 生物力学（domain-bio-0.1）：步态，两段骨骼 IK（余弦定理求膝），
  参数步频 / 步幅 / 下蹲；校验关节屈曲、重心支撑、足部离地间隙。
  局限：是通用双足步态，不是 Claw 的 4 条短腿；不能直接当 Claw 走路。
- 机械（domain-mech-0.1）：曲柄滑块＋齿轮，活塞 x 由曲柄几何算出，
  从动轮转速 = −曲柄 / 齿轮比；参数曲柄半径 / 连杆长 / 转速 / 齿轮比。
  局限：只有曲柄滑块一种机构，没有凸轮、连杆组、带传动。
- 天体（domain-astro-0.1）：开普勒轨道＋近地点点火，校验能量与角动量；
  参数偏心率 / 周期 / 点火增量。
  局限：二体问题＋一次点火，没有 N 体、多段变轨、大气。

建议：三域保持独立，不并入 Claw 对接；Claw 的桥只走两条——
本页的战斗帧（时间＋事件）与节点名映射。以后如果要做
“Claw 版步态”，应以 Claw 的 4 腿＋方块体重心另起一域，
而不是把通用双足步态改个名就当成 Claw 的。
