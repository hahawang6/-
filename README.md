# 蛙一下去水印项目入门教程

## 数据下载与解压

数据下载链接：

https://huggingface.co/datasets/WendellH/Watermark_dataset_LOGO-Full_Multi/tree/main
打开后选择 LOGO-Multi.ra
下载 LOGO-Multi，用户可以根据需要下载或者补充其它开源数据集

解压到

```text
D:\AIProjects\watermark-removal\data\raw
```

（可按自己需求调整目录结构，以下不提及）

## 模型处理流程

```text
带水印图片 → 共享编码器 → SMR定位与MBE背景恢复 → 粗恢复图 → 跨阶段特征连接＋3个CFF细化模块 → 最终恢复图
```

## 创建项目目录

Windowspowershell中执行：

设置项目根目录：

```powershell
$projectRoot = "D:\AIProjects\watermark-removal"
```

```powershell
$folders = @(
    "configs",
    "data\archives",
    "data\raw\LOGO-Multi",
    "data\manifests",
    "data\splits",
    "data\reports",
    "src\models",
    "src\data",
    "src\training",
    "src\inference",
    "src\evaluation",
    "scripts",
    "checkpoints\pretrained",
    "checkpoints\runs",
    "logs",
    "reports",
    "examples\inputs",
    "outputs\images",
    "outputs\masks",
    "app",
    "tests"
)
```

目录说明：

| 目录 | 说明 |
| --- | --- |
| `configs` | 存放配置 |
| `data\archives` | 存放下载的数据压缩包 |
| `data\raw\LOGO-Multi` | 存放有效的可训练数据 |
| `data\manifests` | 存放数据清单 |
| `data\splits` | 存放训练集验证集和测试集的划分 |
| `data\reports` | 存放数据检查报告 |
| `src\models` | 存放模型结构代码 |
| `src\data` | 存放数据预处理等操作的代码 |
| `src\training` | 存放训练代码 |
| `src\inference` | 存放推理代码，模型处理新图片 |
| `src\evaluation` | 存放效果评估代码 |
| `scripts` | 存放辅助脚本 |
| `checkpoints\pretrained` | 存放预训练的模型权重 |
| `checkpoints\runs` | 存放模型检查点 |
| `logs` | 存放运行和训练的日志 |
| `reports` | 存放项目整体的实验报告、评估结果 |
| `examples\inputs` | 存放用于演示后的照片 |
| `outputs\images` | 存放处理后的图片 |
| `outputs\masks` | 存放掩码图片，标记水印地区 |
| `app` | 存放应用界面代码 |
| `tests` | 存放测试代码 |

创建根目录结构：

```powershell
New-Item -ItemType Directory -Path $projectRoot -Force | Out-Null

foreach ($folder in $folders) {
    New-Item -ItemType Directory `
        -Path (Join-Path $projectRoot $folder) `
        -Force | Out-Null
}
```

把 PowerShell 当前所在的工作目录切换到项目根目录：

```powershell
Set-Location $projectRoot
```

展示文件夹：

```powershell
tree /A
```

## 数据划分

原始下载数据存在划分问题，原始划分确实存在背景泄漏，需要调整后再训练。

| 用途 | 样本数 | 划分方式 |
| --- | --- | --- |
| 训练集 | 10,389 | 原 trainmulti 排除重复背景后，再划出验证集 |
| 验证集 | 1,000 | 从剩余训练来源中确定性抽取，固定不变 |
| 测试集 | 2,000 | 保留原 valmulti |
| 排除训练 | 611 | 与测试集背景重复的 trainmulti 样本 |

下载资料包中的logo-multi-splits-v1.zip到

```text
D:\AIProjects\watermark-removal
```

```powershell
cd "D:\AIProjects\watermark-removal"

Expand-Archive -LiteralPath ".\logo-multi-splits-v1.zip" -DestinationPath "." -Force
```

清单将位于data\splits\logo_multi_v1\

