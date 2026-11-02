# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 这个仓库是什么

不是一个可直接运行的应用，而是一个 **Agent Skill 包**（无 root package.json、无测试、无 lint）：两个 skill 文件夹 + 文档 + 演示素材。

- `skills/digital-human-koubo/` — 主 skill，做数字人口播视频的完整流水线（写稿 → 生成 → 质检 → 对齐 → Remotion 剪辑 → 封面）
- `skills/finance-explainer/` — 可选引擎，纯图表、不出镜的财经长片（episode.json 拍表驱动）

改这个仓库 = 改 skill 的指令、脚本和数据契约，不是"跑起来看看"。改之前先读 `SKILL.md`：它是 agent 实际执行的流程说明书，脚本和 references 都服务于它。

## 常用命令

所有 Python/shell 脚本都以**剪辑工程目录为 CWD** 运行（`assemble.py` 读 `gen/`、`cut/`，`align.py` 读 `cut/qa.json`），不要在本仓库根目录跑。

```bash
# 本机环境体检（缺什么列什么装法，不自动安装）
bash skills/digital-human-koubo/scripts/doctor.sh

# 起一个剪辑工程（复制模板 + 克隆 node_modules + 合成音效/占位配乐）
bash skills/digital-human-koubo/scripts/new_edit.sh huazi-916 <工程目录>
bash skills/digital-human-koubo/scripts/new_edit.sh lin-169 <工程目录>
bash skills/digital-human-koubo/scripts/new_edit.sh reference/tutorial-169 <工程目录>   # launch / multihost 同理
bash skills/finance-explainer/scripts/setup_engine.sh <工程目录>                       # finance-explainer 引擎

# 口播生成流水线（工程目录内）
python3 <skill>/scripts/build_tasks.py script.json tasks.json --purpose "…"
python3 <skill>/scripts/rangedl.py                       # 读 urls.txt，6 段 Range 并发下载
python3 <skill>/scripts/assemble.py script.json ids.json --out 成片_完整版.mp4
python3 <skill>/scripts/misread_check.py script.json     # 查念错字
python3 <skill>/scripts/patch_words.py script.json patch.json --ref s05,s24
python3 <skill>/scripts/align.py script.json --out <工程>/src/tl.json

# 小林式（A）额外步骤
python3 <skill>/scripts/lin_subs.py script.json --tl <工程>/src/tl.json --out <工程>/src/cues.json --disp disp.txt
python3 <skill>/scripts/lin_scenes.py plan.json --tl <工程>/src/tl.json --out <工程>/src/scenes.json --face
python3 <skill>/scripts/lin_sfx.py <工程>/src/scenes.json --out audio
python3 <skill>/scripts/lin_mix.py --voice cut/joined.mov --bgm 曲子.mp3 --sfx audio/sfx_bus.wav --out <工程>/public/mix.wav

# 封面（默认每条都做）
python3 <skill>/scripts/make_cover.py cover.json --candidates
python3 <skill>/scripts/make_cover.py cover.json
python3 <skill>/scripts/cover_first.py 成片.mp4 cover/cover_16x9.jpg 交付.mp4

# 渲染前校验 + 渲染 + 出片
python3 <skill>/scripts/check_edit.py <工程>/src
npx remotion still  src/index.ts <Comp> out/a.jpg --frame=<N>
npx remotion render src/index.ts <Comp> out/raw.mp4 --concurrency=6
ffmpeg -i out/raw.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11 -c:v libx264 -crf 20 成片.mp4
```

Composition id：`Huazi`（huazi-916，1080×1920）· `Lin`（lin-169，1920×1080）· `Film`（launch-169）· `Tutorial`（tutorial-169 / multihost-169）· `Gallery`（packaging 图鉴）· `Episode`（finance-explainer 引擎，尺寸来自 episode.json）。

## 架构

三层，改任何一层都要同时看另外两层：

**1. 指令层** — `SKILL.md` 描述 agent 的执行顺序（准备 → 选风格 → 写稿 → 报价 → 生成 → 下载 → 回听 → 对齐 → 剪辑 → 封面），`references/*.md` 是各步细节；`persona.json` 是形象/音色/提示词模板/价格，`build_tasks.py` 直接读它拼 prompt 和估价。

**2. 数据契约层**（跨越步骤的中间文件，字段改动会波及下游脚本和模板）：

```
brief.json  风格/比例/时长/主播  ← 用户选择
script.json [{id,shot,t,host,lang,sub}]   shot = full|pip|vlog|depth
tasks.json  → Renoise create_tasks → ids.json {段 id: taskId}
cut/qa.json + cut/<id>.json  ← assemble.py 的 whisper 结果
tl.json     {total, segs[{start,end,text,chars{字符下标: 秒}}]}  ← align.py
edit.json   (B)   plan.json → scenes.json (A)   每期重写 Scene (C/D/E)
cover.json  → cover_16x9/9x16/3x4.jpg
```

时间一律写成 `"段:词"`：`"3:"` 段首、`"3:end"` 段尾、`"3:关键词"`、`"3:词#2"`，或直接写秒。A 风格的 `plan.json` 用 `"@s07:两千|-0.2"`。改契约就要同步改 `check_edit.py`（校验 `段:词` 能否在 tl.json 解析）和参考文档。

**3. 渲染层** — Remotion 4.0.408 工程。`huazi-916` 是真正数据驱动的填空模板；`lin-169` 由 scenes.json 驱动 13 种场景；`reference/*` 是出过的成片工程，**保留布局/字幕/音效/过渡，按本期稿子重写每段 Scene**，不是填空。`templates/packaging/src/kit/` 是 47 个包装组件的共享实现，`new_edit.sh` 起工程时把 kit 一并带进 `src/kit/`，模板里用 `<PackLayer items={edit.pack} …>`。

## 修改时的约定

- `assemble.py` 拼接前把每段音频 pad/trim 到画面帧长（否则几十段拼下来声音累积落后口型）；`--rejoin` 只重拼不重转写。
- 数字人视频轨必须 `muted`，人声单独铺一条，且先用 `scripts/voicefx/unify.py` 统一频谱 + 同一房间混响，否则段间齿音差和梳状滤波会很明显。
- 判断念错不能用 `assemble.py` 的转写（带原文 `--prompt` 会把错字纠正回去），只有 `misread_check.py` 的中性提示转写可信；修法是用 `patch_words.py` 换整句、切在停顿上，不要整片换音色。
- 抠像（`cutout_video.py` / `personmask.swift`）依赖 macOS Vision；其余脚本跨平台。
- 模板的 `node_modules` 用 `cp -cR` 从本机已有 Remotion 工程克隆（npm 源常 SSL 失败），`new_edit.sh` 里有一串候选路径，可用 `DH_NODE_MODULES_FROM` 覆盖。
- 仓库不带第三方音频：`make_assets.py` 用代码合成音效和占位配乐，`new_edit.sh` 自动跑。
- 生成花用户积分：任何 `create_tasks` 之前先报「段数 · 秒数 · 预估 credits · 余额」并等明确同意；先出第 1 段给用户确认音色。
- 文档是双语的：`README.md`（英）/ `README.zh-CN.md`（中）需同步改；`SKILL.md` 和 `references/` 用中文。

## 提交

仓库里的历史提交用 `skill:` / `docs:` 前缀（例：`skill: make a cover by default and splice it in as the first frame`）。涉及第三方依赖或许可证变化时同步更新 `THIRD_PARTY.md`。
