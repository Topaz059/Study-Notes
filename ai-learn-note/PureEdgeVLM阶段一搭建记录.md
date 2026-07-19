# PureEdgeVLM 阶段一搭建记录

> 项目代号：**PureEdgeVLM** · 目标设备：骁龙865手机，8GB 内存、纯 CPU · 技术栈：Kotlin + C++17 + NCNN + llama.cpp
>
> 一句话概括：在旧手机上纯本地跑通"拍照 → 视觉理解 → 大模型回答"的多模态系统，零网络依赖。



---

## 0. 为什么要做这个

端侧 AI 我自己很看好，模型跑在设备本地、不依赖云端，这件事本身就很有意思，也很有前景。我一直想亲手把一个多模态系统真正部署到手机上跑通，而不是只停留在调用接口、运行现成演示的层面。

目标很明确：在一台旧的骁龙 865 手机上，不联网、纯靠手机自己的 CPU，跑一个"看到照片 → 理解内容 → 用大模型回答"的多模态 App。

定了几条硬约束，后面所有技术选择都围着它们转：

- **纯 CPU，不用 DSP / GPU**：旧机不怕损耗，开发简单，跨平台兼容。
- **零网络依赖**：模型全打包进安装包，断网也能跑。
- **不做算法**：用现成预训练权重。
- **框架统一**：视觉模型全用 NCNN，大模型用 llama.cpp，只用两个框架，避免编译灾难。

阶段一的目标很明确：把环境装好、把 4 个模型准备好、在电脑上验证它们都能用。

---

## 1. 技术选型

| 模型                 | 干什么                   | 权重来源            | 量化           | 框架        |
| ------------------ | --------------------- | --------------- | ------------ | --------- |
| YOLOv11n           | 物体检测，认人、车、瓶子等 80 类    | Ultralytics 官方  | FP16         | NCNN      |
| ResNet50 Places365 | 场景识别，认机场、卧室、花园等 365 类 | MIT 官方          | FP16         | NCNN      |
| PP-OCRv5           | 中英文文字识别               | 百度开源 NCNN 版     | FP16         | NCNN      |
| MiniCPM5-1B        | 本地大模型，生成自然语言回答        | OpenBMB 官方 GGUF | INT4（Q4_K_M） | llama.cpp |

两个关键决定：

- **三个视觉模型统一用 NCNN**：腾讯开源的推理框架，对 ARM CPU 优化好、库体积小，是端侧视觉推理的主流选择。
- **大模型用 llama.cpp**：GGUF 格式生态成熟，跨平台，MiniCPM5 官方就提供 GGUF 版。

量化上，视觉模型走 FP16，只有大模型压到 INT4，0.5GB 就能装下 1.08B 参数的模型，是端侧跑得动的关键。

---

## 2. 环境搭建

先搭建开发环境，准备好以下工具：

- **Android Studio**：写安卓 App 的主软件，装 Standard 版，主题随便。
- **NDK r26c + CMake 3.22.1**：让 App 能用 C++ 跑模型推理，版本固定别用太新的。
- **Python 3.10**：处理模型用。本机有两个 Python，干活这个叫 `py -3.10`，环境里装好了 torch、ultralytics、onnx 等库。
- **adb**：电脑和手机通信的工具，用于装 App、看日志。
- **手机开开发者选项 + USB 调试**：连上电脑能被识别。

环境搭好后，用 Android Studio 建一个 Native C++ 模板项目，点 Run 装到手机上，看到 "Hello from C++" 就说明整条"电脑编译 → 手机运行"的链路通了。

---

## 3. 模型准备

四个模型分开准备，每个做完先验证再做下一个。

### 3.1 YOLOv11n（物体检测）

流程最简单的一个，一行命令导出 NCNN 格式：

```bash
yolo export model=yolo11n.pt format=ncnn imgsz=640 simplify=True
```

跑完得到 `model.ncnn.param`（结构）和 `model.ncnn.bin`（权重），约 3.2MB。

### 3.2 ResNet50 Places365（场景识别）

下载 MIT 官方的 Places365 权重，用 `predict_places365.py` 在电脑上验证能输出 top5 场景就算过。NCNN 格式转换留给阶段二，因为要先编译好 NCNN 再转。

> 这个模型来源中途换过，详见第 6 节踩坑。最终定的是 MIT 官方 ResNet50，不是 MobileNet。

### 3.3 PP-OCRv5（文字识别）

百度新一代 OCR，直接拿现成的安卓 NCNN 移植版：

```bash
git clone https://github.com/equationl/ncnn-android-ppocrv5
```

仓库里 `app/src/main/assets/` 自带 det（检测）和 rec（识别）两套 NCNN 模型，字符表内嵌在代码里，不用单独下词典。

### 3.4 MiniCPM5-1B（本地大模型）

从 ModelScope 下 GGUF 格式，务必是 Q4_K_M 这个 4 比特版本，约 500MB：

```bash
modelscope download --model OpenBMB/MiniCPM5-1B-GGUF minicpm5-1b-Q4_K_M.gguf --local_dir ./models/minicpm5
```

别下 Q2_K，会乱码；也别下 Q8，体积过大手机无法运行。

---

## 4. 模型放进项目

把四个模型拷进安卓项目的 `app/src/main/assets/models/`，分四个子目录：

