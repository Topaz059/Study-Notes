# PureEdgeVLM 阶段三搭建记录

> 项目代号：PureEdgeVLM · 目标设备：骁龙 865 手机，8GB 内存、纯 CPU · 技术栈：Kotlin + C++17 + NCNN + OpenCV-mobile + llama.cpp
>
> 一句话概括：在阶段二已跑通 YOLO 检测与 ResNet50 场景识别的基础上，接进百度 PP-OCRv5 文字识别，让手机选一张带字的图就能把图里的字读出来。
>
> 本文路径均相对项目根 `C:\Users\Blue\Desktop\work\localai`。

---

## 0. 阶段三

阶段二结束时，手机已经能选一张图看到 YOLO 检测框和场景 Top5。但看懂一张图还差最后一块，就是图里写了什么字。阶段三就是补上这一块，把 PP-OCRv5 文字识别接进 App，让手机选一张带字的图就能把字读出来。

阶段三只加 OCR 这一块：模型用现成的 NCNN 版，代码从第三方库移植，主要工作是下载一个精简版 OpenCV 做检测后处理，再把这套能力接到已有的图片管道和界面上。MiniCPM5 大模型留到阶段四。

---

## 1. 这个阶段做了什么

阶段三只加 OCR，复用阶段二写好的图片转矩阵、JNI 桥、Kotlin 调度，不碰 YOLO 和场景识别。新增与改动的文件如下：

| 文件 | 内容 |
| --- | --- |
| `app/src/main/cpp/ocr/ppocrv5.h` | OCR 引擎头文件，声明检测、识别、加载等接口 |
| `app/src/main/cpp/ocr/ppocrv5.cpp` | OCR 引擎主体，检测与识别逻辑，移植自第三方库，删掉了原版在图上画框的 draw |
| `app/src/main/cpp/ocr/ppocrv5_dict.h` | 约 18400 字的字表，识别出的编号查这个表变文字 |
| `app/src/main/cpp/opencv_omp_shim.cpp` | 一个空实现函数，修 OpenMP 版本错配，见第 7 节坑五 |
| `app/src/main/cpp/third_party/opencv-mobile-4.13.0-android/` | 精简版 OpenCV，只含 core 与 imgproc，供检测后处理 |
| `app/src/main/assets/models/ocr/` | 4 个 OCR 模型文件，det 与 rec 各一份 param 和 bin，fp16 版 |
| `app/src/main/cpp/native_bridge.cpp` | 加 OCR 引擎全局变量，nativeInit 里加载 OCR，新增 ocrRecognize 接口 |
| `app/src/main/cpp/CMakeLists.txt` | 加 OpenCV 查找与链接，把 ocr/ppocrv5.cpp 和 opencv_omp_shim.cpp 加进编译 |
| `app/src/main/java/com/topaz/pureedgevlm/NativeBridge.kt` | 加 ocrRecognize 桥接口声明 |
| `app/src/main/java/com/topaz/pureedgevlm/MainActivity.kt` | 选图后统一转 ARGB_8888，调用 ocrRecognize，结果拼到界面下方 |

对应提交 `b8badbe`：阶段三 PP-OCRv5 OCR 集成，含 OpenMP deinit 空实现修复，2026-07-20。

---

## 2. 准备 OpenCV-mobile

**做什么**：下载精简版 OpenCV 放到工程。OCR 文字检测算完后，要把歪着的文字框摆正、裁剪出来才能识别，这一步的几何变换靠 OpenCV 完成，具体是找轮廓、旋转矩形、透视变换。用 nihui 编译的 opencv-mobile，比官方 OpenCV 小很多，只含 core 与 imgproc 两个模块。

**怎么做**：

