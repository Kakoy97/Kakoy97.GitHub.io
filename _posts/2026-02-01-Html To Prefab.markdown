---
layout: post
title: "独立开发的救星：从 Stitch 到 Unity Prefab，再到单图秒修复"
date: 2026-02-18
author: "Kakoy"
header-img: "img/home-bg-geek.jpg"
mathjax: false
catalog: true
categories: [独立开发, 实战复盘]
tags: [Unity, Stitch, Puppeteer, UI自动化, 工具链]
description: "从手工截图到 HTML→Prefab 自动化，再到单图增量修复（Repair Pipeline）：我如何把 UI 生产链路做成可迭代的工程系统。"
---

# 独立开发的救星：从 Stitch 到 Unity Prefab，再到单图秒修复

对于独立开发者来说，UI 往往是最容易被低估的“时间黑洞”：

> 设计 → 找资源 → 切图 → 拼预制体 → 修细节 → 返工

流程不难，但重复劳动极多。
尤其当你要快速试错时，UI 生产会直接拖慢迭代速度。

这篇文章复盘我最近做的一套工具链升级：
不仅实现了 **HTML → Unity Prefab** 自动转换，还新增了 **单图修复（Repair Pipeline）**，把“修一张图要全量重跑”的痛点彻底打掉。

> 项目地址：
> **https://github.com/Kakoy97/Html-To-Unity-Prefab/tree/main**

![img](/img/htmlToPrefab/01.png)
*(左边是导出效果，右边是效果图)*

---

## 01. 起点：Stitch 很好用，但产物和 Unity 不同路

Google 的 Stitch 能快速生成高质量界面，这点对独立开发非常友好。
问题在于它产出的是 HTML + CSS，不是 Unity 直接可用的切图资源。

我最早的方案是：

1. F12 隐藏文字
2. 手动截图
3. 导入 Unity 切图
4. 手工拼回预制体

缺点很明显：

- 机械劳动太多
- 像素风 UI 容易切边、漏底色
- 每次改版都要重复整套流程

一句话：这活应该交给代码。

![img](/img/htmlToPrefab/02.png)
*(手动在Sprite Editor下切图)*

![img](/img/htmlToPrefab/03.png)
*(使用工具自动导出已裁剪图片)*

---

## 02. 方案抉择：UI Toolkit 是保底，Prefab 是主线

我尝试过 HTML/CSS → UI Toolkit/USS。
技术上可行，但对长期使用 UGUI + Prefab 的开发习惯不够友好。

所以我最终走的是：
**保留 Unity 现有工作流，把 H5 渲染结果自动还原成 Prefab。**

核心思路很简单：

- PSD 是图层树
- DOM 本质也是树
- 能递归 DOM，就能做“自动切图 + 层级还原”

---

## 03. 主流程：HTML → Prefab 的八段流水线

我把系统分成 Node.js 工具端 + Unity Editor 端：

### Node.js（分析与截图）
- `Analyzer`：遍历 DOM，提取位置、样式、旋转、文本、特殊控件
- `Planner`：决定捕获策略（clone/inPlace/rangePart/backgroundStack）
- `Baker`：执行截图
- `Assembler`：生成 `layout.json`

### Unity Editor（构建与落地）
- `NodeBakeRunner`：拉起 Node 脚本
- `BakePipeline`：管理完整流程
- `UiAssetImporter`：统一纹理导入规则
- `PrefabBuilder`：根据 `layout.json` 递归构建 Prefab
- `LayoutIndexBuilder`：生成 `ui_index.json` 索引

---

## 04. 真正的难点：不是“截不截图”，是“怎么截图”

我一开始以为截图是简单操作，后来发现是策略问题：

### Clone 模式
克隆节点到干净区域截图，适合纯图标、简单背景，干净高效。

### InPlace 模式
原位截图，保留上下文，适合 `backdrop-filter`、`mix-blend-mode`、祖先裁剪、旋转上下文等复杂效果。

