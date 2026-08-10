# GPU server 使用

## 1. 开启客户端
✅

## 2. 配置 SSH

将下面内容加入本机 `~/.ssh/config`。Windows 路径是 `C:\Users\你的用户名\.ssh\config`。

```sshconfig
Host gpu-a
    HostName ip
    User ubuntu
    ServerAliveInterval 30
    ServerAliveCountMax 3
    ConnectTimeout 15

Host gpu-b
    HostName ip
    User ubuntu
    ServerAliveInterval 30
    ServerAliveCountMax 3
    ConnectTimeout 15
```

macOS 和 Linux 额外执行：

```bash
chmod 600 ~/.ssh/config
```

连接分配给你的容器：

```bash
ssh gpu-a
ssh gpu-b
```

首次连接输入 `yes` 接受主机指纹。密码输入时没有回显是正常的。

## 3. 运行任务

登录后在 `tmux` 中运行长任务。断开后重新 SSH，再执行 `tmux attach -t main` 即可回到原会话。

```bash
tmux new -As main
```

所有需要保留的代码、数据、模型和虚拟环境都放在 `~`，即 `/home/ubuntu`。容器重建时，`~` 保留；其他路径中的文件和手动安装的系统软件可能丢失。

```bash
mkdir -p ~/projects ~/data ~/venvs
python3 -m venv ~/venvs/project
source ~/venvs/project/bin/activate
```
```
## 4. 资源规则

两个容器共享同一组四张 A100 GPU、94 个逻辑 CPU 和 100 GiB RAM 池。运行前先看状态：

```bash
nvidia-smi
free -h
df -h ~
```

请明确选择 GPU：

```bash
CUDA_VISIBLE_DEVICES=1 python train.py
CUDA_VISIBLE_DEVICES=1,0 python train.py
```

建议按 `1 -> 0 -> 3 -> 2` 的顺序使用 GPU。GPU 2 目前有临时散热问题，使用它时监控温度，持续超过 85°C 就降低负载或停止任务：

```bash
watch -n 5 nvidia-smi --query-gpu=index,temperature.gpu,utilization.gpu,memory.used --format=csv
```

单容器内存上限是 94 GiB；两个容器合计 100 GiB。swap 只用于短暂尖峰，内存持续过高时进程可能被终止。GPU 显存没有容器隔离。

## 传文件

大文件使用可续传的 `rsync`：

```bash
rsync -avP --partial ./data/ gpu-a:~/data/
rsync -avP --partial gpu-a:~/results/ ./results/
```

小文件可用：

```bash
scp train.py gpu-a:~/
```

连接或传输中断后，重新运行相同的 SSH 或 rsync 命令即可；正在 tmux 中运行的任务会继续执行。
