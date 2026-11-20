# NeRFBlendShape 复现接续记录

本仓库的训练实现和 [NeRFBlendShape 官方实现](https://github.com/USTC3DV/NeRFBlendShape-code) 需要分别验证。官方仓库提供独立的推理入口、预处理数据和权重。先用官方配套样例验证环境、输入格式和渲染输出，再接入论文第四章的统一测试序列。

## 官方样例准备

1. 从官方 README 指向的下载目录取得 `pretrained.zip` 和 `preprocessed.tar.gz`。本次服务器直下权重约 45 KB/s，Mac 下载 3.76 GB 权重包约 69 秒。跨机时可以先在 Mac 解出所需人物的文件，再传单个人物的权重和少量测试素材。
2. 官方包的 `id1` 权重目录包含 `pretrained.pth.tar`、`transforms.json`、`min_46.txt`、`max_46.txt` 和视频。正式启动前核对权重和预处理数据对应人物，并记录源文件 SHA-256 及样本帧号。
3. 用隔离的 Linux 环境安装官方依赖和 tiny-cuda-nn，先用一个已配对的测试帧检查加载和渲染，再扩大样本范围。共享服务器上单卡串行运行，监测本任务的显存并保留其他作业的余量。

## 本仓库的输入

本仓库的训练入口读取与视频帧对齐的 `expr.npy`、`expr_min.npy`、`expr_max.npy`。README 说明原预处理使用了未公开的表情提取器。使用其他跟踪器时，需要记录基底、系数顺序、归一化与帧号，并使用在相同输入约定下训练的检查点。官方的 46 维系数与其他跟踪器输出需显式核对。仓库里的 `data/example.mp4` 是随源码提供的示例视频。

## 官方样例实测（2026-11-20）

官方提交 `8b2a2402c8f5fc6f2af05e009531241df2eb94f7` 已在 RTX 4090、Python 3.9、PyTorch 2.3.1、CUDA 12.1 下跑通。用隔离环境安装 OpenCV 4.11.0.86 和 NumPy 1.24.3，原版 tiny-cuda-nn 1.6 与项目的 `raymarching`、`gridencoder` 扩展使用 C++17，固定目标架构为 `sm_89`。在 Conda 工具链中显式配置 CUDA 头文件、GPU 驱动链接桩与编译器 sysroot。构建阶段隐藏 GPU，使用两个编译作业。`normal_test` 的渲染路径不使用 LPIPS，本轮副本让纯推理跳过该模块初始化，避免加载额外的 VGG 权重。

- 权重取自作者 `pretrained.zip` 的 `id1/pretrained.pth.tar`。GPU 回读的 SHA-256 为 `99703db2300fc7a67726faf0b830dc6480fee3ba4bda9701cc08820b92b3f369`。相机、表情以及 `min_46.txt`、`max_46.txt` 取自同一权重包，原始 RGB 和关键点取自同人物预处理包。
- 运行 `run_nerfblendshape.py` 的 `normal_test`，四个 `*_idpath` 都指向抽帧后的 `transforms.json`，保留 `--test_start 0 --fp16 --tcnn --cuda_ray --basis_num 46 --add_mean --to_mem`。本轮验证 3251、3252、3253 三帧，生成六张 512×512 渲染。`_nvs.png` 使用官方代码生成的新视角，表情仍来自同一驱动帧。
- 固定相机的单帧控制保留 3251 的相机矩阵和内参，替换为 2911 的 `exp`、`exp_ori`。另生成原视角和新视角各一张。两次共享卡短跑峰值均为 1460 MiB，最低采样剩余显存分别为 14744 MiB 和 14732 MiB。
- 官方测试代码加载检查点时报告训练阶段的 `exp_delta` 为额外键，推理模型未启用这一参数。两个运行均无缺失键，日志记录写图结束。预处理包与权重包的内参一致，所测三帧的姿态与表情存在小量差异，保留权重包中的配套驱动参数。原视频为绿幕，官方推理采用白背景。

本轮实测针对官方实现及其公开检查点。本仓库的独立推理复现仍需匹配的检查点和表情输入，论文同人对照需统一源视频、帧号、相机、裁剪与背景。