```
assets/models/
├─ yolo/    # YOLO 的 param + bin
├─ scene/   # ResNet50 的 NCNN，阶段二转换后放
├─ ocr/     # PP-OCRv5 的 det + rec
└─ llm/     # MiniCPM5 的 gguf
```

安卓会从这里读模型。总大小控制在 700MB 以内。

---

## 5. 电脑端验证

四个模型在手机上跑是后面的事，阶段一先在电脑确认它们没坏、能正常推理。写好验证脚本，核对输出。

| 验证       | 脚本                     | 怎么算过                             |
| -------- | ---------------------- | -------------------------------- |
| YOLO     | `test_yolo.py`         | 能检出 bus、person 等物体，打印"验证通过"      |
| ResNet50 | `predict_places365.py` | 输出 top5 场景名，不是报错        |
| PP-OCRv5 | 模型文件到位即过               | NCNN 版电脑跑要先编译 NCNN 不划算，功能验证留到手机端 |
| MiniCPM5 | `test_llm.py`          | 输出一句通顺中文，打印"验证通过"                |

最后一步最关键：MiniCPM5 用 llama-cpp-python 加载，能生成通顺中文，说明模型文件没坏、格式能被 llama.cpp 读、中文输出正常。

---

## 6. 踩过的坑

以下是阶段一实际遇到的问题，记录下来避免重复踩坑。

1. **同盘拖拽变移动**：Windows 同一块硬盘内拖文件默认是"移动"不是"复制"，源文件被直接移走，assets 涨到 677MB 但源目录空了。往 assets 放模型一律用 Ctrl+C / Ctrl+V，或拖拽时按住 Ctrl 出现"+"才是复制。
2. **两个 Python 打架**：机器上有 `py -3.10` 和系统另一个 3.13。用裸 `python` 跑脚本报缺库，统一前缀 `py -3.10` 就正常。
3. **MiniCPM5 文件名大小写 404**：教程写 `minicpm5-1b-Q4_K_M.gguf`，仓库真实文件名是 `MiniCPM5-1B-Q4_K_M.gguf`，下载直接 404。下之前核对大小写。
4. **ResNet50 权重源迭代**：原计划 clone 的仓库已下架，临时改方案 B 用 ImageNet 权重做场景识别，复查发现 ImageNet 训的是"物体"不是"场景"，和 YOLO 功能重叠；又试 MIT MobileNetV2 Places365，最终改用 MIT 官方 **ResNet50** Places365。认准官方源别将就。
5. **误装 paddlepaddle + 错误镜像**：想用 paddle 验证 OCR，镜像路径早废了装不上；更关键，PP-OCRv5 是 NCNN 版，电脑跑根本不需要百度飞桨框架。已卸载误装的包。教训：先想清楚技术路线再装依赖。
6. **Git 删文件还在历史里**：文档从仓库删除并提交，旧提交仍能翻到全文。要彻底消失得重写历史。私人文档从第一天就进 `.gitignore`。
7. **cmd 命令行直接敲 Python 代码**：在 `C:\>` 下输入 `import torch` 全报"不是内部或外部命令"。命令行窗口只认 Windows 命令，Python 代码要写进 `.py` 文件用 `py -3.10 xxx.py` 跑，或先 `py -3.10` 进交互模式。
8. **包名大小写改名失败**：默认包名 `com.Topaz...` 想改成小写，Windows 不区分大小写导致改名不生效。用两步改名法：先改成 `topaztmp` 再改成 `topaz`。
9. **INSTALL_FAILED_TEST_ONLY**：点 Run 装 App 报这个，因为调试包带了测试标记，真我 / OPPO / vivo 会拒装。在 `gradle.properties` 加 `android.injected.testOnly=false` 重装即可。
10. **项目路径含空格警告**：保存路径有空格时 NDK 工具会出问题，路径改成不含空格。

---

## 7. 一个重要的架构更正

阶段一收尾时核实 MiniCPM5 的架构，发现原方案文档写错了，需要特别记录，架构名是后续所有文档的基准，错了会一路传下去。

原文档多处写"MiniCPM5 使用 SALA 混合注意力架构，需 llama.cpp b3500+ 支持"。查 HuggingFace 官方模型卡，事实是：

- **MiniCPM5-1B 是标准 LlamaForCausalLM 架构**，无需定制内核，llama.cpp 原生支持。
- SALA 是另一个独立模型（MiniCPM-SALA），跟 MiniCPM5-1B 不是一回事。
- 因此没有"b3500+ 版本门槛"这回事，实测 llama-cpp-python 0.3.34 直接就能加载。

已把正确结论写进项目根的 `notes.md`，并把两份方案文档里的 SALA / b3500+ 全部改成标准 LlamaForCausalLM。

> 教训：写进文档的架构名，一定要去官方来源核实再落笔，别照抄方案草稿，否则被追问细节就会出错。

---

## 8. 写在最后

到这，阶段一全部完成：环境通了、四个模型齐了、电脑验证全过。接下来是阶段二，编译 NCNN 静态库、写检测器、让 YOLO 先在 865 手机上真正跑起来。

这一阶段最花时间的不是技术，而是把环境弄对、把模型来源理顺、把一路踩的坑填平。做完回头看，核心逻辑不复杂，难的是"每一步都别想当然"。

项目地址：https://github.com/Topaz059/PureEdgeVLM

*本文写于 2026-07-19，记录 PureEdgeVLM 端侧多模态系统阶段一的搭建过程。*
