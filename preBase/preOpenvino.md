# OpenVINO Runtime

本文按 `zimiao/sp_vision_25` 中的实际调用整理 OpenVINO C++ API。项目使用 OpenVINO 2024.6，头文件为 `<openvino/openvino.hpp>`，命名空间为 `ov`。

## 一、项目中的调用链

OpenVINO 推理通常分为以下步骤：

```cpp
ov::Core core;
std::shared_ptr<ov::Model> model = core.read_model(model_path);
ov::CompiledModel compiled = core.compile_model(model, device);
ov::InferRequest request = compiled.create_infer_request();
request.set_input_tensor(input_tensor);
request.infer();
ov::Tensor output = request.get_output_tensor();
```

- `ov::Core`：运行时入口，读取模型、查询/选择设备和编译模型。
- `ov::Model`：网络结构的内存表示，通常由 `read_model` 返回 `std::shared_ptr<ov::Model>`。
- `ov::CompiledModel`：针对指定设备编译后的模型，可创建推理请求。
- `ov::InferRequest`：一次推理执行上下文，负责设置输入、执行推理和读取输出。
- `ov::Tensor`：输入或输出的多维数据，包含元素类型、形状和数据指针。

## 二、`ov::Core`

### 1. `read_model`

```cpp
auto model = core.read_model(model_path);
```

读取 OpenVINO IR 模型或 OpenVINO 支持的模型格式。项目中的配置项 `yolo5_model_path`、`yolov8_model_path`、`yolo11_model_path`、`model` 和 `classify_model` 提供模型路径。

项目模型主要有两种形式：

- IR 模型：`.xml` 描述网络结构，`.bin` 保存权重；两个文件必须匹配。
- ONNX 模型：部分分类器路径使用 `.onnx`。项目同时保留了 OpenCV DNN 的加载代码，但 `ovclassify` 使用 OpenVINO 编译模型。

源码：`tasks/auto_aim/yolos/yolov5.cpp`、`yolov8.cpp`、`yolo11.cpp`、`tasks/auto_aim/classifier.cpp`、`tasks/auto_buff/yolo11_buff.cpp`。

### 2. `compile_model`

```cpp
compiled_model_ = core_.compile_model(model, device_);
compiled_model_ = core_.compile_model(
  model, "AUTO", ov::hint::performance_mode(ov::hint::PerformanceMode::LATENCY));
```

将模型编译为目标设备可执行的 `ov::CompiledModel`。项目的 `device` 从 YAML 读取，常见值为 `CPU`、`GPU` 或 `AUTO`。

项目中的性能提示：

- `ov::hint::PerformanceMode::LATENCY`：单帧低延迟，YOLO 和分类器使用。
- `ov::hint::PerformanceMode::THROUGHPUT`：提高并行吞吐，多线程检测器使用。
- Buff 检测器当前直接使用 `core.compile_model(model, "CPU")`，没有显式性能提示。

## 三、`ov::Model` 和模型输入输出

`ov::Model` 表示网络结构。项目在 Buff 检测器中通过端口查询模型信息：

```cpp
network.get_friendly_name();
network.inputs();
network.outputs();
```

`network.inputs()` 和 `network.outputs()` 返回 `std::vector<ov::Output<const ov::Node>>`。端口对象常用接口：

```cpp
input.get_names();
input.get_any_name();
input.get_element_type();
input.get_shape();
```

- `get_any_name()`：取得一个可用的端口名称。
- `get_element_type()`：取得端口元素类型，例如 `f32`。
- `get_shape()`：取得静态形状，例如 `[1, 3, 640, 640]`。

项目封装的 `printInputAndOutputsInfo` 位于 `tasks/auto_buff/yolo11_buff.cpp`，适合在模型输入输出形状不确定时调试。

## 四、`ov::CompiledModel`

```cpp
auto request = compiled_model_.create_infer_request();
auto input_port = compiled_model_.input();
```

- `create_infer_request()`：创建一个可执行的 `ov::InferRequest`。
- `input()`：取得编译模型的默认输入端口，多线程检测器用它读取输入信息。

同一个 `ov::InferRequest` 不应在多个线程中同时使用。项目的全向感知模块为每个 YOLO 实例单独创建线程和模型对象；多线程检测器则维护自己的请求队列。

## 五、`ov::InferRequest`

### 1. 设置输入

```cpp
request.set_input_tensor(input_tensor);
```

