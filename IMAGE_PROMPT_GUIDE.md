# AI文字生图提示词工程：从失败到精通的实战手册

> 基于OpenAI / Claude / Gemini / DeepSeek / Kimi 五角色娘化项目的完整迭代经验

---

## 一、核心认知：AI的"理解槽位"有限

### 1.1 7±2 法则
AI图像模型（无论是Midjourney、Stable Diffusion还是Nano Banana）对提示词的理解存在**认知槽位限制**。它能稳定记住并执行的视觉锚点大约只有 **7±2 个**。

**后果**：
- 提示词超过200词 → AI开始"平均化"处理，忽略细节
- 五个角色各写200词 → 五个人最后穿上同样的衣服（白袍化悲剧）
- 堆砌20个负面禁令 → AI反向理解，画出奇怪的东西

**解法**：
- 每个角色只保留 **5个核心锚点**：发色 + 服装 + 1个化形痕迹 + 1个姿态 + 1个表情
- 总提示词控制在 **150-250词**（英文）以内
- 负面词只留 **最核心的10条**

---

## 二、风格锚定：先定画风，再定内容

### 2.1 暴力锁定二维动画
MJ默认美学 = 油画厚涂 + 体积光 + 景深模糊 + 写实比例 + ArtStation trending

**必须放在提示词最开头的风格锚**：
```
flat cel-shaded anime style, 2D hand-drawn TV animation aesthetic,
anime production cel look, clean thick black outlines,
flat color blocks with minimal hard shading, no gradient shadows,
Kyoto Animation character design, Bocchi the Rock key visual,
```

**必须封死的MJ毒药**（负面词前10条）：
```
no oil painting, no digital painting texture, no semi-realistic,
no volumetric lighting, no depth of field, no cinematic lighting,
no soft focus, no bokeh, no ArtStation trending, no 3D render
```

### 2.2 中式美学 ≠ 古风元素堆砌
真正要注入的是：
- **留白**：每格至少40-50%空空间
- **虚景**：背景不是画出来的，是"没画"出来的（墨晕、光 streak、渐变消失）
- **宋瓷色调**：天青、米白、藕荷、黛蓝、月灰 —— 降饱和 + 加灰
- **气韵**：人物只占画面六成，剩下四成是呼吸

---

## 三、角色设计：差异化公式

### 3.1 五人五色的铁律
| 维度 | 必须差异化 |
|------|-----------|
| 发色 | 五个人绝对不能重复，甚至不能同色系相邻 |
| 服装体系 | 校服 / 风衣 / 水手服 / 工装 / 毛衣 —— 各成体系 |
| 姿态矩阵 | 前倾 / 正坐微后仰 / 侧身回眸 / 蜷缩侧对 / 低坐仰望 |
| 视线方向 | 看窗外 / 看书 / 回眸看镜头 / 低头 / 看琴 —— 不能全看镜头 |
| 表情温度 | 明亮 / 平静 / 调皮 / 腹黑微笑 / 疏离 —— 一眼可读 |

### 3.2 性格密码
**关键洞察**：她们四个人如果都换成同样的表情和站姿，仅靠发色区分，魅力会暴跌90%。

翻译到AI角色：
- **OpenAI** = 前倾 + 主动面向 + "我有话想说"的开放感
- **Claude** = 微后仰 + 端正坐 + "我不知道"的诚实凝视
- **Gemini** = 侧身回眸 + 手指向上 + "快看这个"的挑衅
- **DeepSeek** = 蜷缩侧对 + 盲眼微笑 + "我知道你的秘密"的深渊式温柔与病娇危险感
- **Kimi** = 低坐姿 + 仰望 + "我记得"的包容

---

## 四、化形设计：亦妖亦人亦仙

### 4.1 西游式变身的表达公式
**错误写法**：`cat ears hairpin, cute fox tail accessory`（变成兽耳娘Cosplay）

**正确写法**：
- **有机残留**：`delicate fin-like ears peek through her hair`（鳍状耳藏于发间）
- **胎记式纹路**：`faint crab-shell pattern glows across collarbone`（锁骨处蟹纹如胎记）
- **瞳孔变异**：`iris contains a faint rotating four-pointed star pattern`（星形瞳孔）
- **未完全 retracted**：`translucent blue whale tail extends naturally from lower back`（鲸尾从身后自然延伸）
- **气息残留**：`bare heels show a faint silvery glow`（足跟银辉如月尘）

**核心原则**：痕迹必须是 **biological/spiritual remnant**（生物/灵性残留），不是 **costume accessory**（服装配饰）。

---

## 五、诱惑与矜持的平衡

### 5.1 从"露肉"退回"气质"
| 错误方向 | 正确方向 |
|---------|---------|
| 衬衫敞开、外套滑落 | 衣物完整，只留一个"不完美"细节（领带松散、袖口卷起） |
| 露肩、露背、光腿 | 日常着装覆盖完整，诱惑来自眼神、发丝、姿态留白 |
| 正面大开腿/蜷缩暴露 | 自然日常姿态，像动画截图而非写真 |
| 表情夸张（张嘴笑、吐舌） | 收敛表情（微笑、平静、专注、温柔），让观者脑补 |

### 5.2 中式含蓄的诱惑密码
- **似露非露**：半滑落的外套 → 改成 `buttoned but slightly rumpled`
- **不经意间**：湿发贴颈、一缕头发落在锁骨上、 barefoot 触地
- **留白想象**：她不看镜头，你看她 → `she does not look at the viewer`

---

## 六、负面提示词：少即是多

### 6.1 负面词的陷阱
写20条负面禁令，AI可能：
1. 忽略后半段
2. 反向理解（把"no text"理解成"画满文字"）
3. 过度补偿（把"no dark gloomy"理解成"必须极度明亮"）