包含 train.csv、val.csv、test.csv、excluded_train.csv 和划分规则说明。后续数据加载器直接读取这些清单，原始数据保持原样。

## 模型与训练代码

| 项目 | 处理规则 |
| --- | --- |
| 输入与恢复目标 | image → natural，三通道 RGB |
| 定位监督 | mask / 255，不把它当作透明度 α |
| 几何增强 | 输入、背景、mask 同步变换 |
| mask 缩放 | 最近邻插值，保持离散标签 |
| 数据划分 | 固定使用已生成的三个清单 |
| 异常数据 | 加载时检查透明通道和标签值，发现异常明确报错，不静默转换 |
| 推理 | 只输入图片，不需要真实 mask 或背景 |

下载压缩包 `slbr-fourier-project-v1.zip`。

并解压

执行：

解压到：`D:\AIProjects\watermark-removal`。

```powershell
Expand-Archive -LiteralPath "D:\microdownload\slbr-fourier-project-v1.zip" -DestinationPath "D:\AIProjects\watermark-removal" -Force
```

```powershell
cd "D:\AIProjects\watermark-removal"
```

## 安装依赖，获取固定源码和权重

```powershell
& ".\.venv\Scripts\python.exe" -m pip install torch==2.5.1 torchvision==0.20.1 --index-url https://download.pytorch.org/whl/cu124

& ".\.venv\Scripts\python.exe" -m pip install -r requirements.txt

& ".\.venv\Scripts\python.exe" scripts/fetch_reference.py

& ".\.venv\Scripts\python.exe" scripts/fetch_weights.py
```

## 精度修复与运行检查

经检验有数据溢出问题，发现问题后修复补丁

下载slbr-precision-fix-v2.zip保存到 D:\microdownload执行

```powershell
Expand-Archive -LiteralPath "D:\microdownload\slbr-precision-fix-v2.zip" -DestinationPath "D:\AIProjects\watermark-removal" -Force

cd "D:\AIProjects\watermark-removal"

& ".\.venv\Scripts\python.exe" -m wmr.preflight `
    --device cuda `
    --size 256 `
    --batch 1 `
    --data-root data/raw/LOGO-Multi `
    --base-weights weights/slbr-clwd-official.pth.tar `
--output runs/preflight-gpu-v2.json
```

## 正式训练与恢复训练

下载：slbr-formal-training-v1.zip

此包启用正式配置并更新训练入口，保留完整 SLBR＋Fourier 和已通过的精度修复

保存到 D:\microdownload，执行解压：

```powershell
Expand-Archive -LiteralPath "D:\microdownload\slbr-formal-training-v1.zip" -DestinationPath "D:\AIProjects\watermark-removal" -Force

cd "D:\AIProjects\watermark-removal"
```

开始训练：

```powershell
& ".\.venv\Scripts\python.exe" -u -m wmr.train `
    --config configs/train-formal.json `
    --base-weights weights/slbr-clwd-official.pth.tar
```

暂停按 Ctrl+C。 恢复时执行：

```powershell
& ".\.venv\Scripts\python.exe" -u -m wmr.train `
    --config configs/train-formal.json `
    --resume runs/slbr-fourier-v1/last.pt
```

## 服务器部署教程

frog-watermark-app.zip保存到：

```text
D:\microdownload\frog-watermark-app.zip
```

### 连接服务器

在 Windows PowerShell 逐条执行，固定 IP：


```powershell
$serverIp = "你的公网ip"
```

使用用户连接：

```powershell
ssh "ubuntu@$serverIp"
```

出现下面这种提示时，输入服务器密码：

```text
ubuntu@你的公网ip's password:
```

（也可以在你所购买的平台的控制台直接进行以下操作）

### 上传程序和权重

当前服务器窗口执行：

```bash
mkdir -p /home/ubuntu/frog-upload
```

另外打开一个 Windows PowerShell 窗口，不要关闭当前服务器窗口。

新窗口的提示符应该以 PS 开头。在新窗口执行：