1. 从发布页 `https://github.com/nihui/opencv-mobile/releases` 下载。版本要注意：顶部最新的 v36 是 opencv 5.0.0，不要下；下标签为 v35 的（opencv 4.13.0），资产里点 `opencv-mobile-4.13.0-android.zip`。
2. 解压，把文件夹重命名成 exactly `opencv-mobile-4.13.0-android`，整体移动到 `app/src/main/cpp/third_party/`。最终路径应存在：
   `app/src/main/cpp/third_party/opencv-mobile-4.13.0-android/sdk/native/jni/OpenCVConfig.cmake`

**为什么单独要 OpenCV**：YOLO 与 ResNet50 的推理全靠 NCNN，但 PP-OCRv5 的检测后处理是几何操作，把斜的字摆正、裁剪出来都要靠它，用 OpenCV 最方便。只装 core 与 imgproc，App 体积增加不多。

---

## 3. 确认 OCR 模型文件就位

**做什么**：确认 4 个 OCR 模型文件已在 assets。它们是从 `equationl/ncnn-android-ppocrv5` 拿的现成 NCNN 格式，这个仓库移植自 nihui 原版，阶段一就放进来了，本阶段默认已在：

```
app/src/main/assets/models/ocr/
  PP_OCRv5_mobile_det.ncnn.param
  PP_OCRv5_mobile_det.ncnn.bin
  PP_OCRv5_mobile_rec.ncnn.param
  PP_OCRv5_mobile_rec.ncnn.bin
```

这 4 个文件是 fp16 版，和代码里 `use_fp16 = true` 对应。字表不单独放文件，内嵌在 `ocr/ppocrv5_dict.h` 里，约 18400 字。

---

## 4. 写 OCR 引擎

OCR 引擎代码直接放进工程，三件套：

- `ocr/ppocrv5.h`：声明检测、识别、加载等接口，对外暴露 `detect`、`recognize`、`detect_and_recognize` 三个方法。
- `ocr/ppocrv5.cpp`：检测用 DB 算法找出文字区域，识别用 CRNN 加 CTC 把裁出来的字区域变成文字。移植自第三方库，删掉了原版在图上画框写字的 draw，因为框显示交给 Kotlin 界面统一处理。
- `ocr/ppocrv5_dict.h`：约 18400 字的字表，识别模型输出的是字符编号，查这个表才变成真字。

**关键设计点**：

- 模型加载用 OCR 库自带的、从 `AAssetManager` 直接读的 load 版本，路径相对 assets 根目录，例如 `models/ocr/PP_OCRv5_mobile_det.ncnn.param`。
- `use_fp16 = true`：读 fp16 权重，更快、更省内存，精度基本无损。
- 识别结果排序：先按文字框的 y 坐标分行，同一行里按 x 坐标从左到右排，每行文字用换行拼起来返回。横排中文很准，竖排或多栏排版可能顺序乱，属后续优化项，不阻塞本阶段。

---

## 5. 接进工程

**CMakeLists.txt**：加 OpenCV 查找，把 OCR 源文件和 OpenMP 补丁文件加进编译列表，链接时加 `${OpenCV_LIBS}`：

```cmake
set(OpenCV_DIR ${CMAKE_CURRENT_SOURCE_DIR}/third_party/opencv-mobile-4.13.0-android/sdk/native/jni)
find_package(OpenCV REQUIRED core imgproc)

add_library(${CMAKE_PROJECT_NAME} SHARED
        native_bridge.cpp
        ...
        ocr/ppocrv5.cpp
        opencv_omp_shim.cpp)

target_link_libraries(${CMAKE_PROJECT_NAME}
        ...
        ${OpenCV_LIBS}
        ...)
```

说明：这份 CMakeLists 阶段二只链接 NCNN 与 OpenMP，阶段三加了 OpenCV，阶段四又加 llama.cpp。OpenCV 必须 `find_package(OpenCV REQUIRED core imgproc)` 且链接 `${OpenCV_LIBS}`，否则报 `undefined reference to cv::findContours` 之类。

**C++ 桥** `native_bridge.cpp`：

