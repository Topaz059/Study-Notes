# PureEdgeVLM 阶段二搭建记录

> 项目代号：PureEdgeVLM · 目标设备：骁龙 865 手机，8GB 内存、纯 CPU · 技术栈：Kotlin + C++17 + NCNN
>
> 一句话概括：把手机上跑视觉模型的地基打牢，编译 NCNN 推理库、把 ResNet50 也转成 NCNN、写出 YOLO 和场景识别两个检测器，让你在旧手机上选一张图就能看到检测框和场景名。
>
> 本文所有路径都相对项目根 `C:\Users\Blue\Desktop\work\localai`。

---

## 0. 阶段二

阶段一结束时，四个模型文件都下好了、在电脑上分别验证能跑。但能在电脑上跑和能在手机上跑之间，还隔着一整条工程链：

- 手机是 ARM CPU，NCNN 推理库得专门给它编译一份，不能直接用电脑上的；
- ResNet50 在阶段一只下了 PyTorch 权重，还没变成手机能用的 NCNN 格式，scene 目录是空的；
- 模型只是躺在那，没人把它接进 App：图片怎么变成矩阵、检测框怎么画出来、界面怎么调 C++，全得从零写。

阶段二就是干这三件事：编译 NCNN、转好 ResNet50、写出 YOLO 和场景识别这条可运行的链路，让前两个视觉模型在真机上第一次跑起来。OCR 和大模型留到阶段三、四，本阶段先不管。

---

## 1. 这个阶段做了什么

阶段二只碰两个视觉模型，其余严格不动：

- 只做 YOLO（物体检测）+ ResNet50 场景识别
- 不做 PP-OCRv5（阶段三）
- 不做 MiniCPM5 大模型（阶段三、四）
- 不做三个模型串联（阶段四）

最终产出的代码骨架如下，这套图片转矩阵、检测器、识别器、JNI 桥后面阶段三、四接着复用：

| 文件 | 干什么 |
| --- | --- |
| `app/src/main/cpp/image_util.h` / `image_util.cpp` | 把安卓 Bitmap 锁出像素指针，零拷贝、速度快 |
| `app/src/main/cpp/yolo_detector.h` / `yolo_detector.cpp` | YOLO 检测器：加载模型、letterbox、解码、NMS、坐标反算 |
| `app/src/main/cpp/scene_classifier.h` / `scene_classifier.cpp` | 场景识别器：加载模型、归一化、softmax、取 Top5 |
| `app/src/main/cpp/native_bridge.cpp` | C++ 与 Kotlin 的桥：加载模型 + 两个对外接口 |
| `app/src/main/cpp/CMakeLists.txt` | 把 NCNN 静态库 + OpenMP 链进 App |
| `app/src/main/java/com/topaz/pureedgevlm/NativeBridge.kt` | Kotlin 侧的桥：声明 external 函数、类别名、读场景标签 |
| `app/src/main/java/com/topaz/pureedgevlm/YoloBox.kt` / `SceneResult.kt` | 检测结果的数据类 |
| `app/src/main/java/com/topaz/pureedgevlm/MainActivity.kt` | 临时界面：点按钮选图，显示检测框 + 场景名 |

这套临时界面只为验证阶段二。后面阶段五会做正式的相机页和聊天页，到时这套会被替换或整合。

---

## 2. 编译 NCNN 安卓静态库

**做什么**：把 NCNN 源码编译成一份专门给骁龙 865（arm64）用的零件包 `libncnn.a`，C++ 代码要链接它才能跑模型。

**怎么做**（阶段一已装好 NDK r26c）：

```bash
cd /c/Users/Blue/Desktop/work/localai
git clone https://github.com/Tencent/ncnn.git
cd ncnn
mkdir build-android-arm64-v8a && cd build-android-arm64-v8a
cmake -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE="你的NDK路径/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a \
  -DANDROID_PLATFORM=android-28 \
  -DNCNN_VULKAN=OFF \
  -DNCNN_BUILD_EXAMPLES=OFF \
  -DNCNN_BUILD_TESTS=OFF \
  -DNCNN_BUILD_TOOLS=OFF \
  ..
cmake --build . -j$(nproc)
cmake --install . --prefix /c/Users/Blue/Desktop/work/localai/ncnn/ncnn-android-install
```

关键两处：`NCNN_VULKAN=OFF`（纯 CPU，不依赖显卡）；`NCNN_BUILD_TOOLS=OFF`（手机端不需要转换工具，编得快）。

