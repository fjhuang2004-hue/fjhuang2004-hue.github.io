---
title: "从头设计蛋白：小白也能懂的完整教程（RFdiffusion → ProteinMPNN → AlphaFold3）"
date: 2026-05-15
draft: false
description: "从 SSH 登录服务器开始，手把手教你用 RFdiffusion 生成全新蛋白骨架、ProteinMPNN 设计序列、AlphaFold3 验证折叠效果，形成完整的设计-验证闭环。"
tags: ["从头设计", "RFdiffusion", "ProteinMPNN", "AlphaFold3", "教程", "蛋白设计"]
categories: ["从头设计"]
---

## 前言

你是一个合成生物学方向的研究生，想要设计一个全新的酶——不是从自然界里找，而是从零开始"画"一个。听起来像科幻？2023-2024 年，AI 驱动的蛋白设计工具已经让这件事变得非常接地气了。

这篇教程会带你走完一个完整的从头设计流程，从**SSH 登录服务器**开始，到 **AlphaFold3 验证设计结果**结束。全过程都是手把手的命令级教学，你只需要会复制粘贴就行。

### 你将学到

| 步骤 | 工具 | 做什么 |
|------|------|--------|
| ① 环境搭建 | conda/mamba + CUDA | 准备计算环境 |
| ② 生成骨架 | **RFdiffusion** | 从噪声中生成全新蛋白骨架 |
| ③ 设计序列 | **ProteinMPNN** | 为骨架设计氨基酸序列 |
| ④ 验证折叠 | **AlphaFold3** | 验证设计的序列是否正确折叠 |
| ⑤ 迭代优化 | 综合上述 | 根据结果调整参数，迭代改进 |

### 从头设计 vs 传统方法

传统改酶：找一个天然酶 → 突变 → 筛选 → 反复。瓶颈在于**筛选**——你不可能把几百万个突变体一个个验证。

从头设计：用 AI 直接"打印"你想要的蛋白结构——**"先设计骨架，再填充序列"**。理论上你可以设计自然界中不存在的任何蛋白形状。

今天的流程用一个生动的比喻：**先画房子的骨架（RFdiffusion），再决定用什么砖瓦水泥（ProteinMPNN），最后检查房子会不会塌（AlphaFold3）。**

---

## 1. 登录服务器

> **小白提示**：如果你有自己的服务器/工作站，跳不过这一步。如果你打算用 Google Colab（免费 GPU），可以跳过本节的 SSH 部分，但后面的安装步骤需要做调整。

### 1.1 SSH 登录

SSH（Secure Shell）是远程登录 Linux 服务器的标准方式。假设你的服务器 IP 是 `123.456.789.0`，用户名是 `ubuntu`：

```bash
# 基本 SSH 登录
ssh ubuntu@123.456.789.0

# 如果有密钥文件（推荐，不需要输密码）
ssh -i ~/.ssh/my-key.pem ubuntu@123.456.789.0
```

第一次连接会提示确认主机指纹，输入 `yes` 即可。

> **小白提示**：服务器 IP 和登录方式请咨询你的 IT 管理员或云服务商（阿里云/AWS/腾讯云）。如果你用的是学校的高性能计算集群，通常会有一个专门的登录节点地址。

### 1.2 VS Code Remote（推荐）

用 VS Code 远程开发比纯命令行舒服太多了：

1. 在 VS Code 中安装 **Remote - SSH** 扩展
2. 按 `F1` → 输入 `Remote-SSH: Connect to Host...`
3. 输入你的 SSH 连接命令
4. 连接成功后，VS Code 左侧会显示远程服务器的文件系统

好处：你可以像在本地一样编辑文件、看图片、用终端。

### 1.3 tmux：防止训练中断

```bash
# 安装 tmux（如果还没装）
sudo apt install tmux -y

# 创建新会话
tmux new -s protein-design

# 以后可以随时断开（Ctrl+B 然后按 D）
# 重新连接：
tmux attach -t protein-design
```

