---
name: 拆解视频脚本
description: 用户说「拆解视频」时使用——把短视频反向拆解成可复刻的拍摄脚本：逐镜分镜表（景别/机位/时长/转场）、逐句口播台词稿、结构大纲与钩子归因、可复用模板，Excel 交付。触发词：拆解视频、拆解视频脚本、拆视频。当用户发送视频并要求拆解脚本、分析分镜、提取台词、总结大纲、复刻对标视频、"这条视频怎么剪的"时，同样适用。支持本地文件与目录自动发现。
agent_created: true
---

# 拆解视频脚本

> **触发即执行**：只要用户说出「拆解视频」或同类说法并发来视频，
> 立刻按本文件第 1-6 步调用 scripts/ 下的现成脚本，**不要重新写代码、不要现场造轮子**。
> 脚本已在 `C:\Users\Administrator\.workbuddy\skills\拆解视频脚本\scripts\` 下，参数化且已验证可用。

## 何时使用

适用场景：，要分镜表、脚本、台词、大纲、结构拆解
- 对标账号分析、爆款复刻、同赛道模板提炼
- 需要给别人讲清楚"这条片是怎么做出来的"

## 环境（本机已就绪，无需重装）

Python 解释器（含全部依赖）：

```
C:/Users/Administrator/.workbuddy/binaries/python/envs/default/Scripts/python.exe
```

已装：`imageio-ffmpeg`（自带 ffmpeg 7.1 full build）、`faster-whisper`、`openpyxl`、CUDA 12 运行库。
显卡：RTX 5060 8GB。**整套环境已验证可用，不要再 pip install 任何东西。**

> ⚠️ 如果环境丢失要重装，需先告知用户：venv 2.4GB + large-v3 模型 2.9GB，共约 5.3GB。
> 重装后必须踩的两个坑见文末「已知坑」。

## 六步工作流

所有脚本在本 skill 的 `scripts/` 下，`<视频>` 支持文件路径或目录（自动发现首个视频）。

### 第 1 步｜探底

本机 PATH 里**没有** ffmpeg，它藏在 Python 包里，先取出路径再探测：

```bash
FF="$($PY - <<'PYEOF'
from imageio_ffmpeg import get_ffmpeg_exe
print(get_ffmpeg_exe())
PYEOF
)"
"$FF" -i "$V" 2>&1 | grep -E "Duration|Stream"
```

（嫌麻烦可以跳过——第 2 步一上来就会打印源文件与总时长。）

时长 > 10 分钟要提醒用户：抽帧和处理时间会失控，建议先剪出片段或指定时间段。记下总时长秒数，后续要用到。

### 第 2 步｜场景检测 → shots.json

```bash
python scripts/vd_scenes.py "<视频>" "<工作目录>" [阈值]
```

默认阈值 0.30。输出 `shots.json`（每镜：`idx/start/end/dur/score`）。自动把间隔 <0.6s 的相邻切点合并，避免同一转场被重复计数。

### 第 3 步｜抽帧 + 拼概览图

```bash
python scripts/vd_frames.py "<视频>" "<工作目录>" [并发数]
```

每镜取起点后 25% 处的一帧（避开转场黑帧），缩到 440px 宽并**用 drawtext 在左上角烧镜号**，然后每 12 镜拼一张 `3x4` 网格图到 `grids/`。

这一步是关键的成本控制：**不要逐张读 98 张单帧**，读 9 张网格图即可，图文量大为节省且上下文更连贯。

### 第 4 步｜语音转写

```bash
python scripts/vd_asr.py "<视频>" "<工作目录>" [model]
```

默认 `large-v3` + `int8_float16` + GPU，4 分半音频约 24 秒跑完。VAD 开启，输出带词级时间戳的 `transcript.json`。

`initial_prompt` 要按视频主题改写（告诉模型这是什么领域的说话内容），对专有名词准确度提升明显。

### 第 5 步｜看图写画面描述（这一步必须由 AI 完成）

依次读 `grids/grid_*.jpg`，逐镜写四元组：`(画面内容, 景别, 机位/运镜, 转场)`，并存成一个 `desc.py`：

```python
DESC = {
 1: ("开场：年轻夫妻背对镜头站在院中看工地，院子还是泥地", "全景", "手持固定", "硬切"),
 ...
}
```

读图要点：
- **优先读画面自带的字幕文字**，那通常是作者写的核心文案，比口播更精炼
- 金额/单据/手机屏幕类特写要单独记录，往往是账本线的关键证据
- 景别按「远景/全景/中景/近景/特写」五级判定别过头

同时给 `NOTES`（字典 `{镜号: 备注}`）标注关键信息镜头，例如金额、单据、屏幕内容。

### 第 6 步｜台词对齐 → Excel

```bash
python scripts/vd_excel.py "<工作目录>" "<desc.py路径>" "<输出.xlsx>"
```

desc.py 需提供的字段：`TITLE, DESC, NOTES, FIX, KEY_SHOTS, HOOK_SHOTS, OUTLINE, EXTRA_BLOCKS, TEMPLATE`
（照 `scripts/desc_template.py` 填，缺字段会报错或留空）

台词按**时间重叠度最大化**匹配到镜头；`FIX` 字典按台词起始秒校订同音误识别。

> ASR 中文专有名词错误率不低，「借婚房/建婚房」「婚嫁/婚假」「化粪区/化粪池」这类必然出错，
> **必须对照画面和语义人工校订**，否则台词稿不可用。

## 交付物结构（4 个页签）

| 页签 | 列 |
|---|---|
| 分镜脚本总表 | 镜号 / 时间码 / 时长 / 景别 / 机位运镜 / 画面内容 / 口播台词 / 转场 / 关键信息备注 |
| 逐句台词稿 | 序号 / 开始 / 结束 / 时长 / 台词原文 / 校订后 / 所属镜头 |
| 结构大纲与钩子拆解 | 段落 / 时间 / 镜号 / 内容概要 / 叙事功能 / 关键信息爆点 + 钩子拆解、情绪曲线、完播设计三个补充块 |
| 可复用模板 | 模块 / 做法 / 本片实例（定位、标题公式、钩子、叙事骨架、双暗线、拍摄清单、节奏、口播风格、结尾、卡点提示） |

钩子/关键镜头用底色高亮：**P1 钩子段用红、关键镜头用橙**，一眼能看到重点。

## 参数调优

| 内容类型 | scene 阈值 | 说明 |
|---|---|---|
| 快剪 vlog / 抖音生活类 | 0.30 | 默认，一镜往往只有 2-3 秒 |
| 访谈 / 口播 / 慢节奏 | 0.20 | 切得少，阈值高了会漏切 |
| PPT / 屏幕录制 | 0.45 | 整屏变化才算切，避免逐动画触发 |
| 广告大片 / 多闪白 | 0.35 | 闪白帧会被误判，宁可稍微高一点 |

## 目录结构

```
SKILL.md                          本文件：工作流 + 坑位清单 + 参数调优
scripts/
  vd_scenes.py                    第 2 步 场景检测 → shots.json
  vd_frames.py                    第 3 步 抽帧 + 拼 3x4 网格
  vd_asr.py                       第 4 步 GPU 语音转写
  vd_excel.py                     第 6 步 合成四合一 Excel
  desc_template.py                第 5 步产物的空白模板（复制后填写）