**最佳实践**：
- 负面词控制在 **8-12条**
- 按优先级排序：文本 > 画风污染 > 人体错误 > 构图错误
- 用逗号分隔的短句，不用长段落

### 6.2 核心负面清单模板
```
no text, no oil painting texture, no realistic shading, no 3D render,
no cluttered backgrounds, no high saturation, no animal ears as cute accessories,
no weightless hair, no symmetrical perfect poses
```

---

## 七、迭代策略：从失败中学习

### 7.1 我们的迭代路径
| 轮次 | 问题 | 解法 |
|------|------|------|
| 1 | 有文字标签、画风写实 | 删除文字，锁定赛璐璐 |
| 2 | 格子太瘦长 | 明确4:5方格比例 |
| 3 | Claude螃蟹变紫色 | 修正品牌色为暖橙+栗棕 |
| 4 | 画面太花、MJ味重 | 做减法，留白，封死油画质感 |
| 5 | 缺少中式美学 | 加入宋瓷色、虚景、气韵 |
| 6 | 缺少化形痕迹 | 西游式有机残留设计 |
| 7 | 暴露感太多 | 退回气质诱惑，删除露肤描述 |
| 8 | 性格不够鲜明 | 引入姿态矩阵 |
| 9 | DeepSeek腹黑不足 | 修改姿态为 lounging + 微笑深渊 |
| 10 | 提示词太长导致白袍化 | 精简到每人30词，保留骨架 |

### 7.2 调试 checklist
每轮生成后检查：
- [ ] 五个人服装是否重复？
- [ ] 是否出现文字/水印？
- [ ] 是否有油画/厚涂质感？
- [ ] 化形痕迹是"长出来的"还是"戴上去的"？
- [ ] 表情是否一眼可读？
- [ ] 背景是否太满？
- [ ] 手/关节是否有明显错误？

---

## 八、终极提示词模板

### 8.1 五联格版（精简稳定）
```
flat cel-shaded anime style, 2D hand-drawn TV animation, clean thick black outlines,
flat color blocks, minimal shading, large anime eyes, Chinese Song Dynasty留白,
50% empty space per panel, muted ceramic palette, pure white background,
five equal panels in horizontal row, thin white borders, no text, no watermarks,

Panel 1 — [角色名]: [发色], [服装], [化形痕迹]. [姿态], [表情]. [性格词].
Panel 2 — [角色名]: [发色], [服装], [化形痕迹]. [姿态], [表情]. [性格词].
Panel 3 — [角色名]: [发色], [服装], [化形痕迹]. [姿态], [表情]. [性格词].
Panel 4 — [角色名]: [发色], [服装], [化形痕迹]. [姿态], [表情]. [性格词].
Panel 5 — [角色名]: [发色], [服装], [化形痕迹]. [姿态], [表情]. [性格词].

All five: clean lineart, flat fading backgrounds, youthful faces, same scale,
organic traces only, no costume accessories.

Avoid: text, oil painting, realistic shading, 3D render, cluttered backgrounds,
high saturation, same outfits, cute animal ear accessories, weightless hair.
```

### 8.2 单幅群像版（互动场景）
```
flat cel-shaded anime style, 2D hand-drawn TV animation, clean thick black outlines,
flat color blocks, minimal shading, large anime eyes,
single wide illustration with five girls in one minimalist room, natural depth,
no panels, no text, no speech bubbles, Chinese Song Dynasty留白,
40% empty space, muted ceramic palette, soft lighting from left,

[角色A]: [发色], [服装], [化形痕迹]. [位置], [姿态], [表情].
[角色B]: [发色], [服装], [化形痕迹]. [位置], [姿态], [表情].
[角色C]: [发色], [服装], [化形痕迹]. [位置], [姿态], [表情].
[角色D]: [发色], [服装], [化形痕迹]. [位置], [姿态], [表情].
[角色E]: [发色], [服装], [化形痕迹]. [位置], [姿态], [表情].

Room: bare wooden floor, large window, minimal furniture. Background fades to white.
Avoid: text, oil painting, realistic shading, 3D render, cluttered backgrounds,
same outfits on multiple characters.
```

---

## 九、常见错误与急救方案

| 错误现象 | 原因 | 急救 |
|---------|------|------|
| 五个人穿同样的衣服 | 提示词太长，AI平均化 | 精简到每人只写一件主服装 |
| 手/手指畸形 | 手部动作描述太复杂 | 删除手部细节，只保留姿态 |
| 背景太满像海报 | 环境描写太多 | 删除小物件，只留1-2个环境锚点 |
| 动物痕迹像Cosplay | 用了"accessory/hairpin/costume" | 改用"peek through/glows across/extends from" |
| 表情千篇一律 | 没有明确性格词 | 每人在结尾加1个性格形容词 |
| 格子比例不一致 | 没有明确尺寸指令 | 加`equal panels, same size, 4:5 aspect ratio` |

---

## 十、结语：提示词是减法艺术

**最好的提示词不是写得最多的，而是删得最狠的。**

每当你想加一个"美丽的修饰"时，先问自己：
- 这个细节会帮助AI理解，还是增加混淆？
- 如果删掉它，角色还成立吗？
- 这个描述是"我能看到的"，还是"AI能画出的"？

记住：**AI不是画家，是执行者。你的提示词是施工图，不是散文。**

---

*文档版本：v1.0*
*基于项目：AI五角色娘化（OpenAI/Claude/Gemini/DeepSeek/Kimi）*
*迭代次数：10+轮*
*生成模型：Nano Banana / Midjourney / GPT Image*
