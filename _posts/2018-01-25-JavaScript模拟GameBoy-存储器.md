---
layout: post
title: JavaScript模拟GameBoy:存储器
description: "Translation of \"GameBoy Emulation in JavaScript\" series articles."
modified: 2018-01-25
tags: [GameBoy Emulation in JavaScript,Translation,JavaScript]
categories: []
---

这是《JavaScript模拟GameBoy》系列文章的第二篇。目前已经完成了前十篇，其余的我将会继续跟进。

1. [中央处理器]({{ site.owner.site }}/2018/01/22/JavaScript模拟GameBoy-CPU.html)
2. [存储器]({{ site.owner.site }}/2018/01/25/JavaScript模拟GameBoy-存储器.html)
3. [GPU时序]({{ site.owner.site }}/2018/01/28/JavaScript模拟GameBoy-GPU时序.html)
4. [图形]({{ site.owner.site }}/2018/01/30/JavaScript模拟GameBoy-图形.html)
5. [整合]({{ site.owner.site }}/2018/02/01/JavaScript模拟GameBoy-整合.html)
6. [输入]({{ site.owner.site }}/2018/02/02/JavaScript模拟GameBoy-输入.html)
7. [精灵]({{ site.owner.site }}/2018/02/05/JavaScript模拟GameBoy-精灵.html)
8. [中断]({{ site.owner.site }}/2018/02/06/JavaScript模拟GameBoy-中断.html)
9. [内存扩展]({{ site.owner.site }}/2018/02/07/JavaScript模拟GameBoy-内存扩展.html)
10. [定时器]({{ site.owner.site }}/2018/02/24/JavaScript模拟GameBoy-Memory-定时器.html)

