# 新机器安装教程 (Install on a New Machine)

## 0. 重要：先提交当前机器的代码修复（在原机器上执行）

仓库里有两个 bug 修复**还没有提交**，如果不先推送，新机器 clone 后会在 `lerobot-calibrate` / `lerobot-teleoperate` 时报同样的错误：

```
ModuleNotFoundError: No module named 'lerobot.robots.so_follower'
```

在原机器上执行：

```bash
cd ~/lerobot-so101-bimanual
git add lerobot/src/lerobot/robots/bi_so101_follower/bi_so101_follower.py \
        lerobot/src/lerobot/teleoperators/bi_so101_leader/bi_so101_leader.py
git commit -m "Fix so_follower/so_leader import paths to so101_follower/so101_leader"
git push
```

> 验证是否已推送：`git status` 无输出即为干净；`git log --oneline -1` 能看到该提交。

---

## 1. 判断新机器 GPU（决定要不要装 cu128 PyTorch）

```bash
nvidia-smi
```

| GPU 型号 | 架构 (sm) | 需要 cu128 torch? |
|---|---|---|
| RTX 50 系列（5060 Ti / 5070 / 5080 / 5090） | Blackwell, sm_120 | ✅ **需要** |
| RTX 30 / 40 系列 | Ampere / Ada, sm_86 / sm_89 / sm_90 | ❌ 跳过（用默认 torch） |
| 无 NVIDIA GPU | — | ❌ 跳过（训练用 `--policy.device=cpu`） |

---

## 2. 创建环境

```bash
conda create -y -n lerobot python=3.10
conda activate lerobot
conda install ffmpeg -c conda-forge
```

## 3. 安装 LeRobot

```bash
git clone https://github.com/OneRobotAI/lerobot_bimanul.git
cd lerobot_bimanul/lerobot
pip install -e .
pip install -e ".[feetech]"
```

## 4. （仅 RTX 50 系）重装 cu128 PyTorch

默认安装的 torch（cu126）在 50 系 GPU 上会报：
`RuntimeError: CUDA error: no kernel image is available for execution on the device`

```bash
pip install --force-reinstall torch==2.7.1+cu128 torchvision==0.22.1+cu128 --index-url https://download.pytorch.org/whl/cu128
```

> 国内网络不稳时，用阿里云镜像并绕过代理：
>
> ```bash
> env -u http_proxy -u https_proxy -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY -u all_proxy \
> pip install --force-reinstall torch==2.7.1+cu128 torchvision==0.22.1+cu128 \
>   --index-url https://mirrors.aliyun.com/pytorch-wheels/cu128
> ```

## 5. （仅执行过步骤 4）修复 fsspec 冲突

重装 torch 会把 `fsspec` 升到 2026.4.0，与 `datasets 4.1.1`（要求 `<=2025.9.0`）冲突：

```bash
pip install "fsspec[http]==2025.9.0"
```

## 6. 摄像头画面显示（推荐）

`--display_data=true` 需要 Rerun viewer：

```bash
pip install rerun-sdk
```

## 7. 验证安装

```bash
# GPU 验证（RTX 50 系预期: 2.7.1+cu128 + 数字，无 sm_120 警告）
python -c "import torch; print(torch.__version__); x = torch.randn(1000, 1000, device='cuda'); print((x @ x).sum().item())"

# lerobot 导入验证
python -c "import lerobot.policies; print('lerobot OK')"

# fsspec 版本验证（应显示 2025.9.0）
python -c "import fsspec; print(fsspec.__version__)"

# CLI 验证
lerobot-calibrate --help
```

---

## 速查表

| 场景 | cu128 torch（步骤4） | fsspec 修复（步骤5） | rerun-sdk（步骤6） |
|---|---|---|---|
| RTX 50 系 | ✅ 需要 | ✅ 需要 | 推荐 |
| RTX 30/40 系 | ❌ 跳过 | ❌ 跳过 | 推荐 |
| 无 NVIDIA GPU | ❌ 跳过 | ❌ 跳过 | 推荐 |
| 纯 CPU 训练 | ❌ 跳过 | ❌ 跳过 | 推荐 |

> fsspec 冲突是 torch 重装的**副作用**：不重装 torch 就不会升级 fsspec，也就不需要修复。
