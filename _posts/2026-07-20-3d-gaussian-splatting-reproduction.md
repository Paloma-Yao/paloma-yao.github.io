---
layout: post
title: "从零复现 3D Gaussian Splatting：Ubuntu 环境搭建与 Lego 实验"
date: 2026-07-20 20:00:00+0800
description: "记录在 Ubuntu 24.04 上配置 3DGS，并完成 Lego 数据集训练、渲染与评估。"
tags: [3D Gaussian Splatting, NeRF, CUDA, Computer Vision]
related_posts: false
---

## 研究背景

目前确定的课题标题是：

> 基于光学标记之多姿态物件拍摄对位与三维高斯潑濺重建

本阶段先掌握多视角三维重建流程，再研究光学标记对多姿态拍摄对位和三维高斯潑濺重建质量的影响。今天使用官方 3DGS 实现和 NeRF Synthetic Lego 数据集完成基础复现。

## 实验环境

- Ubuntu 24.04.4
- NVIDIA GeForce RTX 2080 SUPER，8 GB 显存
- NVIDIA Driver 595.71.05
- CUDA Toolkit 12.8
- Python 3.10、PyTorch CUDA 12.8

使用 Miniforge 建立独立 Conda 环境，并为 RTX 2080 SUPER 指定计算能力 7.5 编译 `diff-gaussian-rasterization` 和 `simple-knn`。过程中解决了 CUDA 头文件缺少 `#include <cstdint>` 及新版 Pillow 不接受 `np.byte` 图像数组的问题。

## 数据集与训练

`nerf_synthetic/lego` 是 Blender 格式数据集，包含训练视角和测试视角；NeRF 是数据集来源和另一类神经渲染方法的名称，本次训练的是 3DGS。

```bash
python train.py -s ~/workspace/gaussian-splatting/data/nerf_synthetic/lego -m ~/workspace/gaussian-splatting/output/lego_smoke --iterations 1000 --resolution 4 --white_background --data_device cpu --test_iterations -1 --save_iterations 1000 --disable_viewer
```

之后使用 `--eval` 留出测试视角，完成 7000 次迭代。7000 是优化迭代次数，不是 7000 张训练图片；模型保存为 `output/lego_7k_eval/point_cloud/iteration_7000/point_cloud.ply`。

## 渲染与评估

```bash
python render.py -m ~/workspace/gaussian-splatting/output/lego_7k_eval --skip_test
python render.py -m ~/workspace/gaussian-splatting/output/lego_7k_eval --skip_train
python metrics.py -m ~/workspace/gaussian-splatting/output/lego_7k_eval
```

结果为：

| 指标 | 数值 |
| --- | ---: |
| SSIM | 0.2224 |
| PSNR | 2.3165 |
| LPIPS | 0.5460 |

指标偏低，是因为本次使用了 `resolution 4` 和较短配置；但训练、保存、渲染、测试和指标计算的完整链路已经跑通，可以视为工程复现成功，而不是最终质量结果。

## 下一步

1. 提取光学标记角点或中心点，辅助不同姿态之间的相机对位；
2. 比较有标记和无标记时的相机位姿误差；
3. 统计对位策略对 PSNR、SSIM、LPIPS 和点云完整性的影响；
4. 在 Lego 数据集验证后，使用真实物件和多姿态拍摄数据实验。

后续可参考 [Nerfstudio 的 Splatfacto 文档](https://docs.nerf.studio/nerfology/methods/splat.html) 做交互式可视化，但应与当前官方 graphdeco 实现分开建立环境。