> **为什么需要 tmux？** 训练/推理可能跑几个小时。没有 tmux，你一关电脑 SSH 连接就断了，程序也会被杀掉。tmux 让你的程序在后台持续运行。

---

## 2. 环境部署

### 2.1 安装 conda / mamba

```bash
# 下载 Miniconda（约 100MB）
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# 安装（一路 yes）
bash Miniconda3-latest-Linux-x86_64.sh

# 重新加载配置
source ~/.bashrc

# 验证
conda --version

# 安装 mamba（比 conda 快 10 倍的包管理器）
conda install -n base conda-forge::mamba -y
```

之后所有 `conda install` 都可以用 `mamba install` 替代，速度明显更快。

### 2.2 检查 GPU

蛋白设计工具基本都需要 GPU（NVIDIA 显卡）。检查你的服务器有没有：

```bash
# 检查 GPU 型号和驱动
nvidia-smi

# 期望输出类似：
# +-----------------------------------------------------------------------------+
# | NVIDIA-SMI 535.86.10    Driver Version: 535.86.10    CUDA Version: 12.2     |
# | GPU 0: NVIDIA A100 80GB PCIe                                          |
# +-----------------------------------------------------------------------------+
```

> **没有 GPU？** RFdiffusion 需要至少 12GB VRAM，AlphaFold3 需要 24GB+。免费的替代方案是 Google Colab（T4 GPU 15GB VRAM），但只能跑小任务。

### 2.3 安装 CUDA 工具包

```bash
# 注意：检查你的 GPU 驱动的 CUDA 版本（nvidia-smi 输出中有 CUDA Version）
# 这里以 CUDA 12.2 为例
conda install -c nvidia cuda-toolkit=12.2 -y
nvcc --version  # 验证
```

### 2.4 安装 PyTorch

```bash
# 从 PyTorch 官网获取适合你 CUDA 版本的命令
# https://pytorch.org/get-started/locally/
# 以 CUDA 12.1 为例：
conda install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia -y

# 验证 GPU 可用
python -c "import torch; print(f'PyTorch {torch.__version__}, CUDA available: {torch.cuda.is_available()}')"
```

> 输出 `CUDA available: True` 说明 GPU 可用。如果显示 `False`，检查 CUDA 版本是否匹配。

---

## 3. RFdiffusion：从头生成蛋白骨架

**RFdiffusion**（RoseTTAFold Diffusion）是 Baker 实验室 2023 年发表在 Nature 上的重磅工具。它的核心思想：**用扩散模型（类似 DALL·E/Midjourney 的原理）从纯噪声中逐步"去噪"，最终生成一个全新的蛋白骨架。**

### 3.1 安装

```bash
# 创建独立环境
conda create -n RFdiffusion python=3.10 -y
conda activate RFdiffusion

# 安装 PyTorch（GPU 版）
conda install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia -y

# 安装其他依赖
pip install einops omegaconf hydra-core wandb pandas seaborn matplotlib

# 克隆仓库
git clone https://github.com/RosettaCommons/RFdiffusion.git
cd RFdiffusion

# 安装 SE3Transformer 等依赖
pip install -e .
```

### 3.2 下载预训练权重（约 2GB）

```bash
# 下载权重文件
wget https://files.ipd.uw.edu/pub/RFdiffusion/6f5902ac237024bdd0c176cb93063dc4/Base_ckpt.pt -P ./models/
wget https://files.ipd.uw.edu/pub/RFdiffusion/6f5902ac237024bdd0c176cb93063dc4/Complex_beta_ckpt.pt -P ./models/

# 检查文件大小
ls -lh models/
# 期望输出：Base_ckpt.pt (~1.1GB), Complex_beta_ckpt.pt (~1.1GB)
```

> 📝 **网络慢怎么办？** 如果下载太慢，可以用 `aria2c -x 16 [URL]` 多线程下载，或通过校园网/代理加速。

