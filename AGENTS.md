# AGENTS.md — 魔兽世界·无限服全景地图辅助器

## 项目性质
纯前端离线工具，无后端、无构建、无依赖安装。双击 `index.html` 即可运行（已验证 `file://` 直开可用）。
Tailwind 与 Google Fonts 走 CDN，需要联网才有样式；断网时页面仍能用（自定义 CSS 在 `css/style.css`）。
底图改为真实大陆美术后，**`assets/` 目录必须与 `index.html` 一起分发**（相对路径引用，`file://` 直开可用）。

## ⚠️ 最重要：脚本存在两份副本
`index.html` 里的**内联 `<script>`** 与 `js/app.js` 是**同一份代码的两份副本**。

- 浏览器实际执行的是 **`index.html` 里的内联脚本**，`js/app.js` 只是同源副本（README 里称"独立提取"，容易误解成 index.html 会引用它 —— 并没有）。
- `css/style.css` 同理，也是 `index.html` 内联 `<style>` 的副本。
- **改任何逻辑必须两处同步**，否则改 `js/app.js` 不会影响实际运行结果。

当前状态：两份脚本已做到**字节级完全一致**。改完后用下面这条命令复核：

```bash
python3 -c "
import re
h=open('index.html',encoding='utf-8').read(); js=open('js/app.js',encoding='utf-8').read()
m=re.search(r'<script>\n(.*?)\n[ \t]*</script>', h, re.S)
print('一致' if m and m.group(1)==js.rstrip('\n') else '不一致')"
```

同步方式（以 `js/app.js` 为唯一真源回写 HTML）：

```bash
python3 - <<'PY'
import re
h=open('index.html',encoding='utf-8').read(); js=open('js/app.js',encoding='utf-8').read()
m=re.search(r'<script>\n(.*?)\n[ \t]*</script>', h, re.S)
h=h[:m.start()]+'<script>\n'+js.rstrip('\n')+'\n  </script>'+h[m.end():]
open('index.html','w',encoding='utf-8').write(h)
PY
cp index.html wow_infinite_map_helper.html   # 备份副本必须一起更新
```

`wow_infinite_map_helper.html` 是 `index.html` 的完整备份，README 声明两者相同 —— 任何改动后用 `cp` 覆盖，或 `md5` 复核。

## 数据结构（`js/app.js` 顶部四个常量）
- `MAP_LAYOUT` — 真实地图布局常量：`viewBox`、`scale`，以及每块大陆的 `continents[1414|1415]`
  （卡利姆多 / 东部王国）：`img` 底图路径、`crop` 在美术图里的裁剪原点、`x/y/w/h` 在全景画布上的位置与尺寸、
  `cropW/cropH` 裁剪后的像素尺寸。
  **底图 `<image>`、大洲标题、无尽之海标签、大漩涡位置全部由 `applyMapLayout()` 在运行时写入 DOM，
  不要在 HTML 里写死坐标** —— 素材重算后 HTML 里的硬编码会立刻与数据脱节。
- `ZONES` — 50 个区域（卡利姆多 21 + 东部王国 29），含 `poly` 多边形坐标、`cx/cy` 中心、
  `detail: {cont, x, y, w, h}`（该区域在**全景画布坐标系**里的裁剪框，用于详情图取真实地形）、
  `pois` 子坐标（每个 POI 额外带 `wx/wy`，同样是全景画布坐标）
  - 其中 `riverglades`（河林，ek，Lv.33-54）与 `zephras`（泽风岛，位面浮空岛，Lv.1-14）是
    《无限》服航线落点新增区，**边界为示意轮廓**（客户端未提供对应掩膜），位置取自官方客户端
    `ZoneShapes.lua` 的 UiMapID 2548 / 2521，经大陆 UV 标定换算。
    二者暂无 `BORDER_PASSES` 陆路连接，详图显示「0 个通道」是预期行为。
  - 标着 `(无限服)` 的 6 个区（泰拉比姆 / 奎尔萨拉斯 / 吉尔尼斯 / 库尔提拉斯 / 吉利吉姆岛 / 拉匹迪斯岛）
    是无航线指向的占位区，保留以维持地图完整性。
