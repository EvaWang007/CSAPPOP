## 导出ONNX文件

在 MLIR 的视角下，模型的导出其实是一个 分层（Multi-Level） 的过程：

PyTorch 层：你写的 forward 函数（Python 逻辑）。

Torch-ONNX 转换层：这里就需要 onnxscript。它会将你的 permute、reshape 和 stack 这种高级指令，翻译成 ONNX 定义的标准数学算子。

ONNX 层 (dt_raw.onnx)：这就是我们说的“网表”。

MLIR 优化层 (OpenVINO)：这是重头戏。它会扫描这个网表，执行算子融合（Fusion）