### 3.3 运行第一个无条件生成

先跑一个最简单的例子——无条件生成（不给任何约束，让 AI 自由发挥）：

```bash
# 激活环境
conda activate RFdiffusion

# 进入项目目录
cd ~/RFdiffusion

# 运行无条件生成
python scripts/run_inference.py \
    inference.output_dir=./outputs/unconditional \
    inference.model_directory_path=./models \
    inference.num_designs=10 \
    inference.design_startnum=0
```

参数说明：
- `output_dir`：输出目录
- `model_directory_path`：权重文件目录
- `num_designs`：生成多少个骨架（越多越耗时）
- `design_startnum`：起始编号

运行时间：10 个骨架约 10-15 分钟（A100 GPU）。慢是正常的，因为扩散模型需要逐步去噪 50 步。

### 3.4 理解输出

运行后，`./outputs/unconditional/` 目录下会有：

```
outputs/unconditional/
├── pdb/              # PDB 格式的骨架文件（可导入 PyMOL/ChimeraX 查看）
│   ├── design_0.pdb
│   ├── design_1.pdb
│   └── ...
├── trb/              # 轨迹文件（用于后续 ProteinMPNN 输入）
│   ├── design_0.trb
│   └── ...
└── summary/          # 汇总
```

查看生成的骨架：

```bash
# 用 PyMOL 或 ChimeraX 可视化，也可以用命令行检查
# 查看设计 0 的序列信息
grep "ATOM" outputs/unconditional/pdb/design_0.pdb | awk '{print $4}' | sort -u | wc -l
echo "生成的残基数："
grep "CA" outputs/unconditional/pdb/design_0.pdb | wc -l
```

### 3.5 条件生成：指定形状

无条件生成的骨架可能毫无功能。实际设计时，我们会给定一个"支架"（如活性位点、结合口袋的位置），让 AI 围绕它生成骨架。

```bash
# 以下假设你有一个目标位点的 PDB 文件（名为 hotspot.pdb）
# 这里用一个示例：在 A 链的第 10-15 位残基位置上生成结合口袋
python scripts/run_inference.py \
    inference.output_dir=./outputs/conditional \
    inference.model_directory_path=./models \
    inference.input_pdb=./inputs/hotspot.pdb \
    inference.contig_map.contigs=[\"10-30/A40-50/10-30\"] \
    inference.num_designs=5
```

> `contigs` 参数是关键——它指定了骨架的"拼图"规则。上面的例子说：前面生成 10-30 个残基，中间固定 A 链 40-50 号残基（活性位点），后面再生成 10-30 个残基。

### 工具对比：从头设计工具速查

下表来自前期调研成果，帮助你了解 RFdiffusion 在全部从头设计工具中的位置：

| 工具 | 方法 | GPU 必需？ | 安装难度 | GitHub Stars | 一句话 |
|------|------|-----------|---------|-------------|--------|
| **RFdiffusion ⭐** | 扩散模型/骨架生成 | ✅ ≥12GB | ⭐⭐⭐⭐ | 2,865 | **从头生成蛋白骨架的首选** |
| ProteinMPNN | MPNN/序列设计 | ❌ CPU 可 | ⭐⭐ | 1,725 | 给定骨架→设计序列 |
| AlphaFold3 | 扩散模型/结构预测 | ✅ ≥24GB | ⭐⭐⭐⭐⭐ | 7,981 | 终极验证工具 |
| BindCraft | RFdiffusion+MPNN | ✅ | ⭐⭐⭐⭐ | 1,095 | 一键式结合剂设计 |
| RoseTTAFold2 | 三轨注意力网络 | ✅ | ⭐⭐⭐ | 2,246 | Baker 自家结构预测 |
| ESMFold | 语言模型 | ✅ 推荐 | ⭐⭐ | 4,074 | 秒级验证 |
| ColabFold | AF2 封装 | ❌ 云端 | ⭐ | 2,758 | 零安装入门 |
| FoldingDiff | 微软扩散模型 | ✅ | ⭐⭐⭐ | 568 | 可控制折叠类型 |
| ProtGPT2 | GPT-2 生成序列 | ✅ 推荐 | ⭐⭐ | — | 生成全新序列 |

