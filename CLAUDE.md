# Shattered Crown — Project Notes for Claude

## 项目概览

FIT1073 A3c 作业。Tactics roguelike + 内部 playtest 工具 + GM 辅助。
单 HTML 文件，纯 vanilla JS，无 build step。直接浏览器打开就跑。

**GitHub Pages 上线：** https://ckaii.github.io/shattered-crown/shattered-crown-playtest.html

## 文件

| 文件 | 用途 |
|------|------|
| `shattered-crown-playtest.html` | 主工具 + 游戏（~4500 行单文件） |
| `shattered-crown-website.html` | 旧的展示用网站 |
| `README.md` | GitHub 简介 |

**重要：所有改动都在 `shattered-crown-playtest.html`。**

## 架构

### 五个画面（scenes）
HTML 里有 6 个 `<div class="game-screen">`：
- `screen-title` — 开场
- `screen-hero-select` — 选 4 个英雄
- `screen-map` — 节点地图
- `screen-shop` — 商店
- `screen-ending` — 结局
- `screen-combat` — 战斗（包含原本的 GM Sandbox UI）

切换用 `switchScene(name)`。所有 modal 在 `<body>` 顶层（不在任何 screen 内）。

### OOP Class 层级
```
GameEntity
├─ Token (id,name,hp,mana,atk,def,speed,range...)
│   ├─ Hero (extends, side='ally', equipment[], potions[], baseStats)
│   └─ Enemy (extends, side='enemy', isBoss())
├─ EquipmentItem (canBeUsedBy, sellValue, statSummary)
├─ GameSkill (isPassive, cooldownKey, canBeUsedBy)
├─ CampaignManager — singleton CAMPAIGN
├─ GameMap — singleton GAME_MAP
└─ CombatScene — per combat
```

**Token instance API：**
```js
hero.takeDamage(5)        // 扣血
hero.heal(3)              // 治疗
hero.distTo(enemy)        // 曼哈顿距离
hero.canReach(r, c)       // 移动检查
hero.inRangeOf(target)    // 射程检查
hero.canAfford(mp)        // MP 检查
hero.isMyTurn()           // 回合检查
hero.equipById(itemId)    // 装备
hero.applyLevelUp()       // 升级
```

旧 procedural function（`resolveAttack`、`checkDeath`、`renderGrid` 等）仍然可用，跟 class instance 并存。

### 关键全局
| 变量 | 用途 |
|------|------|
| `tokens[]` | 战斗中所有 token（Hero/Enemy instance） |
| `turnOrder[]` | 当前回合顺序 |
| `turnIndex` | 当前 token index |
| `CAMPAIGN` | CampaignManager instance |
| `souls`, `trust`, `gold` | 全局资源 |
| `skillPool[]` | GM 可编辑的技能池（localStorage 持久） |

### 数据常量
| 常量 | 内容 |
|------|------|
| `ALLY_CLASSES[]` | 7 个职业 template (Fighter/Tank/Rogue/Ranger/Cleric/Wizard/Mage) |
| `PLAYABLE_CLASSES[]` | Daniel 设计的 5 个（不含 Tank/Mage） |
| `ENEMY_CLASSES[]` | 怪物 template |
| `CLASS_SKILL_POOLS{}` | 每职业 5 个技能（首次加载自动塞进 skillPool） |
| `EQUIPMENT{}` | 70+ 件装备（Daniel docx） |
| `STARTER_WEAPONS{}` | 每职业起始武器 ID |
| `MAP_NODES[]` | 8 节点固定地图 |
| `ENCOUNTERS{}` | 节点 → 敌人配置 |
| `LEVEL_DESIGNS{}` | 节点 → 地形 + 出生点（详细 layout） |
| `VISIONS{}` | 灵魂阈值 3/7/12/20 的故事 modal |
| `TRUST_EVENTS{}` | 信任 0/20/80 的故事 modal |
| `TERRAIN_INFO{}` | 地形 emoji + 描述 + tooltip |
| `CLASS_SPRITES{}` | 职业 → emoji portrait + 武器 |
| `POTION_TYPES{}` | 6 种药水 |
| `HELP_TOPICS{}` | 16 个 ? 按钮说明 |

