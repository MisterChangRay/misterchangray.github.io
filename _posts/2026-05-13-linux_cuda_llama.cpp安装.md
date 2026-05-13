---
layout: post
title:  "linux大模型cuda安装"
date:   2016-05-13 10:29:20 +0800
categories:
      - 大模型
      - cuda
tags:
      - llama.cpp
---

我之前一直使用的是5070显卡，显存12g, 只能用9B模型。 想想用个16G的显卡会不会好点。

于是购入了5060TI 16G

### 工作站参数如下：

cpu 22核44线程
gpu 5060ti 16g
内存 64g
系统 ubuntu 24.04.1-Ubuntu

### 安装过程

1. 首先是插入显卡
2. 准备安装驱动，我这里是重新安装了系统。之前是系统是安装好的，安装驱动莫名其妙的问题，千兆网卡也只有百兆，搞不懂。重装系统就好
3. 开始安装cuda,
   这一路直接参考官网文档安装就行[cuda官方地址](https://developer.nvidia.com/cuda-12-8-0-download-archive?target_os=Linux&target_arch=x86_64&Distribution=Ubuntu&target_version=24.04&target_type=deb_network)
   <img width="1391" height="679" alt="image" src="https://github.com/user-attachments/assets/a11a25e6-0dae-49e9-9c87-0bc6bbb7d965" />
   <img width="1386" height="868" alt="image" src="https://github.com/user-attachments/assets/6be4d929-68da-45e7-8136-465279dccc37" />
   按照流程安装就行
   
   这里备注一个坑，之前我安装的13.2的， 没想到不支持低精度的模型（主要是4bit以下的会回复乱码）。所以这里我安装的12.8的

5. 我这里使用的llama.cpp，安装流程如下
   在rease页面下载最新版本的源码就行,下载地址[llama.cpp](https://github.com/ggml-org/llama.cpp/releases)
   
   解压到目录开始编译，编译前确认有cmake
   
   进入目录执行 `cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_COMPILER=$(which nvcc)`, 如果本地有多版本的nvcc工具，使用` -DCMAKE_CUDA_COMPILER=$(which nvcc)`指定编译的版本
   
   接着执行 `cmake --build build --config Release -- -j8`,这里j8是使用8核进行并行编译，不然编译速度太慢了

安装流程就差不多完成了。接着就可以下载模型启动了。


### 实测结果

5060ti 在使用 qwen3.6-27b-q3的情况下，上下文能给到32k， 30t/s。 如果64就爆显存了，基本只有10t/s

还使用了一个模型（gemma-4-26B-A4B-it-GGUF.gguf），这里也是给32k上下文，结果只有10t/s

这玩意感觉有点难受，32k上下文能做啥。唉。 

看样子至少得32g显存才能干事，即使24G显存，估计q4模型上下文也只能开64k左右。

个人电脑部署确实也有点难受。

说实话，27bq3的模型还不错，用它写了两个应用，一个贪吃蛇，一个旋转魔方，一次性通过，也能玩

魔方截图
<img width="910" height="825" alt="image" src="https://github.com/user-attachments/assets/7a5c9b6d-555c-4c97-9368-396c96b5aa96" />

贪吃蛇截图
<img width="570" height="747" alt="image" src="https://github.com/user-attachments/assets/ea13fd82-083d-4797-9feb-0d79cf85c0d3" />