---

## 4. ProteinMPNN：骨架→氨基酸序列

RFdiffusion 只产生了骨架（Cα 原子的三维坐标）。要得到真正的蛋白，我们需要给每个位置"填上"最合适的氨基酸。这就是 **ProteinMPNN** 的工作。

**ProteinMPNN**（Protein Message Passing Neural Network）发表于 2022 年 Science，核心能力：给定一个蛋白骨架结构，预测哪些氨基酸序列能稳定地折叠成这个结构。实验验证成功率高达 **52%**——在蛋白设计领域这是非常高的数字。

### 4.1 安装

```bash
# 创建独立环境
conda create -n proteinmpnn python=3.10 -y
conda activate proteinmpnn

# 安装 PyTorch
conda install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia -y

# 克隆仓库
git clone https://github.com/dauparas/ProteinMPNN.git
cd ProteinMPNN

# 不需要额外 pip install！代码就是可执行的 Python 脚本
# 查看结构和帮助
ls -la
```

### 4.2 将 RFdiffusion 输出转为 ProteinMPNN 输入

ProteinMPNN 需要的输入是包含**骨架所有原子**（N, CA, C, O）的 PDB 文件。RFdiffusion 输出的 PDB 已经包含了这些原子，所以可以直接用——但需要做一些格式调整：

```bash
# ProteinMPNN 需要一个文件夹中的所有 PDB 文件
# 假设 RFdiffusion 输出在 ~/RFdiffusion/outputs/unconditional/pdb/
# 把这些 PDB 复制到 ProteinMPNN 的输入目录

mkdir -p ~/ProteinMPNN/inputs
cp ~/RFdiffusion/outputs/unconditional/pdb/*.pdb ~/ProteinMPNN/inputs/
```

### 4.3 运行序列设计

```bash
cd ~/ProteinMPNN

python helper_scripts/parse_multiple_chains.py \
    --input_path=./inputs \
    --output_path=./parsed_pdbs.jsonl

python protein_mpnn_run.py \
    --jsonl_path=./parsed_pdbs.jsonl \
    --out_folder=./outputs \
    --num_seq_per_target=8 \
    --sampling_temp=0.1 \
    --model_name=v_48_020 \
    --seed=42
```

参数说明：
- `num_seq_per_target`：每个骨架设计 8 条候选序列
- `sampling_temp`：采样温度，0.1=保守（倾向于高概率氨基酸），1.0=探索（更多变体）
- `model_name`：预训练模型版本，`v_48_020` 是最新版本

运行时间：8 条序列约 2-5 分钟（CPU 即可，GPU 更快）。

### 4.4 理解输出

```
outputs/
├── seqs/               # FASTA 格式的序列
│   ├── design_0.fa
│   └── ...
├── pdb/                # 含侧链的 PDB 文件（可可视化）
│   ├── design_0_0.pdb  # design_0 的第 1 条候选序列
│   ├── design_0_1.pdb  # design_0 的第 2 条候选序列
│   └── ...
└── parsed_pdbs.jsonl
```

查看生成的序列：

```bash
cat outputs/seqs/design_0.fa
# 输出类似：
# >3T6U_0, score=0.69, global_score=0.62, seq_recovery=0.39
# MKLKKLIAYVPAVVALEVGRYARLLKEGLHPDFYIELGD...
```

**分数解读**：
- `score`：每个位置的平均负对数似然，**越低越好**（0.5-0.7 是良好范围）
- `global_score`：全局平均分数
- `seq_recovery`：与参考序列的相似度（如果有参考序列的话）

### 4.5 固定部分残基的条件设计