- 顶部加 `#include "ocr/ppocrv5.h"` 与 `#include "ocr/ppocrv5_dict.h"`，声明全局 `static PPOCRv5 g_ocr;`。
- `nativeInit` 末尾加一段：用 `AAssetManager` 把 det 与 rec 四个文件读进来，调 `g_ocr.load(...)`，打一条 `ocr init: load=0 loaded=1` 的初始化日志。
- 新增对外函数 `ocrRecognize`：接收 Bitmap，检测加识别，按阅读顺序排序，查字表拼成文字，返回字符串。模型没加载时返回"（OCR 模型未加载）"，没识别到字返回"（未识别到文字）"。

**Kotlin 侧**：

- `NativeBridge.kt` 加一行 `external fun ocrRecognize(bitmap: Bitmap): String?`。
- `MainActivity.kt`：选图后统一把 Bitmap 转成 `ARGB_8888`，OCR 读像素要这个格式，原因见第 7 节坑二，调 `ocrRecognize`，返回文字拼到界面下方的 OCR 文字区域，和已有的检测框、场景 Top5 并列显示。

---

## 6. 编译与真机验收

Android Studio 里点 Run，第一次编译要链接 NCNN 静态库与 OpenCV，可能 3 到 8 分钟。装到手机后点选择图片，选一张字多、清晰、正向拍的图，餐厅菜单最容易一次成功，等几秒看界面下方 OCR 文字后面有没有出现一行行文字。

实测结果：菜单图几百毫秒内输出全部文本，界面下方出现能看懂的文字，和图里写的差不多。Logcat 过滤 `OCR` 能看到 `ocr init: load=0 loaded=1`，说明模型加载成功。YOLO 框与场景 Top5 仍正常，OCR 是叠加，不影响前两个。

---

## 7. 踩过的坑

阶段三真机调试遇到的问题，按发生顺序记录。

1. **OpenCV 文件夹名字不对，编译找不到**。Android Studio 编译报 `Could not find a package configuration file with name "OpenCVConfig.cmake"`。根因是 `CMakeLists.txt` 写死按 `third_party/opencv-mobile-4.13.0-android/sdk/native/jni` 找 OpenCV，但解压出来的文件夹可能叫 `OpenCV-android-sdk` 或带别的版本后缀，名字对不上就找不到。修法：把解压出的文件夹重命名成 exactly `opencv-mobile-4.13.0-android`。教训：路径写死、名字必须匹配，下载解压后第一件事就是核对最终路径和 CMakeLists 里写的是否一致。另外发布页顶部 v36（opencv 5.0.0）别下，下 v35（opencv 4.13.0）即可。

2. **OCR 读图前 Bitmap 格式不对**。Android 的 Bitmap 内存序是 RGBA，OCR 接口要的是 RGB，而且不是所有 Bitmap 都是 ARGB_8888，有的可能是 RGB_565，格式不对 lockBitmap 读出来的像素会错位。修法：在 `MainActivity.kt` 的 `runPipeline` 里，调用 OCR 前统一把图转成 ARGB_8888，再把这张转好的图同时传给 YOLO、场景、OCR。凡是 C++ 要读 Bitmap 像素，统一转 ARGB_8888 最稳。

3. **字表大小和模型输出类数对不上会越界**。PP-OCRv5 识别模型输出 18385 类，对应字表里 18385 个常用字符，其中一个是空白类占位。代码里用 `if (id < 0 || id >= character_dict_size)` 做越界保护，越界的编号会被跳过，不会让程序崩溃。修法：别手改 `ppocrv5_dict.h` 的字表内容或行数，它必须和模型配套。识别结果若整段乱码，先查字表大小是否约 18401 行，以及模型是不是 mobile 版，server 版类数不同。字表是模型配套文件，换模型必须连字表一起换。

