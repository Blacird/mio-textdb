# MIO 文本库（四语对照）

从《MIO: Memories in Orbit》游戏资源中提取的文本库，**运行时完全不依赖游戏本体**。

当前收录 **2,155 条**文本，含简体中文 / 繁体中文 / 英语 / 法语四语对照。

## 文件说明

| 文件 | 用途 |
|---|---|
| `mio_text_tool.html` | 离线检索工具，双击即可用（数据已内嵌，无需联网/服务器） |
| `mio_text.db` | SQLite 数据库，含 `text_unit` / `speaker` / `credit` 表与 FTS5 全文索引 |
| `text_all.csv` | 四语对照表，Excel / 表格软件可直接打开（UTF-8 BOM） |
| `source_manifest.txt` | **数据来源清单**：原始 CSV 的路径、大小、SHA-256 校验值，以及加工与排除的完整记录 |
| `excluded_merged.txt` | 被合并 / 去重的 376 条原始条目及其归属对象 |
| `excluded_empty.txt` | 因四语全空而排除的 7 条条目 |

## 数据来源

游戏内本地化表，位于 `flamby/misc_files.gin` 的 `assets/loc/` 下：

| 原始文件 | 条目 |
|---|---|
| `assets/loc/ockit/spoken.csv` | 语音演出台词 171 |
| `assets/loc/ockit/non_spoken.csv` | NPC 对话 / 系统文本 1,349 |
| `assets/loc/ockit/other.csv` | 名词、UI 标签、区域标题卡 1,022 |
| `assets/loc/a/credits_EXPORT.csv` | 制作名单 420（存于 `credit` 表） |

原始表含 15 种语言，本库仅取**简中 / 繁中 / 英语 / 法语**四列。

> 原始 CSV **始终保持只读**，未作任何修改，作为数据来源的凭证保留。
> 具体的文件路径、大小、SHA-256 与加工记录见 `source_manifest.txt`。

### 三个层级

```
原始 CSV（15 语种，只读证据）        4 个文件
      ↓ 归一化
2,538 条（四语，含被拆散的碎片）
      ↓ 合并 312 + 去重 64 + 排除空条目 7
2,155 条（进入统计与检索）
```

## 字段含义

| 字段 | 说明 |
|---|---|
| `key_id` | 文本唯一 ID，如 `TXT_GA_ROOT_WRONGTURN#1`、`VO_TW_ASMA_COME#1` |
| `speaker_key` | 说话人原始标识，如 `NPC_ARJUNA`、`ASMA`、`HACKER` |
| `speaker_zh` | 说话人中文名（由名词表反查，如 `阿丝玛`） |
| `category` | `dialogue` 对话 / `term` 术语 / `text` 系统文本 / `voiceover` 语音演出 / `mumble` 循环嘟囔 / `item` 物品 / `trinket` 调整元件 / `curio` 古董 / `keepsake` 纪念品 / `memory` 记忆 / `letter` 信件 / `pearl` 「珠」 |
| `area_code` | 区域代码，如 `LQ_CITY`、`GA_BOU`、`TW_ASMA` |
| `area_zh` / `area_en` | 区域正式地名，由游戏内区域标题卡反查，如 `都会 / Metropolis` |
| `title_*` | 条目标题（四语），如信件名、物品名 |
| `author` / `author_zh` | 信件或文档的作者 |
| `doc_date` / `doc_date_zh` | 年份，如 `Year 984 / 984 年` |
| `desc_*` / `flavour_*` | 物品的功能说明与设定风味（各四语） |
| `once` / `mumble` / `condition` | 是否一次性播放 / 是否循环嘟囔 / 触发条件 |
| `en` / `fr` / `zh_s` / `zh_t` | 英语 / 法语 / 简体中文 / 繁体中文 |

## 查询示例

```sql
-- 某角色的全部台词
SELECT zh_s, en FROM text_unit WHERE speaker_key='NPC_SHII' ORDER BY key_id;

-- 某区域所有对话
SELECT key_id, speaker_zh, zh_s FROM text_unit
WHERE area_code='LQ_CITY' AND category='dialogue';

-- 所有信件及其年份
SELECT key_id, title_zh_s, author_zh, doc_date_zh, zh_s
FROM text_unit WHERE category='letter' ORDER BY doc_date;

-- 全文检索（trigram 索引，需 >=3 字符）
SELECT key_id, zh_s FROM text_fts WHERE text_fts MATCH '安港';
```

