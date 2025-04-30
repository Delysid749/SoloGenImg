# SoloGenImg

## 项目简介

SoloGenImg是一个基于SinGAN (Single Image Generative Adversarial Network) 的图像生成项目，旨在从单张图像中生成多个相似但不完全相同的图像变体。本项目采用了SinGAN的变体架构，通过多尺度金字塔结构，逐层增加分辨率，实现高质量图像生成。

## 技术特点

- **单图像训练**：无需大量数据集，仅需一张图像即可训练模型
- **多尺度生成**：从低分辨率到高分辨率逐层训练和生成
- **灵活控制**：可调整噪声参数、训练步数等控制生成效果
- **无条件生成**：不需要额外的条件输入，可直接生成多样化图像

## 安装指南

### 克隆仓库

```bash
git clone https://github.com/Delysid749/SoloGenImg
```

### 安装依赖

```bash
python -m pip install -r ./requirements.txt
```

### 项目配置

如果使用**PyCharm**，请右键点击`generation`文件夹并选择"Mark Directory as Sources Root"。这样可以确保`utils`中的依赖关系正常工作。

### 系统要求

- Python 3.x
- CUDA支持的NVIDIA GPU（推荐）
- 核心依赖包：
  - PyTorch
  - torchvision
  - matplotlib
  - scikit-image
  - scikit-learn
  - scipy
  - numpy
  - tensorboardx

### 确认CUDA可用

```bash
nvidia-smi
```

如果CUDA不可用，您需要安装支持CUDA的PyTorch版本：

1. 检查PyTorch类型：
```python
import torch
print(torch.cuda.is_available())
```

2. 如果返回`False`，请卸载当前PyTorch：
```bash
pip uninstall torch
```

3. 从官方网站安装支持CUDA的PyTorch：
访问 https://pytorch.org/ 选择合适的版本下载安装

## 使用方法

### 训练模型

将目标图像上传到`images`文件夹，例如`balloons.png`。注意，图像大小应该在128×128到256×256之间（推荐256×256）。

```bash
python ./generation/main.py --root ./images/balloons.png
```

命令格式：

```
python3 main.py --root <图像路径>
```



### 评估结果

测试训练好的模型：

```bash
python ./generation/main.py --root ./images/balloons.png --evaluation --model-to-load ./results/2025-02-26_11-17-13/g_multivanilla.pt --amps-to-load ./results/2025-02-26_11-17-13/amps.pt --num-steps 100 --batch-size 16
```

命令格式：

```
python3 main.py --root <图像路径> --evaluation --model-to-load <模型路径> --amps-to-load <振幅参数路径> --num-steps <样本数量> --batch-size <批次大小>
```

参数说明：
- `<图像路径>`：源图像的路径
- `<模型路径>`：训练好的生成器模型路径
- `<振幅参数路径>`：训练好的振幅参数路径
- `<样本数量>`：生成的样本数量
- `<批次大小>`：批处理大小

### 查看生成的图像

您可以在项目根目录下的`results`文件夹中找到所有生成结果，包括训练日志、模型权重和生成的图像。

## 模型架构

### 生成器 (Generator)

- 架构：`g_multivanilla`
- 由多个`Vanilla`模块组成，每个模块对应不同的尺度
- 使用LeakyReLU激活（负斜率0.2）
- 最终输出层使用Tanh()
- 参数量：约1.48M

### 判别器 (Discriminator)

- 架构：`d_vanilla`
- 由多个BasicBlock组成，包含卷积、批归一化和LeakyReLU
- 最终Conv2D输出单通道值，用于区分真实和生成图像
- 参数量：约29K

## 超参数设置

### 训练参数

- 学习率：0.0004
- 训练步数：每个尺度2000步
- 批大小：1（SinGAN通常是单样本训练）
- 多尺度训练：从16×16到200×200逐步放大
- Adam优化器参数：betas=[0.5, 0.9]
- 重建损失权重：10.0
- 对抗损失权重：1.0
- 梯度惩罚权重：0.1

### 评估参数

- 批大小：16
- 生成步数：100

## 项目结构

├── generation/ # 主要代码目录

│ ├── models/ # 模型定义

│ ├── utils/ # 工具函数

│ ├── data/ # 数据处理

│ ├── main.py # 主程序入口

│ └── trainer.py # 训练器实现

├── images/ # 输入图像目录

├── results/ # 结果输出目录

├── figures/ # 文档图表目录

├── README.md # 项目说明文档

├── training.md # 训练过程分析

├── evaluation.md # 评估结果分析

└── requirements.txt # 项目依赖

## 训练过程分析

训练按照尺度(scale)逐层进行，从s0(16×16)到s10(200×200)：

### 损失函数

| 损失项                      | 含义                             |
| --------------------------- | -------------------------------- |
| **D** (判别器总损失)        | `D = D_r + D_f + D_gp`           |
| **D_r** (Real Loss)         | 真实图像的判别器损失             |
| **D_f** (Fake Loss)         | 生成图像的判别器损失             |
| **D_gp** (Gradient Penalty) | 梯度惩罚项(稳定训练)             |
| **G** (生成器总损失)        | `G = G_recon + G_adv`            |
| **G_recon**                 | 重建损失(鼓励生成图像与输入相似) |
| **G_adv**                   | 对抗损失(生成器欺骗判别器的能力) |

### 多尺度金字塔

| 尺度 | 尺寸    | 振幅参数(amp) |
| ---- | ------- | ------------- |
| s0   | 16×16   | 1.000         |
| s1   | 21×21   | 0.026         |
| s2   | 26×26   | 0.033         |
| ...  | ...     | ...           |
| s10  | 200×200 | 0.014         |

s0(16×16)是最基础的尺度，振幅最大(1.0)，决定最底层的结构。随着尺度增加，振幅降低，表明较高分辨率层只需进行小幅修改即可匹配原图风格。

## 优化建议

1. **增加训练步数**：现在是2000步，可以提高到5000以获得更精细的细节
2. **调整噪声权重**：目前是0.1，可以尝试降低，以减少高频噪声
3. **增加批大小**：如果GPU允许，可以增加到2-4以提高稳定性