把 `ov::Tensor` 绑定到模型输入。项目的 YOLOv5/v8/v11 和分类器都采用“创建请求、设置输入、同步推理”的模式。

### 2. 执行推理

```cpp
request.infer();
```

执行同步推理，函数返回后输出张量可读取。YOLO、分类器和 Buff 检测器的单帧路径使用这种模式。

### 3. 异步执行：`start_async` 和 `wait`

多线程检测器把请求放入线程安全队列：

```cpp
infer_request.set_input_tensor(input_tensor);
infer_request.start_async();
queue_.push(std::move(infer_request));

auto request = queue_.pop();
request.wait();
auto output = request.get_output_tensor();
```

- `start_async()`：提交异步推理后立即返回。
- `wait()`：等待该请求完成。

对应源码为 `tasks/auto_aim/multithread/mt_detector.cpp` 的 `push`、`pop` 和 `debug_pop`。项目通过队列保存 `ov::InferRequest`，在消费线程中显式等待，没有使用回调函数。

### 4. 读取输出

```cpp
auto output = request.get_output_tensor();
auto shape = output.get_shape();
const float * data = output.data<const float>();
```

YOLO 输出通常被包装为 `cv::Mat`，再由 OpenCV 完成转置、置信度筛选、关键点解析和 NMS。OpenVINO 只负责模型前向计算，不负责项目自定义的后处理。

## 六、`ov::Tensor`、`ov::Shape` 和元素类型

### 1. 构造输入张量

```cpp
ov::Tensor input_tensor(
  ov::element::u8, {1, 640, 640, 3}, input.data);
```

构造函数参数依次为元素类型、张量形状和外部数据指针。项目常用：

- `ov::element::u8`：相机图像输入，通常是 `CV_8UC3`。
- `ov::element::f32`：分类器归一化后的浮点输入。
- `ov::Shape`：`std::vector<size_t>` 形式的维度列表。

### 2. 常用接口

```cpp
auto shape = tensor.get_shape();
tensor.set_shape({1, 3, 640, 640});
float * ptr = tensor.data<float>();
const float * read_ptr = tensor.data<const float>();
```

- `get_shape()`：读取维度；项目用 `shape[1]`、`shape[2]` 解析检测输出。
- `set_shape()`：修改动态/可调整输入张量的形状；Buff 检测器设置为 `{1, 3, 640, 640}`。
- `data<T>()`：取得连续数据指针，需保证 `T` 与张量元素类型匹配。

注意：张量的形状和内存布局必须与模型输入一致。项目自动瞄准 YOLO 路径显式创建 NHWC 的 `u8` 输入，再通过预处理器转换到模型的 NCHW/f32 输入；Buff 路径另有手工把图像转换为 CHW 的 `fill_tensor_data_image` 辅助函数。

## 七、预处理 API：`ov::preprocess`

项目 YOLOv5、YOLOv8、YOLO11 和多线程检测器使用：

```cpp
ov::preprocess::PrePostProcessor ppp(model);
auto & input = ppp.input();

input.tensor()
  .set_element_type(ov::element::u8)
  .set_shape({1, 640, 640, 3})
  .set_layout("NHWC")
  .set_color_format(ov::preprocess::ColorFormat::BGR);

input.model().set_layout("NCHW");

input.preprocess()
  .convert_element_type(ov::element::f32)
  .convert_color(ov::preprocess::ColorFormat::RGB)
  .scale(255.0);

model = ppp.build();
```

### 1. `PrePostProcessor`

`ov::preprocess::PrePostProcessor` 为模型输入/输出建立预处理描述，`build()` 后返回包含预处理步骤的新模型。

### 2. `input()`、`tensor()`、`model()`、`preprocess()`

- `ppp.input()`：取得默认输入的预处理对象。
- `input.tensor()`：描述实际提供给运行时的张量。
- `input.model()`：描述模型本身要求的输入。
- `input.preprocess()`：添加数据类型、颜色和数值范围转换。

### 3. 类型、形状和布局

- `set_element_type(ov::element::u8)`：声明原始相机图像为 8 位无符号数据。
- `set_shape({1, H, W, 3})`：声明批次、高、宽、通道。
- `set_layout("NHWC")`：输入内存顺序为批次、高、宽、通道。
- `set_layout("NCHW")`：模型节点顺序为批次、通道、高、宽。