### RangePart 模式
专门处理 `input[type=range]` 的 track/thumb 伪元素，避免样式丢失。

### BackgroundStack 模式
处理多层半透明叠加，保证层叠结果和网页一致。

---

## 05. 我踩过的三个核心坑

1. **坐标转换**
HTML 和 Unity 坐标系方向不同，必须做物理像素级映射。

2. **旋转处理**
不是所有旋转都该烘焙到图里，有些要留给 Unity 运行时处理。

3. **透明度双重衰减**
CSS opacity 烘焙一次，Unity Image 再乘一次 alpha，会发灰。
需要做透明度解耦，统一在 `renderOpacity` 控制。


---

## 06. 新增能力：单图修复（Repair Pipeline）

全量流水线跑通后，新的痛点出现了：
**修 1 张图也要全量重跑。**

这就是我新增 Repair Pipeline 的原因。

### 设计目标
- 只修单节点图片，不动主链路
- 快速试错，不影响全量稳定性
- 把边缘 case 从主流程剥离，降低回归风险

### 架构思路：双管线分离
- **Export Pipeline**：全量 HTML → Prefab
- **Repair Pipeline**：单图增量修复

两条链路互不干扰，但共享上下文数据（`analysis_tree.json` / `bake_plan.json`）。

![img](/img/htmlToPrefab/04.png)
*(智能修复模式)*

---

## 07. 单图修复怎么工作？

在 Unity 节点（`UiNodeRef`）上点 `Smart Repair` 后：

1. Unity 写入 `repair_manifest.json`
2. 调用 `tool/src/repair/cli.js`
3. `RepairService` 根据 nodeId 回溯上下文
4. 并行执行多策略生成变体
5. Unity 窗口展示候选图
6. 选择满意方案，一键覆盖原图

这套流程把“反复重跑 + 盲调参数”变成了“可视化选择题”。

---

## 08. Smart Repair 的策略组合

`SmartGenStrategy` 会并行跑多个修复策略：

- **ForceCloneStrategy**：背景隔离，适合污染问题
- **ExpandPaddingStrategy**：扩大边界，修切边和阴影裁剪
- **ForceInPlaceStrategy**：保留上下文，修光效/混合模式
- **ColorCorrectionStrategy**：修发灰、色偏

并行生成多个候选方案后，用户直接选最佳变体应用。

---

## 09. 手动调色：把“玄学修图”变成可控参数

除了智能修复，我还加了手动调色模式：

- Brightness（0.5~1.5）
- Contrast（0.5~1.5）
- Saturation（0.0~2.0）

底层通过 CSS Filter 生成效果，再截图固化到产图中。
这让“色差修复”从主观靠感觉，变成可复现的参数化流程。

![img](/img/htmlToPrefab/05.png)
*(色差修复模式)*

---

## 10. 性能对比：从“全量重跑”到“单点修复”

下面是我当前版本在本地开发机上的实测体感（可按你的真实数据替换）：

| 场景 | 旧方式（仅全量） | 新方式（Repair Pipeline） | 提升 |
|------|------------------|---------------------------|------|
| 修复单个节点切边 | 90~120s（全量重跑） | 5~12s（单图修复） | 约 8x~18x |
| 修复单个节点色差 | 90~120s | 6~15s | 约 6x~15x |
| 尝试 3 种不同参数 | 4~6 分钟 | 20~45s（并行变体+选择） | 约 5x~10x |
| 回归风险 | 高（动全链路） | 低（局部替换） | 明显下降 |

> 注：具体时间受页面复杂度、机器性能、浏览器冷启动状态影响。
> 但“修 1 张图无需全量重跑”这个收益是稳定存在的。

---

## 11. 写在最后

这套工具从“自动转换”进化到“自动转换 + 单图修复”，
本质上是把 UI 生产链路从脚本工具升级成了可维护系统。

先把最重复的环节自动化，再把最痛的返工环节增量化。
这两步做完，效率会是质变。

---