examples/
  case01_婚房院子_DESC.py          真实案例：98 镜画面描述怎么写
  case01_婚房院子_EXCEL.py         真实案例：OUTLINE / EXTRA_BLOCKS / TEMPLATE / FIX 怎么填
```

第一次用建议先翻一下 `examples/` 里的真实案例，比读文档更快上手。

## 完整调用示例

```bash
PY="C:/Users/Administrator/.workbuddy/binaries/python/envs/default/Scripts/python.exe"
SD="C:/Users/Administrator/.workbuddy/skills/拆解视频脚本/scripts"
V="E:/path/to/video.mp4"      # 也支持传目录
W="E:/project/work"           # 中间产物目录

$PY "$SD/vd_scenes.py" "$V" "$W" 0.30     # → shots.json
$PY "$SD/vd_frames.py" "$V" "$W" 6        # → frames/ grids/
$PY "$SD/vd_asr.py"   "$V" "$W" large-v3  # → transcript.json
# ← 读 grids/grid_*.jpg，写 desc.py
$PY "$SD/vd_excel.py" "$W" "desc.py" "拆解包.xlsx"
```

## 已知坑（务必遵守）

1. **源文件路径不要手抄**。长中文名带 `#` 标签（`xxx #记录真实生活 #装修 #xxx.mp4`），手抄必漏。
   所有脚本都用 `os.listdir` 自动发现，**永远不要**在命令里手写完整文件名。
2. **Python 代码必须写成 .py 文件再执行**。不要 `python -c "..."` 内联，
   bash 双引号里正则的 `\d`、`\s` 会被吞掉（表现为 `findall` 返回空列表且无任何报错）。
3. **HuggingFace 直连超时**（本机透明代理返回 502）。必须在脚本里设：
   `os.environ['HF_ENDPOINT'] = 'https://hf-mirror.com'`，并 pop 掉 HTTP(S)_PROXY 环境变量。
4. **CUDA DLL 加载**：Windows 版 nvidia wheel 的 DLL 在 `site-packages/nvidia/*/bin`（**不是 lib**）。
   必须在 import ctranslate2 之前 `os.add_dll_directory()` 注册全部 `*\bin` 与 `*\lib` 目录，
   否则报 `Library cublas64_12.dll is not found or cannot be loaded`。
5. **PowerShell 工具在本机会话无 stdout 输出**，执行系统命令一律用 bash。
6. 结束后若临时装过东西，主动汇报占用并问是否清理（`pip cache purge` 是安全的第 1 档）。

## 质量控制

- 分镜数合理性：短视频通常 1.5-4 秒/镜；若平均 <1 秒说明阈值过低，若 >8 秒说明漏切
- 台词覆盖率：有效语音段应有转写；大片时间无台词且有声，检查是否 VAD 参数过严
- 交付前必做：检查 xlsx 每个 sheet 的行数与预期一致，抽验中间行的内容是否对得上镜号