如果你是设计酶的活性位点，通常需要**固定**某些关键残基（如催化三联体 Ser-His-Asp），只设计其他区域：

```bash
python protein_mpnn_run.py \
    --jsonl_path=./parsed_pdbs.jsonl \
    --out_folder=./outputs_fixed \
    --num_seq_per_target=8 \
    --sampling_temp=0.1 \
    --fixed_positions_jsonl=./fixed_positions.jsonl \
    --omit_AAs='C'  # 可选：避免半胱氨酸（防止二硫键假象）
```

`fixed_positions.jsonl` 文件格式：

```json
{"design_0": [[10, 11, 12], [45, 46, 47]]}
```

这表示 design_0 的第 10-12 和 45-47 号残基保持不变。

---

## 5. AlphaFold3：验证折叠

ProteinMPNN 设计了序列之后，关键问题是：**这些序列真的能折叠成我们设计的结构吗？**

AlphaFold3 是 DeepMind 2024 年发表在 Nature 上的蛋白结构预测终极工具。我们用 AF3 来预测设计序列的三维结构，然后与 RFdiffusion 生成的目标骨架对比。

> ⚠️ **AlphaFold3 安装警告**：官方版基于 JAX，依赖复杂且需要 TPU 或高显存 GPU（≥24GB）。对于个人用户，推荐使用社区维护的 PyTorch 版。

### 5.1 安装 AlphaFold3（PyTorch 社区版）

```bash
# 创建独立环境
conda create -n af3 python=3.11 -y
conda activate af3

# 安装依赖
pip install alphafold3-pytorch einops biopython

# 下载预训练权重（约 1.5GB）
# 注意：权重需要从 HuggingFace 下载
pip install huggingface-hub
huggingface-cli login  # 可选：如果下载公共权重不需要 token
```

### 5.2 准备输入序列

将 ProteinMPNN 输出的 FASTA 序列转为 AlphaFold3 的输入格式：

```bash
# 从 ProteinMPNN 输出中提取序列
# 创建一个简单的输入 JSON
cat > ~/af3_input.json << 'EOF'
{
  "name": "design_0_best",
  "sequences": [
    {
      "protein": {
        "id": ["A"],
        "sequence": "MKLKKLIAYVPAVVALEVGRYARLLKEGLHPDFYIELGD"
      }
    }
  ]
}
EOF
```

> 注意：上面的序列只是示例，请用你的 ProteinMPNN 实际输出的序列替换。

### 5.3 运行结构预测

```bash
# AlphaFold3 推理（不同版本命令可能有差异）
alphafold3-pytorch \
    --input_json ~/af3_input.json \
    --output_dir ~/af3_outputs \
    --model_dir ~/.cache/alphafold3 \
    --num_recycles=3
```

参数说明：
- `num_recycles`：循环次数，3-4 次即可，更多不一定更好
- `model_dir`：预训练权重存放位置

运行时间：一个 100 残基的蛋白约 10-30 分钟（A100 GPU）。

### 5.4 评估结果

预测完成后，AlphaFold3 会输出：
- `predicted.pdb`：预测的 3D 结构
- `confidence.json`：各位置的置信度分数（pLDDT）
- `pae.json`：预测对齐误差（PAE）矩阵

**关键指标**：

| 指标 | 合格 | 良好 | 优秀 | 含义 |
|------|------|------|------|------|
| **pLDDT** > 70 | ✅ | ✅ | ✅ | 整体折叠可靠度 |
| **pLDDT** > 90 | — | — | ✅ | 高置信度区域 |
| **pAE** < 5 | ✅ | ✅ | ✅ | 域间相对位置可靠 |
| **pTM** > 0.5 | ✅ | ✅ | ✅ | 整体拓扑合理 |
| **pTM** > 0.8 | — | — | ✅ | 预测结构非常可靠 |

### 5.5 RMSD：量化与目标骨架的差异

用一个简单的 Python 脚本来比较预测结构和目标骨架：

