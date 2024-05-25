---
title: "macOS安装及卸载Miniconda教程"
date: 2023-11-13
draft: false
description: "macOS安装及卸载Miniconda教程"
tags: ["conda"]
---
**<u>macOS安装及卸载Miniconda教程</u>**



### 1 安装

1) 查看你要安装的版本

 打开[Miniconda版本仓库](https://repo.anaconda.com/miniconda/)，复制你要安装版本的名字，**注意是.sh后缀的文件名**。

{{< alert >}}

**不要点击链接下载！！！** 只复制文件的名字，例如Miniconda3-py39_24.4.0-0-MacOSX-arm64.sh。

{{< /alert >}}

![截屏2024-05-25 21.51.30](https://p.ipic.vip/5g4nzp.png)

2. 快速命令行安装

打开终端执行以下命令：

```bash
mkdir -p ~/miniconda3
curl https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-arm64.sh -o ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm -rf ~/miniconda3/miniconda.sh
```

{{< alert >}}

第二行命令中的Miniconda3-latest-MacOSX-arm64.sh适用于苹果M芯片的最新Miniconda版本。如果你是Intel芯片，或者想安装其他版本，请将你在步骤1)中复制的版本名替换上面第二行命令中的Miniconda3-latest-MacOSX-arm64.sh

{{< /alert >}}

安装后，初始化新安装的Miniconda，打开终端执行以下命令：

```bash
~/miniconda3/bin/conda init bash
~/miniconda3/bin/conda init zsh
```



### 2 conda配置国内镜像源

推荐配置中科大镜像源，其他镜像源可以自行搜索配置。

打开命令行执行以下命令：

```bash
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/conda-forge/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/msys2/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/bioconda/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/menpo/
conda config --set show_channel_urls yes
```

显示现有安装源：

```bash
conda config --show channels
```

恢复默认源：

```bash
conda config --remove-key channels
```

移除某个源：

```bash
conda config --remove channels https://mirrors.ustc.edu.cn/anaconda/cloud/menpo/
```



### 3 conda常用命令

| **功能**                                 | **命令**                                                     |
| :--------------------------------------- | :----------------------------------------------------------- |
| **获取版本号**                           | **conda -v**                                                 |
| **获取帮助**                             | **conda -h**                                                 |
| **创建环境**                             | **conda create -n** environment_name                         |
| **创建指定python版本下包含某些包的环境** | **conda create -n** environment_name **python=** 3.7 numpy scipy |
| **进入环境**                             | **conda activate** environment_name                          |
| **退出环境**                             | **conda deactivate**                                         |
| **删除环境**                             | **conda remove -n** environment_name **- -all**              |
| **列出所有环境**                         | **conda env list**                                           |
| **将旧环境复制到新环境**                 | **conda create - -name** new_env_name **- -clone** old_env_name |
| **安装包**                               | **conda instal** package_name                                |
| **查看当前环境包列表**                   | **conda list**                                               |
| **查看指定环境包列表**                   | **conda list -n** environment_name                           |
| **查看conda源中包的信息**                | **conda search** package_name                                |
| **更新包**                               | **conda update** package_name                                |
| **删除包**                               | **conda remove** package_name                                |
| **清理无用的安装包**                     | **conda clean -p**                                           |
| **清理tar包**                            | **conda clean -t**                                           |
| **清理所有安装包及cache**                | **conda clean -y - -all**                                    |



### 4 卸载

1) 使用Anaconda-Clean包删除所有与conda相关的文件和目录：

```bash
conda activate environment_name
conda install anaconda-clean
anaconda-clean
```

2) 删除整个目录

```bash
rm -rf ~/miniconda3
```

3. 删除PATH环境变量中的conda路径

{{< alert >}}

将# >>> conda initialize >>>和# <<< conda initialize <<<之间的所有内容删除

{{< /alert >}}

```bash
vim ~/.bash_profile
```

![截屏2024-05-25 22.24.52](https://p.ipic.vip/cr0f9l.png)

{{< alert >}}

进入vim后按i进入编辑模式，删除完后按:wq!进行保存并退出。保存后执行下面的命令。

{{< /alert >}}

```bash
source ~/.bashrc
```

同理，对zsh也执行上述操作。

```bash
vim ~/.bashrc
```

![截屏2024-05-24 20.00.56](https://p.ipic.vip/jsjyo2.png)

```bash
source ~/.bashrc
```

4. 删除配置文件

```bash
rm -rf ~/.condarc
```