原注释：“解压文件到服务器的文件夹中”。此处 `scp` 命令执行的是上传；解压操作见后面的 `unzip` 命令。

```powershell
scp "D:\microdownload\frog-watermark-app.zip" `
    "D:\AIProjects\watermark-removal\runs\slbr-fourier-v1\best-weights.pt" `
    "ubuntu@你的公网ip:/home/ubuntu/frog-upload/"
```

Ubuntu 服务器窗口执行

### 1．安装基础工具

更新系统软件包索引：

```bash
sudo apt-get update
```

安装三个工具：

```bash
sudo apt-get install -y python3-venv unzip curl
```

### 2．解压程序并放好权重

创建项目目录：

```bash
sudo mkdir -p /opt/frog-watermark-app
```

解压到部署目录：

```bash
sudo unzip -o /home/ubuntu/frog-upload/frog-watermark-app.zip \
  -d /opt/frog-watermark-app
```

修改文件所有者：

```bash
sudo chown -R ubuntu:ubuntu /opt/frog-watermark-app
```

```bash
cd /opt/frog-watermark-app

mkdir -p weights
```

复制文件到：`weights/best-weights.pt`。

```bash
cp /home/ubuntu/frog-upload/best-weights.pt \
  weights/best-weights.pt
```

### 3．创建环境、安装依赖

```bash
python3 -m venv .venv
```

安装 CPU 版 PyTorch，不需要安装 CUDA：

```bash
.venv/bin/python -m pip install --no-cache-dir \
  torch==2.5.1+cpu \
  --index-url https://download.pytorch.org/whl/cpu
```

完成后再执行：

```bash
.venv/bin/python -m pip install --no-cache-dir \
  -r requirements-web.txt
```

下载期间没有返回命令提示符时，先等待，不要重复执行。

### 4．启动真实模型服务

腾讯云控制台中添加防火墙

| 项目 | 填写 |
| --- | --- |
| 协议 | TCP |
| 端口 | 8081 |
| 来源 | 0.0.0.0/0 |
| 策略 | 允许 |

使用虚拟环境中的python，运行wmr中的web模块

```bash
.venv/bin/python -m wmr.web \
  --weights weights/best-weights.pt \
  --device cpu \
  --host 0.0.0.0 \
  --port 8081 \
  --cpu-threads 4
```


### 配置后台任务：

```bash
sudo tee /etc/systemd/system/frog-watermark.service > /dev/null <<'EOF'
[Unit]
Description=Frog Watermark Removal
After=network.target

[Service]
Type=simple
User=ubuntu
Group=ubuntu
WorkingDirectory=/opt/frog-watermark-app
Environment=PYTHONUNBUFFERED=1
ExecStart=/opt/frog-watermark-app/.venv/bin/python -m wmr.web --weights /opt/frog-watermark-app/weights/best-weights.pt --device cpu --host 0.0.0.0 --port 8081 --cpu-threads 4
Restart=on-failure
RestartSec=5
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
```

启动并设置开机自动运行：

```bash
sudo systemctl daemon-reload
sudo systemctl enable frog-watermark
sudo systemctl restart frog-watermark
```

检查是否真正启动成功：

```bash
curl --fail --retry 10 --retry-connrefused --retry-delay 2 \
  http://127.0.0.1:8081/api/health
```

输出包含 "ready":true 后，就可以关闭控制台

复现从来不是把命令敲完，而是在一次次尝试中，把陌生的知识变成自己的理解。也许你会遇到报错，会等上一段漫长的训练，会发现结果没有想象中理想——这些都不是白走的路。每定位一个问题、弄懂一个原理，你就比昨天更接近独立完成项目的自己。

不用急着一步到位。慢慢学，认真做，让每一次小小的进展，成为下一次出发的底气。愿你最终收获的不只是一份能运行的代码，还有面对新问题时，那句笃定的“我可以试试”。

如果这份教程曾帮你少绕一点弯路，或让你在卡住时找到继续走下去的方向，欢迎留下一个小星星。你的鼓励，也会成为这份分享继续生长的动力。