```bash
cat > ~/calc_rmsd.py << 'PYEOF'
import numpy as np
from Bio.PDB import PDBParser

def calc_rmsd(pdb1, pdb2):
    parser = PDBParser(QUIET=True)
    s1 = parser.get_structure("ref", pdb1)
    s2 = parser.get_structure("pred", pdb2)
    
    atoms1 = [a for a in s1.get_atoms() if a.get_name() == "CA"]
    atoms2 = [a for a in s2.get_atoms() if a.get_name() == "CA"]
    
    if len(atoms1) != len(atoms2):
        print(f"残基数不匹配: {len(atoms1)} vs {len(atoms2)}")
        return None
    
    coords1 = np.array([a.get_coord() for a in atoms1])
    coords2 = np.array([a.get_coord() for a in atoms2])
    
    # 计算 RMSD（需要先对齐）
    from Bio.SVDSuperimposer import SVDSuperimposer
    sup = SVDSuperimposer()
    sup.set(coords1, coords2)
    sup.run()
    rmsd = sup.get_rmsd()
    
    print(f"CA RMSD: {rmsd:.3f} Å")
    print(f"旋转矩阵/平移向量已应用")
    
    # 输出对齐后的结构
    io = PDBIO()
    io.set_structure(s2)
    io.save("aligned_" + pdb2.split("/")[-1])
    
    return rmsd

if __name__ == "__main__":
    import sys
    calc_rmsd(sys.argv[1], sys.argv[2])
PYEOF

python ~/calc_rmsd.py \
    ~/RFdiffusion/outputs/unconditional/pdb/design_0.pdb \
    ~/af3_outputs/predicted.pdb
```

**RMSD 解读**：
- **< 1.0 Å**：几乎完美匹配
- **1.0 - 2.0 Å**：非常好的设计
- **2.0 - 3.0 Å**：合格，可能需要优化
- **> 3.0 Å**：折叠方式与设计不符，需要调整

### 安装难度速查

下表总结了整个流程中涉及的安装难度：

| 工具 | 安装方式 | 硬件要求 | 安装时间 | 小白友好度 |
|------|---------|---------|---------|----------|
| **conda/mamba** | `bash Miniconda-*.sh` | CPU | 5 分钟 | ⭐⭐⭐⭐⭐ |
| **CUDA Toolkit** | `conda install -c nvidia` | NVIDIA GPU | 10 分钟 | ⭐⭐⭐⭐ |
| **RFdiffusion** | git clone + pip install | GPU ≥12GB | 20 分钟 | ⭐⭐⭐ |
| **ProteinMPNN** | git clone（即用） | 任意 | 5 分钟 | ⭐⭐⭐⭐⭐ |
| **AlphaFold3** | pip install | GPU ≥24GB | 30 分钟 | ⭐⭐⭐ |
| **ColabFold** | 浏览器 | 云端 | 0 分钟 | ⭐⭐⭐⭐⭐（零安装） |

---

## 6. 迭代优化：从失败到成功

一次跑通就得到完美设计的概率很低。蛋白设计本质上是一个**迭代优化**的过程。

### 6.1 常见的失败模式

当你发现 AF3 预测的结构和 RFdiffusion 设计的骨架差异很大时（RMSD > 3 Å），通常有几种原因：

| 问题 | 现象 | 解决方案 |
|------|------|---------|
| **骨架太细长** | 生成 100+ 残基的长 α-螺旋束但无核心疏水区 | 缩短骨架长度，设计更多 β-折叠 |
| **packing 不足** | 核心疏水残基比例低 | 在 ProteinMPNN 中降低 `sampling_temp`（如 0.05） |
| **loop 区过长** | 无规卷曲区太多 | 在 RFdiffusion 中控制 loop 比例 |
| **对称性不足** | 寡聚体设计不成功 | 使用 RFdiffusion 的对称设计模式 |

### 6.2 参数调优指南

**RFdiffusion 参数调整**：

