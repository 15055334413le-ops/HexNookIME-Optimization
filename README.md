# HexNookIME 优化方案

> 目标：提升中文拼音输入法的**候选准确度**。
> 本文整理自对 GitHub 上主流开源输入法项目的调研，重点阅读了白霜、万象、LIME 三个项目的 README。

## 结论先行

**候选不准，主要靠“好词库 + 准词频”加“语言模型”一起解决。** 开源圈基本都是在 Rime 引擎上做这两件事。

## 一、提升候选准确度的四个层次

| 层次 | 解决什么问题 | 代表项目 |
|---|---|---|
| **1. 词库质量** | 词收得全，废词少 | [iDvel/rime-ice](https://github.com/iDvel/rime-ice) 雾凇拼音（约 1.96 万星），长期维护的简体词库，是很多方案的基础 |
| **2. 词频准确** | 常用词排在前面 | [gaboolic/rime-frost](https://github.com/gaboolic/rime-frost) 白霜拼音（约 3.7k 星） |
| **3. 语法模型（整句）** | 打长句时能选对整句，不再逐词拼 | [lotem/librime-octagram](https://github.com/lotem/librime-octagram)（八股文插件）、[rimeinn/octagram-data](https://github.com/rimeinn/octagram-data)，以及万象、墨奇各自的模型 |
| **4. 大模型** | 联想和上下文理解 | [xushengfeng/lime](https://github.com/xushengfeng/lime)，用 Qwen3-0.6B 驱动拼音输入 |

## 二、关键做法

### 2.1 白霜拼音的词库流程

目前开源圈公认效果最好的方案之一。

1. **先做减法**：删掉冷僻词（频率为 1 且分词器分不出的词）、不健康词汇，以及“的吧”“的了”这类不是词的词条。
2. **重新统计频率**：用约 7.45 亿字的高质量语料，经 jieba 分词，重新统计字频和词频并归一化。
3. **用评测说话**：配套项目 [gaboolic/rime-schema-compare](https://github.com/gaboolic/rime-schema-compare) 可以复现各方案的准确率对比。
   > 白霜自称在关闭云拼音的条件下超过商业输入法，该结论**尚未独立验证**。
4. **工具和语料**：
   - [jieba](https://github.com/fxsjy/jieba)：分词
   - [pypinyin](https://github.com/mozillazg/python-pinyin)：注音
   - MNBVC：约 33TB 中文语料
   - [kenlm](https://github.com/kpu/kenlm)：训练 n-gram 语言模型

### 2.2 万象拼音

[amzxyz/rime-wanxiang](https://github.com/amzxyz/rime-wanxiang)（约 4.7k 星）

- 词库**带声调标注**，结合语法模型来区分同音词。
- 提供**辅助码**：拼音后面加一两个字形码，用来精确筛选候选。

### 2.3 在 Rime 中开启语法模型

摘自白霜的 README，在方案的 `*.custom.yaml` 里加入：

```yaml
patch:
  grammar:
    language: zh-moqi   # 模型文件名，可选：八股文 / 墨奇 / 万象模型
    collocation_max_length: 5
    collocation_penalty: -14
    non_collocation_penalty: -4
  translator/contextual_suggestions: true
```

### 2.4 LIME（大模型方案）

**思路**：让大模型逐字预测下一个词，用用户输入的拼音约束每一步可选的字。

**优点**：联想能力不错。

**作者自述的问题**：

- 长句不够稳定；
- 需要常驻一个 AI 服务，按键内容会发给该服务，存在隐私风险；
- 没有生词记忆。

## 三、其他有用的资源

| 类别 | 项目 | 说明 |
|---|---|---|
| 词库转换 | [studyzy/imewlconverter](https://github.com/studyzy/imewlconverter) | 深蓝词库转换（约 1 万星），可将搜狗细胞词库等格式导入 Rime |
| 百万级词库 | [wuhgit/CustomPinyinDictionary](https://github.com/wuhgit/CustomPinyinDictionary) | 适用于 Fcitx5 和 Gboard |
| 拼音数据 | [mozillazg/pinyin-data](https://github.com/mozillazg/pinyin-data) | 处理多音字 |
| 引擎本体 | [rime/librime](https://github.com/rime/librime) | Rime 核心引擎 |
| Windows 前端 | [rime/weasel](https://github.com/rime/weasel) | 小狼毫 |
| Windows 前端 | [gaboolic/moqi-im-windows](https://github.com/gaboolic/moqi-im-windows) | 墨奇，内置白霜 |

## 四、建议路线

### 路线 A：只想用得更顺手

- Windows 上装**小狼毫**或**墨奇**；
- 方案选**白霜**或**万象**；
- 开启语法模型（见 2.3）。

这套组合在关闭云输入的条件下，已经很接近搜狗、微软拼音的体验。

### 路线 B：自己做输入法或词库（HexNookIME）

1. 沿用白霜的流水线：**语料 → 分词 → 统计词频 → 评测对比**，其中**评测体系最重要**，先搭评测再调词库。
2. 底层引擎用 **librime**，不要从零写。
3. 整句准确度依靠 n-gram 语法模型（kenlm 训练 / 复用八股文、墨奇、万象模型）。
4. 大模型方向可参考 LIME，但需常驻模型服务，资源占用明显高于普通引擎，且要处理隐私问题，可作为可选增强而非默认路径。

## 后续待办

- [ ] 整理一份小狼毫 + 白霜/万象的完整配置
- [ ] 设计 HexNookIME 的词库流水线（语料选取、清洗规则、词频归一化）
- [ ] 搭建基于 rime-schema-compare 的评测基线