4. **OCR 代码 `#include <net.h>` 裸名，编译报 net.h file not found**。编译 `ppocrv5.cpp` 时卡在 `fatal error: 'net.h' file not found`。根因：从第三方库搬来的代码引用 NCNN 头文件用裸名 `#include <net.h>` 与 `#include "cpu.h"`，但本项目把 NCNN 头文件搜索路径只设到 `third_party/ncnn/include`，而 `net.h` 真实位置是 `third_party/ncnn/include/ncnn/net.h`，比搜索路径多一层 `ncnn/`。项目里 YOLO 与场景的代码早就写成 `#include <ncnn/net.h>`，带 `ncnn/` 前缀，OCR 这段是后来加的、没对齐。修法：把 OCR 代码的 3 处裸名包含改成带 `ncnn/` 前缀，具体是 `net.h` 改 `ncnn/net.h`、`cpu.h` 改 `ncnn/cpu.h`。从别的 NCNN 工程搬 C++ 代码，先看它 include 头文件是裸名还是带 `ncnn/` 前缀。

5. **链接报错 undefined symbol __kmpc_dispatch_deinit（OpenMP 版本错配）**。编译期通过了，到链接 `libpureedgevlm.so` 时失败：`ld.lld: error: undefined symbol: __kmpc_dispatch_deinit`。根因：opencv-mobile v35（opencv 4.13.0）是用比本项目 NDK r28 更新的 LLVM 与 OpenMP 编出来的，新版本把 OpenMP 调度收尾函数从老名字 `__kmpc_dispatch_fini_4u` 改名叫 `__kmpc_dispatch_deinit`，而 NDK r28 自带的 libomp 只有老名字，没有 deinit，于是链接报未定义。修法：新建 `opencv_omp_shim.cpp`，自己提供一个空实现的 `__kmpc_dispatch_deinit`，函数体直接返回、什么都不做，加进 CMakeLists 的 add_library。空实现安全的原因：循环的实际工作已被 init 与 next 跑完，deinit 只是 runtime 收尾清理，空实现不影响结果正确性，代价是每次并行循环结尾有一点 runtime 缓冲没释放，OCR 调用不频繁，可忽略；NCNN 走 fork_call，不调用 deinit，不受影响。第一版补丁曾把 deinit 转发给 NDK 的 `__kmpc_dispatch_fini_4u`，以为同功能直转就行，结果真机一跑就闪退（SIGSEGV、空指针、fault addr 0x8c，崩溃栈落在 libomp.so 的 fini 函数）。根因是 init 与 next 用老布局写、deinit 用新布局读，转发后读错位空指针。改成空实现后崩溃消失，OCR 正常。预编译第三方库和本机 NDK 的 OpenMP 运行库可能版本错配，表现就是链接期冒出一个没见过的 kmpc 符号。先看是少了一个符号还是一堆符号都缺，少一个往往是改名或版本差，用 shim 桥接即可。这种改名类错配不能用老函数简单转发，空实现才安全。

---

## 8. 写在最后

到这，阶段三全部完成：精简版 OpenCV 放进工程，PP-OCRv5 的 det 与 rec 模型接进 App，OCR 引擎移植好并接到界面，手机选一张带字的图就能在下方看到读出来的文字。

本阶段主要工作量不在写业务代码，OCR 引擎是从第三方库移植的，难点在两处环境对齐：OpenCV 文件夹名字必须和 CMakeLists 写死的一致，opencv-mobile 与 NDK 的 OpenMP 版本错配用空实现 shim 绕过。这两处都不在业务逻辑里，却卡在编译、链接和真机启动上。

接下来是阶段四：把 llama.cpp 的大模型运行库编进工程，在 App 里用 MiniCPM5 做纯文字多轮对话，并做绑核调度优化性能。

项目地址：https://github.com/Topaz059/PureEdgeVLM

*本文写于 2026-07-24，记录 PureEdgeVLM 阶段三的搭建过程。*