```bash
# 方案 A：生成更紧凑的骨架
python scripts/run_inference.py \
    inference.output_dir=./outputs/tighter \
    inference.model_directory_path=./models \
    inference.num_designs=20 \
    inference.design_startnum=0 \
    'potentials.guiding_potentials=["type:oligo, oligo:intra, weight_intra:1.5"]'

# 方案 B：增加扩散步数（更精细，更慢）
python scripts/run_inference.py \
    inference.output_dir=./outputs/fine \
    inference.model_directory_path=./models \
    inference.num_designs=10 \
    inference.diffusion_params.timesteps=100  # 默认 50
```

**ProteinMPNN 参数调整**：

```bash
# 更多候选序列 → 更大筛选空间
python protein_mpnn_run.py \
    --num_seq_per_target=48 \     # 默认 8，增加到 48
    --sampling_temp=0.1 \         # 保守模式（减少不合理序列）
    ...
```

**筛选策略**：

1. 用 ProteinMPNN 的 `score` 排序，选 top 10 的序列
2. 用 AlphaFold3 预测这 10 个序列的结构
3. 计算每个预测结构与目标骨架的 RMSD
4. 选 RMSD 最小的 1-2 个候选做进一步优化
5. 回到第 1 步：调整 RFdiffusion 参数，生成新的骨架

### 6.3 一个典型的工作流脚本

```bash
#!/bin/bash
# pipeline.sh — 从头设计的完整自动化流程

DATASET="my_design"
ENV="protein_design"

# Step 1: RFdiffusion 生成骨架
echo "=== Step 1: RFdiffusion ==="
conda run -n RFdiffusion python scripts/run_inference.py \
    inference.output_dir=./outputs/${DATASET}/rf_diff \
    inference.model_directory_path=./models \
    inference.num_designs=10

# Step 2: ProteinMPNN 设计序列
echo "=== Step 2: ProteinMPNN ==="
cp ./outputs/${DATASET}/rf_diff/pdb/*.pdb ~/ProteinMPNN/inputs/
conda run -n proteinmpnn python protein_mpnn_run.py \
    --jsonl_path=./parsed_pdbs.jsonl \
    --out_folder=./outputs/${DATASET}/mpnn \
    --num_seq_per_target=16 \
    --sampling_temp=0.1

# Step 3: 提取最佳序列
echo "=== Step 3: Extract best sequences ==="
python scripts/extract_best_seqs.py \
    --input ./outputs/${DATASET}/mpnn/seqs \
    --output ./outputs/${DATASET}/candidates.fa

# Step 4: AlphaFold3 验证（需要手动准备输入）
echo "=== Step 4: AlphaFold3 verification (manual) ==="
echo "请用 candidates.fa 中的序列依次运行 AF3"
```

---

## 7. 常见坑与解决方案

### 🕳️ 坑 1：CUDA 版本不匹配

**现象**：`RuntimeError: CUDA error: no kernel image is available for execution on the device`

**原因**：PyTorch 编译的 CUDA 版本与驱动支持的 CUDA 版本不匹配。

**解决**：
```bash
# 检查驱动支持的 CUDA
nvidia-smi | grep "CUDA Version"

# 安装匹配的 PyTorch
# 如果驱动是 CUDA 11.8，就装：
pip install torch --index-url https://download.pytorch.org/whl/cu118

# 或者用 conda 避免版本冲突
conda install pytorch pytorch-cuda=11.8 -c pytorch -c nvidia
```

### 🕳️ 坑 2：显存不足（OOM）

**现象**：`CUDA out of memory`

**原因**：生成的骨架太长或 batch size 太大。

**解决**：
```bash
# 减小 batch size（每次处理更少的骨架）
python scripts/run_inference.py \
    ... \
    inference.num_designs=5  # 从 10 减到 5

# 如果还是 OOM，还可以减少生成步数（但会降低质量）
inference.diffusion_params.timesteps=25  # 默认 50
```