**怎么算成功**：最后出现 `[100%] Built target ncnn`，且 `ncnn/ncnn-android-install/lib/libncnn.a` 存在。这一步全量编译 20~40 分钟，期间不要中断。

---

## 3. 把 ResNet50 转成 NCNN

**做什么**：阶段一留下的空 `scene` 目录，这一步填上 ResNet50 的 NCNN 文件。转换需要两个工具 `onnx2ncnn`（ONNX 转 NCNN）和 `ncnnoptimize`（压缩），它们是给电脑（主机）跑的一次性工具，得在电脑或 WSL 里再编一份 NCNN 主机工具版（`NCNN_BUILD_TOOLS=ON`，不指定安卓）。

**流程**（两端配合：Windows 出 ONNX，WSL 转 NCNN）：

1. WSL 里装依赖并编译主机工具版 NCNN，装完工具在 `~/ncnn-host-install/bin/`。
2. Windows 用 `py -3.10` 跑导出脚本 `export_resnet50_places365_to_onnx.py`：建一个 365 类的 ResNet50 空壳，加载 MIT 官方 `.pth.tar` 权重（注意 `state_dict` 里键名带 `module.` 前缀要去掉），导出 ONNX。
3. `py -3.10 -m onnxsim` 简化，再切回 WSL 用 `onnx2ncnn` 转 NCNN。
4. 把成品拷进 `app/src/main/assets/models/scene/`：`resnet50_fp32.param` + `resnet50_fp32.bin` + `categories_places365.txt`（365 类标签，场景识别器靠它把编号翻成中文名）。

**一个重要的真实修正**：方案里原本计划压成 fp16（即 `resnet50_places365_opt`），但实际最后留在 assets 里的是 fp32 版（`resnet50_fp32.param/.bin`）。原因是中途一度怀疑 fp16 把模型压成 NaN 导致场景识别崩，后来用 md5 校验发现 `resnet50_fp32.bin`、`resnet50_places365.bin`、`resnet50_places365_opt.bin` 三份字节完全一样，模型从头到尾是同一份完好的 fp32，所谓 fp16 转坏纯属误判。fp32 在手机上也能跑、精度更稳，于是保留了 fp32，少压一步。

---

## 4. 把 NCNN 接进工程 + 写图片工具

**接 NCNN**：把第 2 步编好的 `libncnn.a` 和头文件放进项目：

```
app/src/main/cpp/third_party/ncnn/lib/libncnn.a
app/src/main/cpp/third_party/ncnn/include/ncnn/*.h
```

`CMakeLists.txt` 让 App 参与编译并链接它。阶段二只需链接 NCNN 和 OpenMP，OpenMP 是 NDK 自带的运行库，NCNN 靠它开多线程，不链接会报 `libomp.so` 找不到。

```cmake
add_library(${CMAKE_PROJECT_NAME} SHARED
        native_bridge.cpp
        image_util.cpp
        yolo_detector.cpp
        scene_classifier.cpp)

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ${NCNN_DIR}/include)
target_link_libraries(${CMAKE_PROJECT_NAME}
        ${NCNN_DIR}/lib/libncnn.a
        omp android log)
```

说明：这份 CMakeLists 后来阶段三、四又陆续加进了 OpenCV 和 llama.cpp，但阶段二的核心就是上面这几行。

**图片工具** `image_util.cpp`：手机图片是 Bitmap，NCNN 要 `ncnn::Mat`。这里只做一件最基础的事，`lockBitmap` 用 `AndroidBitmap_lockPixels` 把像素锁出来（零拷贝，速度快），`unlockBitmap` 用完立刻解锁。YOLO 和场景识别都先调它拿像素，避免每个检测器重复写一遍锁图逻辑。

---

## 5. 写 YOLO 检测器 + 场景识别器

检测器代码已写入项目，两个类的接口很干净：

```cpp
// yolo_detector.h
std::vector<YoloBox> detect(JNIEnv*, jobject bitmap, float conf, float nms,
                            float* max_score_out = nullptr, int* max_label_out = nullptr);

// scene_classifier.h
std::vector<SceneTop> classify(JNIEnv*, jobject bitmap, int topk = 5);
```

**YOLO 检测器要点**：