- `BORDER_PASSES` — 58 个跨区路口，`from`/`to` 是 ZONES 的 `id`，`x/y` 是全景画布坐标
- `TRANSIT_ROUTES` — 14 条航线（飞艇 6 / 船只 7 / 地铁 1），`fromZone`/`toZone` 也是 ZONES 的 `id`，
  `x1/y1/x2/y2/cx/cy` 是全景画布坐标
  - 已按《无限》服官方航线表核对定稿（2026-11）：删 4 条虚构、增 3 条官方、改写 3 条。
    **不要再把旧世经典航线当《无限》新增标**，也不要凭空新增落点不存在的航线。
- `DUNGEON_QUESTS` — 18 个部落练级副本（右侧栏第 4 页签「📜 副本任务」）：
  副本字段 `id/emoji/name/en/level/zones[]/notes[]/quests[]`，
  任务字段 `n`(任务名) / `g`(NPC·地点) / `lv`(等级) / `s`(共享性：`yes`可共享|`no`不可共享|`partial`共享复杂)
  / `scope`(阵营：`双方`|`部落`|`共通`) / `note`(前置与注意事项)。
  `zones[]` 里的 id 必须是 ZONES 的 id —— 渲染成可点击的区域按钮，点一下跳该区详图。

新增数据时三处必须对齐：`from`/`to`/`fromZone`/`toZone`/`zones[]` 引用的 id 必须在 `ZONES` 里存在。

**坐标系只有一套：全景画布 = `viewBox 0 0 1600 940`。** 区域多边形、路口、航线端点、POI 全部在这个坐标系里，
详情图只是把它的一个局部窗口映射到 800×520。

## 副本任务清单的搜索/筛选语义（`renderDungeonQuestList`）
两套筛选**互相独立**：关键词搜索（`q`）与共享性筛选（`share`）。
- `matchDungeon(d)`：副本名 / 英文名 / 副本备注命中 → **整本命中**，展示该副本全部任务（仍受共享性筛选约束）。
- `matchQuest(x)`：任务名 / 接任务 NPC / 任务备注命中 → 只展示命中的那条任务。
- **踩过的坑**：早期版本先按任务级过滤、再判断副本是否为空并跳过，导致搜「黑石」时
  副本名明明命中，任务却因不含关键词被全部滤掉 —— 黑石塔上下层整本消失、黑石深渊显示「0 个任务」。
  正确做法是**副本级命中优先于任务级过滤**。
- 副本既无可见任务又无备注 → 直接隐藏该副本（否则会出现一屏「本副本任务均被过滤掉了」的噪音）。
- 统计行「共 N 个副本 · M 个任务 · 其中 K 个可直接共享」全部由 `DUNGEON_QUESTS` 实时推导，**不要写死数字**。

## 计数展示：不要写死数字
所有界面上的数量角标都通过 `syncDynamicCounts()` 从数据数组实时推导（`buildAtlasSummary()` 生成顶部副标题）。
**不要再写死 48/58/14 这类数字** —— 之前的 bug 就是数据更新了但界面仍显示旧值（44/52/12），且 `returnToWorldMap()` 里单独写死了一份，导致按 ESC 返回后统计错误。

## 详情图标签避让（`renderZoneSubMapSvg`）
详情图是 800×520 的 SVG，背景是该区域的**真实地形裁剪图**（由 `getDetailView()` 把
`zone.detail` 的裁剪框 + 相邻路口坐标一起算出视口，再把大陆底图 `<image>` 铺进去）。
边界路口标签框、POI 名称、POI 坐标都可能落在很近的位置。
`createLabelPlacer()` 负责避让，**必须遵守两阶段顺序**：

1. **阶段一**：算出全部节点坐标并 `reserve()` 所有图标占位（路口节点 r=15、POI 图标 r=13）+ 区域大标题 + 航线去向标签框
2. **阶段二**：才逐个 `place()` 标签

踩过的坑：早期版本在遍历 POI 的循环里才 `reserve()` 自己的图标，导致**靠前的 POI 标签会压到靠后的 POI 图标**上。所有图标必须先一次性占位完再放标签。

`place()` 的兜底策略是"选重叠面积最小的候选位"，不会退回默认位置。标签被挪动超过阈值时会画一条虚线引线连回节点。

### 全景图标签也要用同一套 placer
`renderPanoramicMap()` 里 50 个区域名（★主城 / 区域名 / 等级 三行）**必须**走 `createLabelPlacer()`：
先 `reserve()` 路口节点、航线徽章、航线端点、寻路高亮节点，再 `place()` 区域名块。
直接用固定的 `zone.cx/zone.cy` 居中写法会大面积压字。