### 🕳️ 坑 3：GitHub 下载超时（中国用户）

**现象**：`curl: (28) Operation timed out`

**原因**：从 GitHub 下载大文件在中国大陆很慢。

**解决**：
```bash
# 使用国内镜像
# ghproxy 是一个 GitHub 下载加速代理
git clone https://ghproxy.net/https://github.com/RosettaCommons/RFdiffusion.git

# 或者：使用校园网镜像/北交大镜像
# 或者在浏览器中手动下载后传到服务器
```

### 🕳️ 坑 4：ProteinMPNN 找不到 parsed_pdbs.jsonl

**现象**：`FileNotFoundError: parsed_pdbs.jsonl`

**原因**：忘记先运行 parse_multiple_chains.py。

**解决**：
```bash
python helper_scripts/parse_multiple_chains.py \
    --input_path=./inputs \
    --output_path=./parsed_pdbs.jsonl
```

### 🕳️ 坑 5：AF3 预测结果完全不像

**现象**：预测结构与设计骨架的 RMSD > 5 Å

**原因**：可能是设计的序列本身就不合理（或者 AF3 参数没调好）。

**解决**：
1. 检查 ProteinMPNN 的 `score`——如果 > 1.0，说明序列质量差
2. 降低 `sampling_temp` 到 0.05 重新设计
3. 尝试用不同的骨架（RFdiffusion 生成 20-50 个，选看起来最合理的）
4. 如果是大蛋白（>200 残基），尝试分段设计

### 🕳️ 坑 6：conda 环境冲突

**现象**：`Solving environment: failed with initial frozen solve`

**原因**：不同工具对 Python/PyTorch 版本要求不同。

**解决**：强烈建议 **每个工具使用独立的 conda 环境**：
```bash
conda create -n RFdiffusion python=3.10  # RFdiffusion 专用
conda create -n proteinmpnn python=3.10  # ProteinMPNN 专用
conda create -n af3 python=3.11         # AlphaFold3 专用

# 切换环境
conda activate RFdiffusion
# ... 跑完 RFdiffusion ...
conda deactivate

conda activate proteinmpnn
# ... 跑 ProteinMPNN ...
```

---

## 总结：你已经走完了从头蛋白设计的全流程

恭喜！你现在掌握了：

1. ✅ **SSH + tmux** — 远程工作不被中断
2. ✅ **conda/mamba** — 环境管理不出错
3. ✅ **RFdiffusion** — 从无到有生成蛋白骨架
4. ✅ **ProteinMPNN** — 为骨架填充氨基酸序列
5. ✅ **AlphaFold3** — 验证设计是否合理
6. ✅ **迭代优化** — 从失败中改进

这个流程是目前学术界和工业界最主流的从头蛋白设计范式。掌握了它，你就拥有了"设计自然界中不存在的新蛋白"的能力。

下一步可以做什么？
- 尝试 **BindCraft**（一站式结合剂设计，集成了 RFdiffusion + ProteinMPNN）
- 用 **LigandMPNN** 设计酶的小分子结合口袋
- 用 **ESMFold** 做大规模筛选（比 AF3 快 60 倍）
- 学习 **ThermoMPNN** 预测突变对稳定性的影响

### 参考文献

| 工具 | 论文 | 期刊 | 年份 |
|------|------|------|------|
| **RFdiffusion** | Broadly applicable and accurate protein design... | Nature | 2023 |
| **ProteinMPNN** | Robust deep learning-based protein sequence design... | Science | 2022 |
| **AlphaFold3** | Accurate structure prediction of biomolecular interactions | Nature | 2024 |
| **BindCraft** | One-shot design of functional protein binders | Nature Biotech | 2024 |
| **LigandMPNN** | A general deep learning model for small molecule... | Science | 2024 |
| **ESMFold** | Language models of protein sequences... | Science | 2022 |
| **ColabFold** | Accelerating protein structure prediction... | Nature Methods | 2022 |
