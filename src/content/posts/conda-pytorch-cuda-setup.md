---
path: conda-pytorch-cuda-setup
title: Conda + PyTorch GPU 环境配置手记：清华源加速与 CUDA 验证
pubDate: 2026-03-20
categories: ['Conda', 'PyTorch', 'CUDA']
description: '记录一次在 Conda 环境中配置 PyTorch 与 CUDA 的完整过程，包含镜像源设置、环境创建与安装验证。'
---

最近在折腾新机器，又要配一遍深度学习环境。虽然这事干了无数次，但每次总有些小坑——镜像源没配好下载慢成狗，PyTorch 和 CUDA 版本对不上，或者装完发现 CUDA 不可用。这次干脆把步骤完整记下来，方便自己以后复制粘贴，也给有需要的朋友参考。

最终目标是：用 conda 创建一个独立环境，安装 PyTorch GPU 版本（CUDA 12.1），并且能用清华源加速下载，再装好项目依赖，最后验证 GPU 确实能用。

#### 1. 先收拾 conda 源（清华镜像加速）

conda 默认源在国外，速度看脸。我习惯先配好清华 tuna 源，顺便把 pytorch 和 nvidia 的频道也走镜像，会快很多。

**建议先备份原来的配置**，万一改错了还能救：

```bash
cp ~/.condarc ~/.condarc.bak   # 如果有的话
```

然后用 cat 直接覆盖写入新配置（注意是覆盖，如果之前有重要配置，请手动合并）：

```bash
cat > ~/.condarc <<'EOF'
channels:
  - pytorch
  - nvidia
  - defaults
show_channel_urls: true

default_channels:
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2

custom_channels:
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
EOF
```

简单解释一下：

- `channels` 里指定了 pytorch、nvidia 和 defaults，顺序会影响包的查找优先级。
- `default_channels` 是清华镜像对 defaults 频道的映射。
- `custom_channels` 则告诉 conda，pytorch 这个自定义频道应该去哪个镜像站找。这样安装 pytorch 相关包时就会走 tuna，不用手动加 `-c` 也能加速。

配置完后可以 `conda clean -i` 清理一下索引缓存，确保新源生效。

#### 2. 创建并激活独立环境

环境名就叫 `test` 吧，Python 指定 3.11（现在 PyTorch 对 3.11 支持很好了）：

```bash
conda create -n test python=3.11 -y
conda activate test
```

#### 3. 安装 PyTorch GPU 版本（CUDA 12.1）

这里我选择用 pip 安装，因为 PyTorch 官方 pip 包的更新往往比 conda 快，而且依赖更干净。当然，如果你习惯 conda 安装也可以，但要注意频道混用可能带来的冲突。

我们要装的是 CUDA 12.1 对应的 PyTorch，所以指定 index-url：

```bash
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
```

`cu121` 表示 PyTorch 预编译时依赖的 CUDA 运行时版本是 12.1。这并不意味着系统里必须完整安装 CUDA Toolkit，只需要 NVIDIA 驱动版本够新（一般建议驱动版本 ≥ 525.xx，支持 CUDA 12.x 即可）。

如果想装其他 CUDA 版本（比如 cu118、cu117），把 URL 里的数字改掉就行。可以去 [PyTorch 官网](https://pytorch.org) 获取对应命令。

#### 4. 安装项目依赖

如果项目里有一个 `requirements.txt`，现在就可以装了：

```bash
pip install -r requirements.txt
```

建议在激活的环境下执行，避免污染 base。如果有 conda 特有的包，也可以用 `conda install`，但最好保持 pip 和 conda 安装的包尽量不交叉，减少未来 dependency hell 的风险。

#### 5. 验证 PyTorch 能否调用 GPU

这一步最简单也最重要——装完一定要试一下，不然等跑训练时才发现没 GPU 就尴尬了。

```bash
python -c "import torch; print(torch.__version__); print(torch.cuda.is_available())"
```

正常会输出类似：

```
2.4.0+cu121
True
```

如果打印 `False`，说明 CUDA 不可用。常见原因有：

- 驱动没装好（`nvidia-smi` 看看）
- PyTorch 版本与驱动不兼容（太新的 PyTorch 可能需要更新的驱动）
- 安装成了 CPU 版本（检查一下 index-url 是不是真的 cu121）

#### 一些补充

- 这套流程在 Linux 和 Windows（PowerShell 下用类似命令）都适用，Windows 下 `.condarc` 文件一般在 `C:\Users\用户名\.condarc`。
- 清华源偶尔会同步延迟，如果遇到包找不到，可以临时切回官方源，或者等几小时再试。
- 如果想用 conda 直接安装 PyTorch GPU 版本（不走 pip），可以这样（但要注意 nvidia 频道也要配镜像）：
  ```bash
  conda install pytorch torchvision torchaudio pytorch-cuda=12.1 -c pytorch -c nvidia
  ```
  不过这样容易和 defaults 频道里的包冲突，我更喜欢 pip 方式，干净可控。