### 航线去向标签框的四条边分栏
详情图的航线标签框按航线方向贴到视口边缘：左列 `x≈12+tagW/2`、右列 `x≈788-tagW/2`、
上中 `x=400,y≈90`、下中 `x=400,y≈470`，**同列纵向每次错开 54px**。
四条边各占独立的一列，列与列之间才不会互相压住。
标签框宽度要按文字实际宽度算（`estimateLabelWidth`），写死 156 会被长地名撑出框外。

### 详情图 SVG 的容器高度
`#zoneDetailSvg` 外面必须套一个 `aspect-ratio: 800/520` 的定位盒（`position:relative` + `absolute inset-0` 的 svg）。
之前 svg 直接用 `w-full h-full flex-1`，在右侧栏很长时会被 grid 行高撑到 2000px，
`preserveAspectRatio` 的 meet 缩放会让内容**垂直居中到视口外**，上半屏全是空白。

## 验证方式
纯静态站点，用 Node 跑语法与数据校验最省事：

```bash
node --check js/app.js                                  # 语法
node -e "..."                                             # 统计 ZONES/BORDER_PASSES/TRANSIT_ROUTES 数量、校验 id 引用
```

浏览器侧可复用的检查点：`file://` 打开 → 点区域下钻 → 按 ESC 返回 → 核对顶部统计 → 导出 JSON → 校验导出文件里的 `zoneCount/borderPassCount/transitRouteCount` 与实际数组长度一致。

副本任务页签的检查点：切到「📜 副本任务」→ 搜「黑石」应命中 3 个副本且**每个副本都展示全部任务**
（只命中任务名才只显示那一条）→ 共享性切「不可共享」应得 7 个任务 → 点副本下的 🗺️ 区域标签应跳到该区详图。

## 真实地图素材的生成管线（`/tmp/wowmap/`，不在项目里）
素材来自 GitHub `Sectimus/HighFidelityWorldMap` 的 release（`client.zip` → `WorldMap/Original/`、`ZoneMasks/`、`data/*.lua`）。
重跑顺序：

```bash
python3 /tmp/wowmap/gen_mapdata.py   # 1) 提取轮廓 + 导底图 + 生成 mapdata.json
python3 /tmp/wowmap/patch_app.py     # 2) 把 mapdata.json 写回 js/app.js（会重置 poly/detail/坐标）
python3 /tmp/wowmap/patch2.py        # 3) 重新打标签避让 + 详情图真实底图的交互补丁
python3 /tmp/wowmap/patch3.py        # 4) 详情图标签框宽度/分栏/夹取修复
```

**⚠️ patch_app.py 不是幂等的**：它会再插一份 `detail:`、再追加一份 `wx/wy`、再插一份 `MAP_LAYOUT`。
重跑管线必须先 `cp /tmp/wowmap/app.js.bak js/app.js`（原始无地图数据版本）再依次执行 2→3→4。

### 关键坑
- **美术图的海面是分块拼接的**（纯黑外框 + `[26,50,69]` 深蓝方块 + 墨绿方块）。
  只按亮度阈值（`max(RGB) > 25`）做透明化会留下成片矩形色块，
  必须用**颜色分类**：`dark = max<=22`，`ocean = (B>=R+10)&(B>=G+6)&(max<=135)`，两者都判为水。
- **美术图里还有一条孤立细线残片**（卡利姆多原画 x653-696 / y174-188，深橄榄色），
  用连通域细长比（`bbox面积/像素数 > 4` 且面积 < 500）剔除，真实离岛的细长比都在 2 以内，不会误删。
- **大陆 u/v → 美术像素不能用 `x=u*1024, y=v*768`**，必须用拟合值（见 `fit4_1414.json` / `fit4_1415.json` 的 `fit_px`）。
- `ZoneShapes.lua` 的 `[947]` 块会覆盖 `cont` 里 1414/1415 的大陆归属，要单独维护 `continent_of`。
- `np.cross` 对 2D 向量返回标量，RDP 要手算垂距。
- JS 数据块定位不能用大括号扫描（字符串里可能有花括号），改用缩进匹配 `^      \{$` → `^      \},?$`。
- 浏览器 `fill` 必须用 ref；`click`/`select_option` 可用 selector；**隐藏 tab 里的按钮用 selector 点不会触发 handler**，要先把 tab 切出来。