### 回合制规则
1. `rollInitiative()` 用 Speed + 1d4 排序
2. 每 token 一回合：**1 移动（≤ SPD）+ 1 动作（攻击/技能）**
3. `tok.moved` 移动后 lock，`tok.actions=false` 动作后 lock
4. 用完两个 → `maybeAutoEndTurn` 1 秒后自动 nextTurn
5. `isItMyTurn(tokId)` 检查；`enforceTurn(tok)` 阻挡非当前 token 操作
6. 一轮结束（turnIndex 回 0）→ `nextRound()` 重置所有 moved/actions、tick cooldowns、MP regen、状态 tick

### 存档
- `localStorage.shatteredCrownSave_v1` — 完整 campaign state（JSON）
- `localStorage.shatteredCrownSkillPool_v1` — 自定义技能池
- 导出 / 导入 JSON 文件分享给队友

## 常见任务

### 加新技能
编辑 `CLASS_SKILL_POOLS` 添加；首次加载会自动塞进 `skillPool`（可编辑）。
**OR** GM 在网页 Skill Designer 添加，会存进 localStorage。

### 加新装备
编辑 `EQUIPMENT` 添加新 entry。`classes: 'all'` 或 `['Fighter','Rogue']` 限制职业。
Equipment 系统会自动用 `recalcHeroStats(h)` 重算属性。

### 加新关卡
编辑 `MAP_NODES` + `ENCOUNTERS` + `LEVEL_DESIGNS`。三个都要更新。
渲染：`renderMap()` 重画 SVG。

### 加新故事 modal
编辑 `VISIONS{}` 或 `TRUST_EVENTS{}`。结构：
```js
{title, source, text, unlock, pipColor}
```
调用 `showStoryModal(title, source, text, unlock, color, onClose)`。

### 加新动画
CSS keyframes 已经有：anim-hit / anim-crit / anim-heal / anim-skill / anim-shake / anim-die。
辅助函数：`animCell(r,c,cls)`、`animToken(id,cls)`、`fxBlast(r,c,emoji)`、`fxFlyingWeapon(attackerId,targetId,emoji,type)`、`fxParticle()`、`fxCritFlash()`。

### 加新 help 主题
编辑 `HELP_TOPICS{}`。然后在 UI 加 `${helpBtn('topic.key')}`。

## 行为规范

- **不要改 `shattered-crown-website.html`**（旧版网站）
- **修改 HTML 前先想清楚影响**：4500 行单文件，全局耦合
- **避免大 refactor**：作业期内稳定优先
- **优先用 OOP class methods** 而不是新增 free function
- **新 feature 加 help 主题**：Critical Analysis 拿分点
- **每个改动 push GitHub**：玩家直接刷新 https://ckaii.github.io/shattered-crown/shattered-crown-playtest.html
- **git commit 必须用 WSL**：`wsl -e bash -c "cd '/mnt/c/...' && git ..."`（Windows bash 没 SSH key）

## Playtest 模式开关

- **GM Sandbox**：Title → ⚙ GM Sandbox Mode。允许放置任何 token + 自由操作
- **Campaign**：Title → ☠ New Run / ▶ Continue Run。强制回合 + 故事 + 存档
- 两个模式共用同一战斗系统

## 接下来想做的

（按 user 优先级排）
- [ ] 真正的 PNG sprite（Kenney.nl 包）
- [ ] Aldric boss 多阶段
- [ ] 让 Sword 真的吃属性 → Varn 越战越强但 Trust 加速崩
- [ ] AI 自动敌人回合（现在 GM 手动控）
- [ ] Sound effects

## 联系

- Repo: github.com/Ckaii/shattered-crown
- 项目用户：FIT1073 team（Daniel 等）