- 加载时 `net.opt.use_vulkan_compute = false`（纯 CPU）、`num_threads = 4`。
- 图片处理：按最长边缩到 640，短边补灰边（letterbox 填充，灰边值 114），得到 640×640；RGBA 转 RGB；归一化到 0~1（mean 全 0、norm 全 1/255）。
- 解码：YOLO 输出通常是 `[8400, 84]`，84 是 4 个框坐标加 80 类分数。不同 NCNN 导出对维度排布不同，代码里自动判断 8400 和 84 各在哪个维度，兼容所有布局，不用手改。
- 先按 0.45 的置信度阈值过滤，再对每个类别做 NMS（重叠太多的框只留分数最高的）。
- 坐标反算：检测是在 640 缩放图上做的，最后除以缩放比、夹回原图范围，把框映射回原图像素坐标，Kotlin 直接画。

**场景识别器要点**：

- 直接缩到 224×224（场景识别不补灰边）。
- 归一化是这里最容易写错的地方（详见第 8 节坑四），正确值是 `mean=[123.675, 116.28, 103.53]`、`norm=[0.017129, 0.017507, 0.017425]`。
- 输出 365 个分数，先 softmax 成 0~1 概率，取前 5 个把编号交给 Kotlin，Kotlin 用 `categories_places365.txt` 翻成场景名。

---

## 6. 写 JNI 桥 + Kotlin 临时界面

**C++ 桥** `native_bridge.cpp` 里有三个对外函数（函数名里的 `com_topaz_pureedgevlm` 必须和包名一致）：

```cpp
Java_com_topaz_pureedgevlm_NativeBridge_nativeInit(JNIEnv*, jclass, jobject assetManager);
Java_com_topaz_pureedgevlm_NativeBridge_yoloDetect(JNIEnv*, jclass, jobject bitmap, jfloat conf, jfloat nms);
Java_com_topaz_pureedgevlm_NativeBridge_sceneRecognize(JNIEnv*, jclass, jobject bitmap);
```

- `nativeInit`：拿到 AssetManager，从 `assets/models/yolo/model.ncnn.*` 和 `assets/models/scene/resnet50_fp32.*` 把模型读进内存、加载好，并记一条初始化日志（`g_init_debug`）。
- `yoloDetect` 内部调 `YoloDetector::detect`，把结果拼成 `YoloBox` 对象数组回传，还会记一条最近一次检测的日志。
- `sceneRecognize` 内部调 `SceneClassifier::classify`，拼成 `SceneResult` 对象数组回传。
- 还有一个 `getDebug()` 把上面两段日志吐给 Kotlin，界面上能直接看到，方便排查问题。

**Kotlin 侧**：

- `NativeBridge.kt` 是个单例 `object`，`init { System.loadLibrary("pureedgevlm") }` 加载库；声明 `external fun yoloDetect / sceneRecognize / nativeInit / getDebug`；内置 COCO 80 类名称数组（`cocoLabels`）；`init(context)` 里调 `nativeInit` 并把 `categories_places365.txt` 读成 `sceneLabels` 列表。
- `YoloBox.kt` / `SceneResult.kt` 是两个独立的数据类，Kotlin 和 C++ 两边字段一一对应。
- `MainActivity.kt` 是临时界面：一个选择图片按钮，后台线程里跑 `yoloDetect`、把框画到图上、`sceneRecognize`，在文字区列出场景 Top5 名字。后台线程是为了不让界面卡住。

---

## 7. 真机验收

装到手机，点按钮选一张图，应该看到：

- 图上出现 YOLO 检测框 + 类别标签（如 bus、person）；
- 文字区出现场景 Top5 名字；
- 整个过程 1 秒内完成，界面不卡。

实测结果：选阶段一用的 `bus.jpg`，YOLO 正常框出 bus、person 等；场景识别在修正归一化后 Top5 分散且合理，与电脑端 onnxruntime 标准答案一致。延迟方面 YOLO 单张约百毫秒，ResNet50 更快。

**一个必须讲清的情况**：换一张猫图，YOLO 给的是 `cat=0.239 / dog=0.243`，猫分和狗分几乎平手，模型靠微弱优势把票投给了 dog，于是猫被标成 dog。代码经核对没有问题（猫=15、狗=16 映射正确，红框准确框在猫身上，预处理和输出形状都对），问题出在 yolo11n（nano）这个最小模型本身细分类能力有限。根治办法是换更大的 yolo11s，输出结构相同，只需替换权重重新编译，解码代码不用改，那属于项目做完后的锦上添花，不阻塞阶段二验收。

---

## 8. 踩过的坑

阶段二真机调试踩到的坑，按发生顺序记录，供后续参考。

