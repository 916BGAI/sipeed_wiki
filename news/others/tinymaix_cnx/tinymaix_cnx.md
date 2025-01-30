---
title: TinyMaix ：超轻量级推理框架
keywords: TinyMaix, Sipeed, 框架, 机器学习
date: 2022-08-24
tags: TinyMaix, 推理框架
---

<!-- more -->

## 介绍

TinyMaix 是面向单片机的超轻量级的神经网络推理库，即 TinyML 推理库，可以让你在任意单片机上运行轻量级深度学习模型。

**关键特性**
- 核心代码少于 **400行**(`tm_layers.c`+`tm_model.c`+`arch_cpu.h`), 代码段(.text)少于**3KB**   
- 低内存消耗，甚至 **Arduino ATmega328** (32KB Flash, 2KB Ram) 都能基于 TinyMaix 跑 mnist(手写数字识别)
- 支持 **INT8/FP32/FP16** 模型，实验性地支持 **FP8** 模型，支持 keras h5 或 tflite 模型转换 
- 支持多种芯片架构的专用指令优化: **ARM SIMD/NEON/MVEI，RV32P, RV64V** 
- 友好的用户接口，只需要 load/run 模型~
- 支持全静态的内存配置(无需 malloc )
- 即将支持 [MaixHub](https://maixhub.com) **在线模型训练** 

**在Arduino ATmega328上运行 mnist demo 实例**
```
mnist demo
0000000000000000000000000000
0000000000000000000000000000
0000000000000000000000000000
000000000077AFF9500000000000
000000000AFFFFFFD10000000000
00000000AFFFD8BFF70000000000
00000003FFD2000CF80000000000
00000004FD10007FF40000000000
00000000110000DFF40000000000
00000000000007FFC00000000000
0000000000004FFE300000000000
0000000000008FF9000000000000
00000000000BFF90000000000000
00000000001EFE20000000000000
0000000000CFF800000000000000
0000000004FFB000000000000000
000000001CFF8000000000000000
000000008FFA0000000000000000
00000000FFF10000000000000000
00000000FFF21111000112999900
00000000FFFFFFFFA8AFFFFFFF70
00000000AFFFFFFFFFFFFFFA7730
0000000007777AFFF97720000000
0000000000000000000000000000
0000000000000000000000000000
0000000000000000000000000000
0000000000000000000000000000
0000000000000000000000000000
===use 49912us
0: 0
1: 0
2: 89
3: 0
4: 1
5: 6
6: 1
7: 0
8: 0
9: 0
### Predict output is: Number 2, prob=89
```

## TODO 
1. 将 `tm_layers.c` 优化到 `tm_layers_O1.c`, 目标提升速度到 `1.4~2.0X`
2. 针对 64/128/256/512KB 内存限制，找到合适的骨干网络
3. 增加例程：Detector,KWS,HAR,Gesture,OCR,...
4. ...

如果想参与进 TinyMaix 的开发，或者想与 TinyML 爱好者交流，   
请加入 telegram 交流群：https://t.me/tinymaix  



## TinyMaix 设计思路

TinyMaix 是专为低资源的单片机所设计的 AI 神经网络推理框架，通常被称为 **TinyML**  

现在已经有很多 TinyML 推理库，比如 TFLite micro, microTVM, NNoM, 那为什么又捏了 TinyMaix 这个轮子呢?   

TinyMaix 是两个周末业余时间完成的项目，所以它足够简单，可以再30分钟内走读完代码，可以帮助TinyML新手理解它是怎么运行的。

TinyMaix 希望成为一个足够简单的 TinyML 推理库，所以它放弃了很多特性，并且没有使用很多现成的 NN 加速库，比如 CMSIS-NN

在这个设计思路下，TinyMaix 只需要5个文件即可编译~

我们希望 TinyMaix 可以帮助任何单片机运行 AI 神经网络模型, 并且每个人都能移植 TinyMaix 到自己的硬件平台上~

> 注意：虽然 TinyMaix 支持多架构加速，但是它仍然需要更多工作来平衡速度和尺寸

### 设计特性
- [x] 最高支持到 mobilenet v1, RepVGG 的骨干网络
  - 因为它们对单片机来说是最常用的，最高效的结构
  - [x] 基础的 Conv2d, dwConv2d, FC, Relu/Relu6/Softmax, GAP, Reshape
  - [ ] MaxPool, AvgPool (现在使用 stride 代替)
- [x] FP32 浮点模型, INT8 量化模型, **FP16**半精度模型
- [x] 转换 keras h5 或 tflite 到 tmdl
  - 简单模型使用keras/tf训练已经足够
  - 复用了tflite现成的量化功能
- [x] 模型统计功能
  - 可选以减少代码尺寸

### 可考虑添加的特性
- [ ] INT16 量化模型
  - 优点: 
    - 更精确
    - 对于 SIMD/RV32P 指令加速更友好
  - 缺点: 
    - 占用了 2 倍的 FLASH/RAM
- [ ] Concat 算子
  - 优点: 
    - 支持 mobilenet v2, 模型精度更高
  - 缺点: 
    - 占用了 2 倍的 RAM
    - concat 张量占用了更多时间，使得模型运算变慢
    - 需要更多转换脚本工作转换分支模型到扁平结构
- [ ] Winograd 卷积优化
  - 优点:
    - 可能加速卷积计算
  - 缺点: 
    - 增加了 RAM 空间和带宽消耗
    - 增大了代码段(.text)尺寸
    - 需要很多变换，弱单片机可能会消耗更多时间
    
### 不考虑添加的特性
- [ ] BF16 模型
  - 多数单片机不支持 BF16 计算
  - 精度不会比 INT16 高太多
  - 占用了 2 倍的 FLASH/RAM
- [ ] AVX/vulkan 加速
  - TinyMaix 是为单片机设计的，所以不考虑电脑/手机的支持
- [ ] 其他多样化的算子
  - TinyMaix 仅为单片机提供基础模型算子支持，如果你需要更特殊的算子，可以选择 TFlite-micro/TVM/NCNN... 

## 例程体验

### mnist
MNIST 是手写数字识别任务，简单到以至于可以在 ATmega328 这样的 8 位单片机上运行。  
在电脑上测试： 
```
cd examples/mnist
mkdir build
cd build 
cmake ..
make
./mnist
```

### mbnet
mbnet (mobilenet v1) 是适用于移动手机设备的简单图像分类模型，不过对单片机来说也稍微困难了些。
例程里的模型是 mobilenet v1 0.25，输入 128x128x3 的RGB图像，输出 1000 分类的预测
它需要至少 128KB SRAM 和 512KB Flash, STM32F411 是典型可以运行该模型的最低配置。

在 PC 上测试运行 mobilenet 1000分类图片例程
```
cd examples/mbnet
mkdir build
cd build 
cmake ..
make
./mbnet
```

## 如何使用 (API)
### 加载模型
```
tm_err_t tm_load  (tm_mdl_t* mdl, const uint8_t* bin, uint8_t*buf, tm_cb_t cb, tm_mat_t* in);   
```
mdl: 模型句柄;   
bin: 模型bin内容;   
buf: 中间结果的主缓存；如果NULL，则内部自动malloc申请；否则使用提供的缓存地址
cb: 网络层回调函数;   
in: 返回输入张量，包含输入缓存地址 //可以忽略之，如果你使用自己的静态输入缓存

### 移除模型
```
void     tm_unload(tm_mdl_t* mdl);                         
```
### 输入数据预处理
```
tm_err_t tm_preprocess(tm_mdl_t* mdl, tm_pp_t pp_type, tm_mat_t* in, tm_mat_t* out);              
```
TMPP_FP2INT    //用户自己的浮点缓存转换到int8缓存
TMPP_UINT2INT  //典型uint8原地转换到int8数据；int16则需要额外缓存
TMPP_UINT2FP01 //uint8转换到0~1的浮点数 u8/255.0  
TMPP_UINT2FPN11//uint8转换到-1~1的浮点数

### 运行模型
```
tm_err_t tm_run   (tm_mdl_t* mdl, tm_mat_t* in, tm_mat_t* out);
```


## 如何移植

TinyMaix的核心文件只有这5个：`tm_model.c`, `tm_layers.c`, `tinymaix.h`, `tm_port.h`, `arch_xxx.h`  

如果你使用没有任何指令加速的普通单片机，选择 `arch_cpu.h`, 否则选择对应架构的头文件 

然后你需要编辑 `tm_port.h`，填写你需要的配置，所有配置宏后面都有注释说明 

注意 `TM_MAX_CSIZE`,`TM_MAX_KSIZE`,`TM_MAX_KCSIZE` 会占用静态缓存。

最后你只需要把他们放进你的工程里编译~

## 怎样训练/转换模型

在 examples/mnist 下有训练脚本可以学习如何训练基础的mnist模型

> 注意：你需要先安装TensorFlow (>=2.7) 环境.

完成训练并保存h5模型后，你可以使用以下脚本转换原始模型到 tmdl 或者 c 头文件。 

1. h5_to_tflite.py   
  转换 h5 模型到浮点或者 int8 量化的 tflite 模型
  python3 h5_to_tflite.py h5/mnist.h5 tflite/mnist_f.tflite 0   
  python3 h5_to_tflite.py h5/mnist.h5 tflite/mnist_q.tflite 1 quant_img_mnist/ 0to1   
2. tflite2tmdl.py
  转换 tflite 文件到 tmdl 或者 c 头文件   
  python3 tflite2tmdl.py tflite/mnist_q.tflite tmdl/mnist_q.tmdl int8 1 28,28,1 10  
```
================ pack model head ================
mdl_type   =0
out_deq    =1
input_cnt  =1
output_cnt =1
layer_cnt  =6
buf_size   =1464
sub_size   =0
in_dims    = [3, 28, 28, 1]
out_dims   = [1, 1, 1, 10]
================   pack layers   ================
CONV_2D
    [3, 28, 28, 1] [3, 13, 13, 4]
    in_oft:0, size:784;  out_oft:784, size:680
    padding valid
    layer_size=152
CONV_2D
    [3, 13, 13, 4] [3, 6, 6, 8]
    in_oft:784, size:680;  out_oft:0, size:288
    padding valid
    layer_size=432
CONV_2D
    [3, 6, 6, 8] [3, 2, 2, 16]
    in_oft:0, size:288;  out_oft:1400, size:64
    padding valid
    layer_size=1360
MEAN
    [3, 2, 2, 16] [1, 1, 1, 16]
    in_oft:1400, size:64;  out_oft:0, size:16
    layer_size=48
FULLY_CONNECTED
    [1, 1, 1, 16] [1, 1, 1, 10]
    in_oft:0, size:16;  out_oft:1448, size:16
    layer_size=304
SOFTMAX
    [1, 1, 1, 10] [1, 1, 1, 10]
    OUTPUT!
    in_oft:1448, size:16;  out_oft:0, size:56
    layer_size=48
================    pack done!   ================
    model  size 2.4KB (2408 B) FLASH
    buffer size 1.4KB (1464 B) RAM
    single layer mode subbuff size 1.4KB (64+1360=1424 B) RAM
Saved to tmdl/mnist_q.tmdl, tmdl/mnist_q.h
```

现在你有了 tmdl 或者 C 头文件，把它放到你的工程里编译吧~

## 使用 Maixhub 在线训练模型

TODO

## 怎样添加新平台的加速代码

TinyMaix 使用基础的点积函数加速卷积运算   
你需要在 src 里添加 arch_xxx_yyy.h, 并添上你自己平台的点积加速函数：
```
TM_INLINE void tm_dot_prod(mtype_t* sptr, mtype_t* kptr,uint32_t size, sumtype_t* result);
```


## 贡献/联系

如果你需要向TinyMaix贡献代码，请先阅读“TinyMaix设计思路”一节，我们只需要“设计内的特性”和“可考虑添加的特性”。

如果你想要提交你的移植测试结果，请提交到 benchmark.md.

我们非常欢迎你移植 TinyMaix 到自己的芯片/板子上，这会证明使用 TinyMaix 运行深度学习模型是非常容易的事情~

如果你对 TinyMaix 的使用和移植有问题，可以在此仓库提交 Issues。

如果你有商业或私有项目咨询，你可以发邮件到 support@sipeed.com 或 zepan@sipeed.com (泽畔).  