本系列文章的模拟器代码可以在Github查看：[http://github.com/Two9A/jsGB](http://github.com/Two9A/jsGB)

---

在本系列的前一篇中，计算机被描述为从存储器中获取指令的处理单元。绝大部分情况下，一个计算机的存储器并不是一个简单的连续区域，GameBoy在这方面同样如此。由于GameBoy可以访问其地址总线上的65,536个独立位置，因此可以绘制出CPU可以访问的所有存储器区域的映射。
![GameBoy地址总线的存储器映射](http://imrannazar.com/content/img/jsgb-mmu-map.png)

图1: GameBoy地址总线的存储器映射

更详细的存储器区域如下所示：

* [0x0000-0x3FFF] 卡带ROM，存储区0：卡带存储器程序的前16384字节在存储器映射中始终可用，特殊情况下：
  * [0x0000-0x00FF] BIOS：当CPU启动时，PC（程序计数器）从0x0000即GameBoy BIOS代码的开始位置开始计数。BIOS运行后，将被从存储器映射中移除，然后这部分卡带ROM变为可寻址的。
  * [0x0100-0x014F] 卡带头部：卡带的这部分包含了卡带名称以及制造商的数据，必须按照特定格式书写。（更多关于卡带头部的细节可以阅读[The Cartridge Header](http://gbdev.gg8.se/wiki/articles/The_Cartridge_Header)）
* [0x4000-0x7FFF] 卡带ROM，其他存储区：卡带程序的每个16k存储区可以在这里被CPU逐一获取。卡带上的芯片通常用来在存储区间切换，使得特定的区域可以被访问到。最小的程序为32k，这意味着不需要存储区选择芯片。
* [0x8000-0x9FFF] 图形RAM：图形子系统使用的背景和sprites数据保存在这里，可以通过卡带程序进行更改。这个区域将在本系列的第3部分中进一步详细讨论。
* [0xA000-0xBFFF] 卡带RAM（External 仅限外用）：GameBoy中有少量可写的内存。如果制作的游戏硬件内存不足，可以在这里可以使用额外的8k内存。
* [0xC000-0xDFFF] 工作RAM：GameBoy内部可以由CPU读取或写入的8k内存。
* [0xE000-0xFDFF] 工作RAM（shadow）：由于GameBoy硬件的接线方式，实际的工作RAM比内存映射中少8k。直到内存映射中的最后512字节才可以访问。 
* [0xFE00-0xFE9F]图形，sprite信息：由图形芯片渲染的sprites的数据被保存在这里，其中包括sprite的位置和属性。
* [FF00-FF7F] 存储器映射I/O：GameBoy的每个子系统（图形、声音等）都有控制值，用来让程序生成效果并使用硬件。CPU可以直接通过这块区域在地址总线上使用这些值。
* [0xFF80-0xFFFF] 内存最高位有128字节RAM高速区域。奇怪的是，虽然它是整个存储器的255“页”，但由于大部分程序和GameBoy硬件之间的交互通过使用这块内存，所以它被称为Zero-page。 

## 连接到CPU

为了让模拟的CPU能够独立访问这些区域，每一个区域必须在内存管理单元中被处理为一个特殊的case。前面文章已经部分提及了这部分的代码，并且完成了一个MMU对象的基本接口。实现这个接口只需要一个简单的switch语句：

    MMU.js: 映射读取
    MMU = {
        // 表示BIOS是否已映射进来        
        // BIOS未映射时第一条指令从0x00FF之后开始
        _inbios: 1,
    
        // 内存区域（在复位时初始化）
        _bios: [],
        _rom: [],
        _wram: [],
        _eram: [],
        _zram: [],
    
        // 从内存中读1字节
        rb: function(addr)
        {
    	switch(addr & 0xF000)
    	{
    	    // BIOS (256b)/ROM0
    	    case 0x0000:
    	        if(MMU._inbios)
    		{
    		    if(addr < 0x0100)
    		        return MMU._bios[addr];
    		    else if(Z80._r.pc == 0x0100)
    		        MMU._inbios = 0;
    		}
    
    		return MMU._rom[addr];
    
    	    // ROM0
    	    case 0x1000:
    	    case 0x2000:
    	    case 0x3000:
    	        return MMU._rom[addr];
    
    	    // ROM1 (未备份) (16k)
    	    case 0x4000:
    	    case 0x5000:
    	    case 0x6000:
    	    case 0x7000:
    	        return MMU._rom[addr];
    
    	    // 图形VRAM (8k)
    	    case 0x8000:
    	    case 0x9000:
    	        return GPU._vram[addr & 0x1FFF];
    
    	    // Etx外用RAM (8k)
    	    case 0xA000:
    	    case 0xB000:
    	        return MMU._eram[addr & 0x1FFF];
    
    	    // 工作RAM (8k)
    	    case 0xC000:
    	    case 0xD000:
    	        return MMU._wram[addr & 0x1FFF];
    
    	    // 工作RAM（shadow）
    	    case 0xE000:
    	        return MMU._wram[addr & 0x1FFF];
    
    	    // 工作RAM（shadow）, I/O, Zero-page RAM
    	    case 0xF000:
    	        switch(addr & 0x0F00)
    		{
    		    // 工作RAM（shadow）
    		    case 0x000: case 0x100: case 0x200: case 0x300:
    		    case 0x400: case 0x500: case 0x600: case 0x700:
    		    case 0x800: case 0x900: case 0xA00: case 0xB00:
    		    case 0xC00: case 0xD00:
    		        return MMU._wram[addr & 0x1FFF];
    
    		    // 图形: 对象属性内存(OAM)
    		    // OAM为160字节，其余字节读取为0
    		    case 0xE00:
    		        if(addr < 0xFEA0)
    			    return GPU._oam[addr & 0xFF];
    			else
    			    return 0;
    
    		    // Zero-page
    		    case 0xF00:
    		        if(addr >= 0xFF80)
    			{
    			    return MMU._zram[addr & 0x7F];
    			}
    			else
    			{
    			    // I/O控制处理
    			    // 目前未处理
    			    return 0;
    			}
    		}
    	}
        },
    
        Read a 16-bit word
        rw: function(addr)
        {
            return MMU.rb(addr) + (MMU.rb(addr+1) << 8);
        }
    };
    
在上面的代码中，应该注意0xFF00到0xFF7F之间的内存区域现在未处理。这些位置被用作提供I/O的各种芯片的内存映射I/O，其含义将在后面部分涉及到时介绍。

写字节的处理方式与读非常相似。每个操作都是相反的，值被写入内存的各种区域，而不是从函数中被返回。

## 加载ROM
就像CPU模拟如果没有内存访问、图形等基本部分的支持就毫无意义一样，如果不能加载程序的话，从内存中读取程序同样没什么用。将程序装载到模拟器主要有两种方法：将其直接硬编码到模拟器代码中，或者允许从其他位置加载ROM文件。硬编码进程序明显的缺点是ROM是固定的，不容易更换。

在我们的JavaScript模拟器中，因为GameBoy的BIOS不会改变，所以BIOS被硬编码在MMU中。程序文件是在模拟器初始化之后异步加载的，这一步可以通过XMLHTTP或者Andy Na's BinFileReader之类的二进制文件读取器来完成，最终得到一个包含ROM文件的字符串。

    MMU.js: 加载ROM文件
    MMU.load = function(file)
    {
        var b = new BinFileReader(file);
        MMU._rom = b.readString(b.getFileSize(), 0);
    };

由于ROM文件被保存为字符串而不是整型数组，rb和wb函数必须改成字符串索引：

    MMU.js: ROM文件索引
    	    case 0x1000:
    	    case 0x2000:
    	    case 0x3000:
    	        return MMU._rom.charCodeAt(addr);

## 下一步
当CPU和MMU完成后，我们可以看到程序一步步的被执行：获得一个模拟器，并且在正确的寄存器中生成预期的值。现在还缺少的是图像输出，在本系列的下一篇将讨论图形的问题，包括GameBoy如何构建其图形输出，以及如何将图形渲染到屏幕上。

> 原文链接：[GameBoy Emulation in JavaScript: Memory](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Memory)


