原文链接：【[链接一](https://www.freedidi.com/11907.html)】【[链接二](https://www.freedidi.com/7749.html)】

**方案一：免费的修复平台，优点是即开即用，完全免费，缺点是速度有点慢，功能有限。**

1.CodeFormer 【[点击前往](https://huggingface.co/spaces/sczhou/CodeFormer)】

2.GFPGAN【[点击前往](https://huggingface.co/spaces/akhaliq/GFPGAN)】

**方案二：自建更强大的 AI照片、视频修复工具，依靠自身系统硬件，速度快，功能多效果好，可玩性高！_不仅修复照片，也可以（高清修复视频）_！**

github开源项目【[点击前往](https://github.com/sczhou/CodeFormer)】
使用该项目，需要依赖 cuda 和 pytorch，安装【[参考](https://developer.aliyun.com/article/1062107)】。

**依赖一：安装 conda**
conda 分为 anaconda（包含许多库的集合版本） 和 miniconda（精简版本：只包含 conda、pip、zlib、python 以及它们所需要的包）。
【[git工具](https://git-scm.com/)】
【[python 官网](https://www.python.org/downloads)】
【[miniconda 官网](https://docs.anaconda.com/free/miniconda)】
【[anaconda官网](https://www.anaconda.com/download)】


**依赖二：安装 cuda** 
1. 官网入口 【[点击前往](https://developer.nvidia.com/cuda-downloads)】
2. 查看版本的命令
`命令1：nvcc -V `
`命令2：nvidia-smi`

**依赖三：安装 pytorch**
1. 官网入口 【[点击前往](https://pytorch.org)】
2. 测试方法，进入 python 环境，执行以下测试 code，返回 True，即为成功；
`import torch`
`torch.cuda.is_available()`