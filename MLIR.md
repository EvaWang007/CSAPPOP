## 工作流程

在 MLIR 的视角下，模型的导出其实是一个 分层（Multi-Level） 的过程：

PyTorch 层：你写的 forward 函数（Python 逻辑）。

Torch-ONNX 转换层：这里就需要 onnxscript。它会将你的 permute、reshape 和 stack 这种高级指令，翻译成 ONNX 定义的标准数学算子。

ONNX 层 (dt_raw.onnx)：这就是我们说的“网表”。

MLIR 优化层 (OpenVINO)：这是重头戏。它会扫描这个网表，执行算子融合（Fusion）

导出成功！！🐍

.onnx 文件：保存模型的“骨架”（即 MLIR 最关心的算子连接逻辑）。

.data / .onnx.data 文件：保存真实的权重数值（Linear 层的矩阵参数）


## 实现原因

🫀MLIR编译器通过Torch-ONNX转换层生成的onnx计算图追踪文件，对现有的模型进行算子融合和常量折叠之后得到的模型结构文件dt_raw.xml/dt_raw.bin。

***MLIR的编译过程是一个“逻辑合并”***：

👁️算子融合 (Fusing)：
在你的代码里，embed_state 是 Linear 层。在 ONNX 里，它是 MatMul -> Add。
MLIR 的操作：它发现你的 Intel Ultra 9 处理器有专门的指令（如 AVX-512 VNNI）可以一次性完成“乘加”运算。于是它把两个节点合并成一个 FullyConnected 节点。

👁️常量折叠 (Constant Folding)：
如果你在模型里有一些固定的权重计算（比如预设的 LayerNorm 参数），MLIR 会在编译阶段就把结果算出来，直接存在 .bin 文件里，而不是等运行的时候再让 CPU 去算一遍。

👁️维度消除 (Reshape Elimination)：
你最担心的 permute 和 reshape。MLIR 会分析数据流向。如果它发现下一步操作（比如 Transformer 的 Attention）可以直接通过改变读取步长（Stride）来完成，它就会把这些专门搬运数据的节点直接删掉。

模型结构的两个文件（结构与权重的分离）
这种设计是为了工业级部署的效率：

.xml (The Brain)：这是优化后的 计算图描述。它是一个 XML 格式的文本，记录了：第 1 步做矩阵乘法，第 2 步做激活。它非常小，你可以用文本编辑器打开它，甚至能看到里面标注的 Intel CPU 优化标记。

.bin (The Muscles)：这是 权重二进制。它存储了你 Decision Transformer 所有的 embedding 矩阵和 linear 层的参数。它是经过排列优化的，能够以最快速度被加载到 CPU 的三级缓存（L3 Cache）中。


## 连接硬件

现在有的这个dt_raw文件是我刚刚使用MLIR处理我之前用pytorch写的DT模型之后获得的经过优化之后更加适用于硬件加速环境的代码文件。它们不会自己运行，需要一个 “载体” 去加载它们、给它们喂数据、并取回结果。你接下来要写的 C++ 文件，其核心任务是构建一个高效的推理引擎。

这个 C++ 程序主要充当“指挥官”的角色：

加载模型 (Loading)：
使用 OpenVINO 的 C++ Runtime 库，把 .xml 和 .bin 读入内存。此时，OpenVINO 会针对你的 Intel Ultra 9 (U9 140T) 进行最后的指令集绑定（例如调用 AVX-512 指令集）。

数据预处理 (Pre-processing)：
你在 Python 里用 torch.randn 生成数据，在 C++ 里，你需要把传感器数据（比如 AUV 的姿态 states、奖励 rtgs）打包成 OpenVINO 能理解的 ov::Tensor 格式。

异步执行 (Asynchronous Inference)：
这是结合你“线程池”知识的关键！ 你不希望 CPU 在等模型算完时卡死。你会用一个线程喂数据，让另一个线程处理预测出的 actions。

后处理 (Post-processing)：
模型输出的是一堆概率数字（Tensor），你的 C++ 代码需要从中选出值最大的那个，转化成 AUV 真实的控制指令（如：左转 15°）。











