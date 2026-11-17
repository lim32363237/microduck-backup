# 环境搭建（本服务器专用备忘）

本机 = 36.103.234.29（RTX 4090，x86_64，国内网络）。记录这台机器上重建 `uv` 环境的非标准步骤。标准流程见 README / AGENTS.md，这里只记**这台机器特有的坑和对策**。

## 网络约束（根因）

- 本机 **连不上 `github.com:443`**（路由被卡，DNS 正常），但 `pypi.org`、`astral.sh`、`raw.githubusercontent.com` 都通。
- apt 用清华镜像，属国内网络环境。
- 后果：任何 `git+https://github.com/...` 依赖都会让 `uv sync` 卡超时。

## 两处为此做的改动（已提交在工作副本）

### 1. BAM 依赖改用本地 vendored 副本

`pyproject.toml` 的 `[tool.uv.sources]`：原本
`better-actuator-models = { git = "https://github.com/Rhoban/bam.git", branch = "mjlab_frictionloss" }`
改成本地路径：
`better-actuator-models = { path = "/home/ubuntu/microduck/vendor/bam" }`

**bam 源码在 `/home/ubuntu/microduck/vendor/bam`（仓库外、未被 git 跟踪）。**

换机 / 重建时，若 `vendor/bam` 不存在，需从能连 GitHub 的机器（如本地 Mac）取回：
```bash
# 在能连 github 的机器上：
git clone -b mjlab_frictionloss https://github.com/Rhoban/bam.git
cd bam && rm -rf .git && cd ..
tar czf bam.tar.gz bam
scp bam.tar.gz ubuntu@<本机IP>:/home/ubuntu/microduck/vendor/
# 本机上：
cd /home/ubuntu/microduck/vendor && tar xzf bam.tar.gz
```

**版本注意**：`mjlab_frictionloss` 最新 commit 用的是电阻电压降模型
（`vin_drop_resistance_range`），比仓库原本锁定的旧 commit `62bd8ce`（增益模型
`vin_drop_gain_range`）新。`microduck_constants.py` 已按 `R = gain × kt`
（xl330 kt=0.366）换算，把 `vin_drop_gain_range=(0.0,0.2)` 改为
`vin_drop_resistance_range=(0.0,0.073)`。若换回旧 commit 的 bam，需把这行改回去。

### 2. wandb 新版兼容（Patch 6，无需手改 .venv）

mjlab 要求 `wandb>=0.22.3`，装到的是 0.31.0，它移除了
`wandb.Settings(start_method=...)`，而 rsl_rl 5.0.1 仍在传 → 训练 init 崩。
**已在 `src/mjlab_microduck/tasks/mdp.py` 加 Patch 6**：导入期包裹
`wandb.Settings` 丢弃被移除的 kwarg。这是持久修复（`uv sync` 不会覆盖，HF Jobs 也执行），
**不要再手改 `.venv` 里的 `rsl_rl/utils/wandb_utils.py`**。

## 重建命令（本机）

```bash
export PATH="$HOME/.local/bin:$PATH"
# 清华 PyPI 镜像，否则境外 wheel 极慢（torch 等 ~2GB）
export UV_DEFAULT_INDEX="https://pypi.tuna.tsinghua.edu.cn/simple"
export UV_HTTP_TIMEOUT=600
cd /home/ubuntu/microduck/work/rl_thesis/microduck_rl
uv sync      # 确认 vendor/bam 存在；torch 应为 2.9.1+cu128，CUDA 可用
```

冒烟测试（起正式训练前必跑）：
```bash
export WANDB_MODE=offline
uv run train Mjlab-GroundPick-Flat-MicroDuck --env.scene.num-envs 64 --agent.max_iterations 5
```