1. **选图后永远显示检测到 0 个物体，但不崩溃（最隐蔽的坑）**。编译通过、能选图，但无论选什么图都显示 0 个物体，不报错也不闪退。根因是加载成功和失败的判断写反了。NCNN 里 `load_param_mem`（读结构）返回整数、0 表示成功；而 `load_model`（读权重）返回的是消耗的字节数，0 表示失败、非 0 才是成功。代码曾把 `load_model(...) == 0` 当成功，正好反了，模型实际返回了约一千万字节（权重真读到了），却被当成失败，检测函数一进来发现模型没加载直接返回空数组。正确写法：成功等于 `load_param_mem(...) == 0`，且成功等于 `load_model(...) != 0`。教训：写加载判断前先查所用库的头文件确认返回值语义，不要统一按 0 表示成功处理。

2. **编 NCNN 主机工具后，安装目录里独缺 `onnx2ncnn`**。按 `cmake -DNCNN_BUILD_TOOLS=ON` 编译并安装后，`caffe2ncnn`、`ncnnoptimize` 等都在，就是没有转现代模型必需的 `onnx2ncnn`，且无任何报错。根因是拉的 NCNN 每日构建版把 `onnx2ncnn` 从默认构建列表里移除了。修法：在 clone 下来的 `tools/CMakeLists.txt` 末尾手动加一行 `add_subdirectory(onnx)`，重配重编重装即可。教训：编完主机工具一定先 `ls` 安装目录确认 `onnx2ncnn` 在不在。

3. **WSL 里粘贴命令变乱码，以及 cmd 跑还是 WSL 跑的切换摩擦**。阶段二被拆成两端，Windows（导出 ONNX、onnxsim，在 cmd 或 Git Bash，用 `py -3.10`）和 WSL（转 NCNN、压 fp16，在 Ubuntu 终端，用 `~/ncnn-host-install/bin/...`）。WSL 里别用 `Ctrl+V` 粘贴（会变乱码），要用鼠标右键或 `Shift+Insert`；Windows 的 C 盘在 WSL 里是 `/mnt/c/`；WSL 里敲 `py -3.10` 会报命令找不到（那是 Windows 的 Python）。动手前先想清楚当前这条命令属于哪一端。

4. **场景识别永远输出同一个类（真因，曾误判成 fp16）**。App 装上后场景模型能加载、输出维度 365 也对，但无论选什么图都输出同一个编号（如 idx=306 即天空）且概率 =1.0000，停车场也显示天空。根因是 `scene_classifier.cpp` 里 `substract_mean_normalize(mean, norm)` 的参数写反了。查 NCNN 源码确认：这个函数做的是 `out = (input − mean) × norm`（乘法），而 ImageNet 标准是 `(pixel/255 − mean) / std`。旧代码填的是 `mean=[0.485, 0.456, 0.406]`、`norm=[0.229, 0.224, 0.225]`，且输入是 0~255，没有先除 255，于是实际在算 `(pixel − 0.485) × 0.229`，数值范围严重越界，网络输出退化成某一维等于 1.0。修法：令 `mean = mean×255`、`norm = 1/(255×std)`，即 `mean=[123.675, 116.28, 103.53]`、`norm=[0.017129, 0.017507, 0.017425]`。改后 Top5 分散，与电脑端标准答案一致。教训：第一，文件重命名或复制不等于内容改了，动模型前先用 md5 校验；第二，不要迷信 fp16 转成 NaN 的第一直觉，先查源码确认算子语义；第三，调试时打印输入图的 mean/std，正确应约等于 0/1，是判断预处理对错的黄金指标。

---

## 9. 写在最后

到这，阶段二全部完成：NCNN 编译进 App 了、ResNet50 转成 NCNN 填进了 scene 目录、YOLO 和场景识别两个检测器在骁龙 865 上真机跑通、选一张图就能看到检测框和场景名。

这一阶段最花时间的，一半是编译等待（NCNN 全量编 20~40 分钟），一半是加载判断和归一化这两处写成反了的坑，它们都不崩溃、表面看不出毛病，容易被误判成模型没认出物体。做完回头看，核心逻辑并不复杂，难点在于每一处加载和成功判断都要查头文件确认，不能凭直觉。

接下来是阶段三：把 PP-OCRv5 文字识别接进来，并把 llama.cpp 的大模型运行库编进工程，让手机上能跑通 OCR 和 MiniCPM5 生成文字。

项目地址：https://github.com/Topaz059/PureEdgeVLM

*本文写于 2026-07-21，记录 PureEdgeVLM 端侧多模态系统阶段二的搭建过程。*