项目输入尺寸：YOLOv5/YOLO11 为 `640 x 640`，YOLOv8 为 `416 x 416`。

### 4. 颜色和数值转换

- `set_color_format(ColorFormat::BGR)`：输入来自 OpenCV，相机帧通道顺序为 BGR。
- `convert_color(ColorFormat::RGB)`：转换为模型需要的 RGB。
- `convert_element_type(ov::element::f32)`：将 `u8` 转为浮点。
- `scale(255.0)`：项目代码按该链式调用配置数值缩放；实际使用时应结合模型训练时的归一化约定核对比例方向。

如果模型需要调整尺寸，也可以使用 `resize(...)`；本项目的 YOLO 代码在 OpenVINO 之外先用 OpenCV letterbox/缩放到固定画布，因此 `resize` 在多线程检测器中只是注释示例。

## 八、设备和性能提示

```cpp
ov::hint::performance_mode(ov::hint::PerformanceMode::LATENCY)
ov::hint::performance_mode(ov::hint::PerformanceMode::THROUGHPUT)
```

`ov::hint::performance_mode` 返回编译模型的配置属性，传给 `compile_model`。设备字符串由 YAML 的 `device` 决定；`AUTO` 允许 OpenVINO 在可用设备间选择，`CPU` 强制使用 CPU。

项目顶层和三个任务子目录把 `OpenVINO_DIR` 固定为：

```text
/opt/intel/openvino_2024.6.0/runtime/cmake/
```

因此更换安装位置时，需要同步调整顶层、`tasks/auto_aim`、`tasks/auto_buff` 和 `tasks/omniperception` 的 CMake 配置。

## 九、项目中的四种用法

| 模块 | 模型/设备 | OpenVINO 用法 |
| --- | --- | --- |
| `auto_aim/yolos/yolov5.cpp` | YOLOv5，配置设备，LATENCY | `read_model`、预处理器、`compile_model`、同步推理 |
| `auto_aim/yolos/yolov8.cpp` | YOLOv8，配置设备，LATENCY | 同上，输入尺寸为 `416 x 416` |
| `auto_aim/yolos/yolo11.cpp` | YOLO11，配置设备，LATENCY | 同上，输入尺寸为 `640 x 640` |
| `auto_aim/classifier.cpp` | 分类模型，AUTO，LATENCY | `ov::Tensor` 为 `{1,1,32,32}` 的 `f32` |
| `auto_aim/multithread/mt_detector.cpp` | 多线程 YOLO，配置设备，THROUGHPUT | 每个请求独立处理输入输出 |
| `auto_buff/yolo11_buff.cpp` | Buff YOLO11，CPU | 手工 CHW 填充、读取输出和关键点 |
| `omniperception/*` | 复用 `auto_aim::YOLO` | 多相机并行创建 YOLO/OpenVINO 实例 |

## 十、常见排查点

1. **找不到 OpenVINO**：检查 `OpenVINO_DIR` 是否指向包含 `OpenVINOConfig.cmake` 的目录，并确认 2024.6 runtime 已安装。
2. **IR 加载失败**：确认 `.xml` 和 `.bin` 文件同名且位于配置指定路径。
3. **输入维度错误**：对照模型的输入端口检查 NCHW/NHWC、尺寸和 batch；使用 `network.inputs()` 打印实际形状。
4. **数据类型错误**：`ov::Tensor` 的 `u8`/`f32` 必须与实际内存匹配；`data<T>()` 的模板类型不要写错。
5. **颜色或结果异常**：确认 OpenCV BGR 到模型 RGB 的转换，以及缩放/归一化是否与训练导出设置一致。
6. **性能不符合预期**：单帧任务使用 `LATENCY`，多路并行任务使用 `THROUGHPUT`；同时确认 `device` 能被 OpenVINO 枚举到。

## 十一、项目实现中的注意点

`YOLO11_BUFF::get_multicandidateboxes` 中有一个局部变量也命名为 `input_tensor`：

```cpp
ov::Tensor input_tensor(ov::element::u8, {1, 640, 640, 3}, input.data);
infer_request.infer();
```

它会遮蔽类成员 `input_tensor`，且没有调用 `infer_request.set_input_tensor(input_tensor)`。因此阅读或修改 Buff 推理路径时，应区分“类成员输入张量”和“局部输入张量”，并确认请求实际绑定的输入内存。这是项目源码现状的排查点，不是 OpenVINO API 的推荐写法。
