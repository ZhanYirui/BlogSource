---
title: "MacOS系统M芯片利用AFL++进行模糊测试教程"
date: 2024-06-29
draft: false
description: "MacOS系统M芯片利用AFL++进行模糊测试教程"
tags: ["Fuzzing","AFL++"]
---
<u>**MacOS系统M芯片利用AFL++进行模糊测试教程**</u>



### 1.下载Docker

Docker下载链接：https://docs.docker.com/desktop/install/mac-install/

选择Docker Desktop for Mac with Apple sillicon

![截屏2024-06-30 18.18.25](https://p.ipic.vip/pzsphe.png)

下载完成后进行安装

安装后打开，点击右上角的设置按钮

![截屏2024-06-30 18.21.53](https://p.ipic.vip/6l8r0k.png)

点击Docker Engine，将下面的json内容替换掉原先的内容，完成换源后点击右下角的Apply&restart重启

```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "features": {
    "buildkit": true
  },
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "https://hub-mirror.c.163.com"
  ]
}
```

![截屏2024-06-30 18.23.06](https://p.ipic.vip/jc4cci.png)



### 2.克隆AFL++到本地

首先我们可以创建一个afl++目录用于存放AFL++项目，然后进入这个目录进行git clone

```bash
mkdir afl++
cd afl++
git clone https://github.com/AFLplusplus/AFLplusplus
```

完成上面三步，AFL++项目就被克隆到了本地



### 3.创建一个AFL++容器

打开终端，输入以下命令，将aflplusplus镜像拉到本地

```bash
docker pull aflplusplus/aflplusplus
```

然后，根据镜像创建一个AFL++容器

```bash
docker run -it aflplusplus/aflplusplus /bin/bash
```

这时候应该是在/AFLplusplus目录下，我们用make命令进行编译构建

```
make
```



### 4.将要测试的代码导入容器

{{< alert >}}

这里我将要测试的代码放到了我的github仓库里，然后利用git clone将代码文件克隆到容器内，直接从本地导入代码文件的方法自寻查找

{{< /alert >}}

由于现在是在/AFLplusplus目录下，我们先返回上一级目录

```bash
cd ..
```

然后利用git clone将代码文件克隆下来

```bash
git clone https://github.com/ZhanYirui/Fuzzing_Test_File.git
```

然后进入你存放代码的目录下

```bash
cd /Fuzzing_Test_File/test1
```

创建一个构建目录并进入

```bash
mkdir build
cd build
```

将AFL++工具添加到可执行文件的编译器中

```bash
CC=/AFLplusplus/afl-clang-fast CXX=/AFLplusplus/afl-clang-fast++ cmake ..
```

制作构建中的文件

```bash
make
```

我们需要一个种子目录，这个种子目录你可以根据代码精心构造，我这里是随机初始种子

```bash
cd ..
mkdir seeds
cd seeds
for i in {0..4}; do dd if=/dev/urandom of=seed_$i bs=64 count=10; done
cd ..
cd build
```

拥有种子目录后，就可以进行测试了，输入以下命令

```bash
/AFLplusplus/afl-fuzz -i [full path to your seeds directory] -o out -m none -d -- [full path to the executable]
```

其中`[full path to your seeds directory]`是种子目录的路径，`[full path to the executable]`是你build的可执行文件的路径，比如我这里就是：

```
/AFLplusplus/afl-fuzz -i /Fuzzing_Test_File/test1/seeds -o out -m none -d -- /Fuzzing_Test_File/test1/build/simple_crash
```

然后就可以看到AFL++在进行测试了

![截屏2024-06-30 18.48.18](https://p.ipic.vip/fvpssv.png)

可以通过`control + C`来退出fuzzing，并通过`out/default/crashes`目录来找到导致程序崩溃的输入



### 5.利用Sourcetail分析源代码

sourcetail是一个很好用的源码阅读工具

sourcetail下载链接：https://github.com/CoatiSoftware/Sourcetrail/releases