## 数据处理

### 一、合并被拆散的元数据（312 条）

游戏把同一内容拆成多条记录，共享一个 base key。本库按下表合并：

| 原后缀 | 并入为 | 数量 |
|---|---|---|
| `_title` / `_TITLE` | 标题（四语） | 66 |
| `_author` | 作者 | 28 |
| `_date` / `_year` | 年份 | 27 |
| `_DESC` | 说明 | 11 |
| `ITEM_NAME_*` / `ITEM_DESCRIPTION_*` / `ITEM_FLAVOUR_*` | 同一物品的名称 / 说明 / 风味 | 176 |
| `X_TITLE` + `X_DESCRIPTION` | 弹窗标题 + 说明 | 4 |

例：一封完整的信由四条记录组成，合并后为

```
充满希望的结果    Hopeful Findings · 充滿希望的發現 · Des résultats prometteurs
LETTER_SPREAD · letter · 智 · 995 年
简中 萨姆斯克，有好消息！…
```

> **未合并**：`_CAPUCINED`（卡普辛装饰版）、`_Y`/`_N`（问答分支）、`_XBOX`/`_PS`（平台变体）——判定为独立内容。

### 二、术语去重（64 条）

同一词条在游戏里有多个用途不同的 key，例如「眸 / The Eye」同时存在于 `GUIDE`（图鉴）和
`ASMA_TITLE_CARD`（区域横幅）。本库按 **简中 + 英文** 作签名去重，只保留一条：

- 保留优先级：通用词条 > 区域横幅 / 悬停提示 / 存档点名称
- 共折叠 64 条，术语表最终 655 条

> 语音台词中重复的句子**未去重**：它们出现在不同结局分支里
> （如 `VO_EN_KHLIA_ENDING#6` 与 `VO_EN_KHLIA_ENDING_D#1` 是同一句台词的复用），合并会丢失分支信息。

### 三、排除空条目（7 条）

四语全空、无任何内容的条目直接排除，不进入统计，名单见 `excluded_empty.txt`。

## 界面说明

**筛选**
- 顶部类别索引：`全部 / 无中文 / 对话 / 术语 / …`，类别名中文化，悬停显示原始英文标识
- 角色、区域、子类三个下拉**随类别动态变化**：只列出当前类别里存在的值，无可用值时自动禁用
- 角色与区域均为「中文（English）」对照，如 `阿茹娜（Arjuna）`、`都会（Metropolis）`

**排版**
- 内容区上限 1280px 居中，宽屏两侧留白，正文段落限 44em（约 44 汉字/行）
- 分栏：**术语三栏，其余两栏**；也可手动切换 1 / 2 / 3 栏
- 语言顺序固定为 简中 → 英文 → 繁中 → 法文；无中文的条目自动补显英法，并标注「无中文」

**浮窗**
- 点击任意条目弹出浮窗，正文 15px / 1.85 行高，四语全部展开
- 显示标题与归属标签（KEY、类别、作者年份、说话人、区域），标签可点击跳转到对应筛选
- 关闭方式：右上角 `×`、点击遮罩、按 `Esc`；选中文字时不会误触

## 标记处理

原始文本中的游戏标记已做可读化处理：

| 原始 | 显示 |
|---|---|
| `%D` | 换行（游戏内换行符） |
| `<sustain>` | ［延长］ |
| `<color=ff0000>…</color>` | 按原色渲染 |
| `<Btn_interact>` / `<Btn_jump>` 等 | ［交互键］/［跳跃键］等 |
| `<key=27>` | ［按键 27］ |
| `<icon>…</icon>` | ［图标］ |
| `<b>` / `<large>` | 粗体 / 放大 |
| `<redacted>` / `<expurgé>` | ［数据删除］ |

## 说明

- 文本内容本身未做任何删改，仅做上述标记渲染；`text_all.csv` 与数据库内保留**原始标记**。
- 区域地名与说话人名均来自游戏自身数据，未人工翻译。
- 仅用于个人研究与阅读；全文内容版权归 Douze Dixièmes / Focus Entertainment 所有。
