---
layout:     post
title:      ESP32S3手写数字识别
subtitle:   2025寒假练-使用CrowPanel ESP32 Display 4.3英寸HMI开发板完成手写识别显示
date:       2025-03-08
author:     BY
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - ESP
---

# 项目介绍

本项目为2025寒假练活动ESP32 HMI板卡活动。要求实践时的要求如下：
- 使用LVGL进行人机交互编程
- 在屏幕上设定一正方形书写区域
- 在该区域里书写0-9的数字
- 对书写的数字进行识别，并将识别的数字传递给灯板，在灯板上进行显示

# 硬件说明

本项目所使用的开发板使用ESP32S3为核心，4.3寸触摸屏作为人机交互的媒介，可很方便的做一些物联网应用、智能家居中控等。甚至还有锂电池充放电管理可实现断电续航，有喇叭插口，只要再加上麦克风移植聊天机器人、大模型接入等可以发掘出更多的玩法。

另附送的灯板为两片74HC595级联驱动的64颗LED，可看作一个8*8分辨率的小屏幕，显示单个ASCII字符自然不再话下。

# 方案说明

- 因为官方有移植LVGL的教程，因此不再赘述。
- 第一步需要LVGL搭建屏幕书写区域，很巧的是LVGL的画布（canvas）组件就可以实现这个需求。
- 再就是如何对手写数字进行识别，比较有名的是mnist数据集，但他需要依托一些ml框架部署。在这方面的可选项就太多了，比如乐鑫的ESP-DL、tensorflow lite、tiny ml等。我因为之前有幸参与过开源项目[TinyMaix](https://github.com/sipeed/TinyMaix)的单片机适配测试，所以选择了我相对熟悉的这个框架。
- 灯板像素为8*8，但是8x8的字体不是很常见，我选用了OLED屏幕上常用的6x8大小字体。

# 软件说明

软件框图如下
![框图](/img/post_img/ESP32S3-CROW/框图.png)

## LVGL模拟器实践

因为工程中加入LVGL之后，变得很庞大，倘若每次稍微修改LVGL画面布局，就下载进单片机看看效果，那会浪费极大的时间在下载固件上，然而LVGL官方是有提供模拟器仿真方案来简化调试步骤的。但是官方给出的模拟器方案是基于Visual Studio的，但是VS太过庞大，且个人感觉使用起来没有那么方便，那么有没有其他的模拟器方案呢？答案是当然了！VS也只是调用了SDL做桌面显示，理论上来讲我们只需要适配好SDL调用，那么也能流畅使用任何IDE做LVGL模拟器。我本人比较喜欢用Clion编辑调试C/C++，也正好有看到这么一篇博客教程[在CLion上搭建LVGL模拟器](https://mdlzcool.github.io/post/90c7d419.html),于是就按照这个步骤搭建起来了。其中比较重要的步骤我放在下面记录下，以供后面忘了查阅。
```
git clone -b release/v8.3 --recursive https://github.com/lvgl/lv_port_pc_eclipse.git

[x86_64-13.2.0-release-posix-seh-ucrt-rt_v11-rev1.7z](https://github.com/niXman/mingw-builds-binaries/releases)
[SDL2-devel-2.30.8-mingw.tar.gz](https://github.com/libsdl-org/SDL/releases)

将SDL2/x86_64_w64-mingw32/include/SDL2复制到mingw64/x86_64_w64-mingw32/include文件夹中
将SDL2/x86_64_w64-mingw32/lib所有的文件复制到mingw64/x86_64_w64-mingw32/lib中

CLion配置工具链
工具集：D:\1my_program_study\LVGL_CLION\mingw64
构建工具：D:\1my_program_study\LVGL_CLION\mingw64\bin\mingw32-make.exe
GCC:D:\1my_program_study\LVGL_CLION\mingw64\bin\gcc.exe
G++:D:\1my_program_study\LVGL_CLION\mingw64\bin\g++.exe
调试器：MinGW-w64 GDB

SET(SDL2_DIR  D:/1my_program_study/LVGL_CLION/mingw64/x86_64-w64-mingw32/lib/cmake/SDL2)

```

下一步是写出我们的画布做交互，这里主要参考自百问网的资料[lv_lib_100ask](https://gitee.com/weidongshan/lv_lib_100ask.git)，其中有一个sketchpad的example，我基本是参考的这里的。其中需要注意的是画布的尺寸设置，当大于屏幕尺寸时会出现溢出，具体现象为画一个地方，会有多个地方出现线条。再就是加上两个按钮，分别是清屏和开始识别。这里有个问题是LVGL没有直接读取canvas内容的API，所以只能直接解析缓冲区（我目前只想到这个办法）。如下为读取手写区内容，并降采样使其尺寸符号后面的识别输入大小的处理。
``` c
// 平均池化降采样函数
void resize_array(lv_color_t* src, int src_width, int src_height, int dst_width, int dst_height)
{
    // 检查尺寸是否符合降采样比例
    if (src_width % dst_width != 0 || src_height % dst_height != 0) {
    printf("Error: Source dimensions must be multiples of destination dimensions.\n");
    return ;
    }

    // 计算子块大小
    int block_width = src_width / dst_width;
    int block_height = src_height / dst_height;

    // 分配目标数组内存
    uint8_t* dst = target_img;//(uint8_t*)calloc(dst_width * dst_height, sizeof(uint8_t));

    // 遍历每个子块
    for (int i = 0; i < dst_height; i++) {
        for (int j = 0; j < dst_width; j++) {
            int sum = 0;
            // 计算当前子块的平均值
            for (int bi = 0; bi < block_height; bi++) {
                for (int bj = 0; bj < block_width; bj++) {
                    int src_index = ((i * block_height + bi) * src_width) + (j * block_width + bj);
                    sum += src[src_index].ch.red;
                }
            }
            // 取平均值并截断到0-255
            dst[i * dst_width + j] = (uint8_t)(sum / (block_width * block_height));
        }
    }

    // return dst;
}
```

## TinyMaix载入

TinyMaix整体文件较少，移植比较简单，具体见官方仓库readme。整体API使用比较简单，只需要load，run即可拿到运行结果。但其中因为我画笔的颜色深度问题，导致我的输入图像颜色过浅，所以后面做了加倍的操作，最后算是大致做到了基本的数字识别。

## 点灯

直接使用`simsso/ShiftRegister74HC595`这个库控制595，然后就是数字编码与创建定时行刷新任务即可。

``` c
extern uint8_t volatile display_data;
static void LEDPanel_Task(void* parameter)
{	
  for(uint8_t i = 0; i < 10; i++)
  {
    for(uint8_t j = 0; j < 8; j++)
    {
      NEW_NUMBERS[i][j] = reverse_bits_fast(NEW_NUMBERS[i][j]>>1);
    }
  }
    while (1)
    {
        decode_nums_to_pannel(display_data);
        vTaskDelay(pdMS_TO_TICKS(3));   
    }
}
```

# 效果展示与遇到的问题

## 效果展示

![数字1](/img/post_img/ESP32S3-CROW/1.jpg)
![数字2](/img/post_img/ESP32S3-CROW/2.jpg)
![数字3](/img/post_img/ESP32S3-CROW/3.jpg)
![数字4](/img/post_img/ESP32S3-CROW/4.jpg)
![数字7](/img/post_img/ESP32S3-CROW/7.jpg)
完整识别见视频

## 遇到的问题

- canvas画布buf大小设置的问题，修改为小于屏幕区域解决
- 画笔颜色过浅，难以匹配模型数字，增大画笔区域权重
- 灯板显示数字反相，将每一行字模高低位取反


# 心得体会

- 这次直接用的TinyMaix开源的手写数字模型，没有自己去尝试训练优化，谁还不是个炼丹术士呢（狗头）
- 这个板子的潜力还很大，等待我继续挖掘.jpg，起码wifi、ble和音频我还没用起来
- 看到群里有大佬移植了小智，待活动结束后开源我也去学习下~~~