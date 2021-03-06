---
layout: post
title: JavaScript模拟GameBoy:中断
description: "Translation of \"GameBoy Emulation in JavaScript\" series articles."
modified: 2018-02-06
tags: [Translation]
categories: []
---

这是《JavaScript模拟GameBoy》系列文章的第八篇。目前已经完成了前十篇，其余的我将会继续跟进。

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

在前面的部分中，通过介绍精灵我们完成了模拟游戏的基础，然而模拟器还缺少垂直消隐中断。在本篇中，将完整引入中断，特别是中断的实现。一旦完成这部分，模拟器将可以运行俄罗斯方块。

想象一下你有一台带网卡的计算机，还有一些处理网络数据的软件。从计算机的角度来看，数据会经常出现，所以你需要有一些方法让软件知道新数据已经到达了，这可以通过两种方法来实现：

* 轮询：软件经常询问网卡是否有新数据到达。这种方法实施起来很简单，但有以下缺点：
    * 软件在定期检查之前对新数据一无所知，这意味着数据到达计算机和计算机处理之间的延迟。
    * 必须定期抽时间进行检查，即使没有数据到达也要从其他工作中抽出时间。
    * 如果数据到达的速度比轮训过程处理数据的速度快，则网卡中的数据会产生积压，并且有可能丢失一些数据。
    * 如果没有工作要做，软件仍必须检查数据，这会导致计算机在没有工作要做的情况下仍然全速运行。
* 中断：网卡通知软件新数据已经到达。这是一个需要更多步骤更加复杂的接受数据的方式，但它能够更好的应对轮询方式带来的缺点：
    * 新数据一到达就可以被处理，到达和处理数据之间没有任何延迟。
    * 只有确实有数据要处理时，软件才花费时间处理数据。并且可以根据需要经常调用处理程序清除积压。
    * 如果没有其他工作要做，计算机可以进入低功耗模式，直到网卡接受新数据唤醒计算机。

## 中断和中断处理
很明显，中断这个概念很有用，但中断需要硬件和软件的同时支持。在硬件方面，中断到达时CPU必须暂停执行中断服务程序（有时称为中断服务程序）。在上述场景中，网卡和CPU有一条线路允许数据到达时网卡通知CPU。

![图1：中断的硬件实现](http://imrannazar.com/content/img/jsgb-int-hardware.png)

图1：中断的硬件实现

CPU将在每条指令结束时检查中断输入，如果某个连接的外围设备（如网卡）给出了中断信号，CPU将会启动中断处理步骤。CPU将会保存正常执行离开时的位置，记录中断产生，然后跳到处理程序。

![图2：CPU中断处理程序](http://imrannazar.com/content/img/jsgb-int-cpu.png)

图2：CPU中断处理程序

在GameBoy中有五个从各种外设反馈进来的中断线路。其中每个有自己的ISR，位于内存的不同地址中。中断列表如下：


| 中断 | ISR地址 |
| --- | --- |
| 垂直消隐期 | 0x0040 |
| LCD状态触发器 | 0x0048 |
| 计时器溢出 | 0x0050 |
| 串行链接 | 0x0058 |
| 手柄按压 | 0x0060 |

表1：GameBoy中的中断

在垂直消隐期情况下，连线连在LCD底部，一旦GPU扫描完所有的LCD线进入屏幕底部，中断就会触发，CPU跳转到0x0040，执行消隐期ISR。

## 实现：中断标志
大部分CPU包含一个用于中断的主标志，只有在这个标志启用时，CPU才会执行中断。GameBoy的Z80也不例外，同时Z80还有其他寄存器可以处理GameBoy中的各个中断。这些寄存器是内存寄存器，所以它们由内存管理单元处理：


| 寄存器 | 位置 | 注释 |
| --- | --- | --- |
| 是否可中断 | 0xFFFF | 当位被设置后，相应的中断可以被触发 |
| 中断标志 | 0xFF0F | 当位被设置后，一个中断就产生了 |

表2：MMU中的中断标志（细节见下表）


| Bit | When 0 | When 1 |
| --- | --- | --- |
| 0 | Vblank off  | Vblank on |
| 1 | LCD stat off | LCD stat on |
| 2 | Timer off | Timer on |
| 3 | Serial off | Serial on |
| 4 | Joypad off | Joypad on |

表3：MMU中的中断标志细节

因为这些是内存寄存器，他们在MMU中实现：

    MMU.js: 中断标志
    MMU = {
        _ie: 0,
        _if: 0,
    
        rb: function(addr)
        {
    	switch(addr & 0xF000)
    	{
    	    ...
    	    case 0xF000:
    	        switch(addr & 0x0F00)
    		{
    		    ...
    		    // Zero-page
    		    case 0xF00:
    		    	if(addr == 0xFFFF)
    			{
    			    return MMU._ie;
    			}
    		        else if(addr >= 0xFF80)
    			{
    			    return MMU._zram[addr & 0x7F];
    			}
    			else
    			{
    			    // I/O控制处理
    			    switch(addr & 0x00F0)
    			    {
    			    	case 0x00:
    				    if(addr == 0xFF0F) return MMU._if;
    				    break;
    			    	...
    			    }
    			    return 0;
    			}
    		}
    	}
        },
        ...
    };


Z80的主启动开关以相似的方式在Z80中实现。CPU为软件提供操作码，以便将主启动器打开或关闭，所以这些也需要实现：

    Z80.js: 中断主启动
    Z80 = {
        _r: {
            ime: 0,
            ...
        },
    
        reset: function()
        {
            ...
    	Z80._r.ime = 1;
        },
    
        // 禁用IME
        DI: function()
        {
        	Z80._r.ime = 0;
    	Z80._r.m = 1;
    	Z80._r.t = 4;
        },
    
        // 启动IME
        EI: function()
        {
        	Z80._r.ime = 1;
    	Z80._r.m = 1;
    	Z80._r.t = 4;
        }
    };

## 实现：中断处理
使用中断标志后，主执行循环可以重新开发，从而更符合图2的执行路径。执行后，中断标志需要检查是否出现启动的中断。如果有的话可以调用它的处理程序。

    Z80.js: 垂直消隐期中断处理
    Z80 = {
        _ops: {
    	...
    
            // 启动垂直消隐期处理（0x0040）
            RST40: function()
            {
            
            // 禁用更多的中断
        	    Z80._r.ime = 0;
        
        	    // 将当前SP保存到栈
        	    Z80._r.sp -= 2;
        	    MMU.ww(Z80._r.sp, Z80._r.pc);
        
        	    // 转跳到处理程序
        	    Z80._r.pc = 0x0040;
    	    Z80._r.m = 3;
        	    Z80._r.t = 12;
            },
            
            // 从中断中返回（处理程序调用）
            RETI: function()
            {
    	    // 恢复中断
    	    Z80._r.ime = 1;
    
    	    // 转跳到栈上的地址
    	    Z80._r.pc = MMU.rw(Z80._r.sp);
    	    Z80._r.sp += 2;
    
    	    Z80._r.m = 3;
    	    Z80._r.t = 12;
            }
        }
    };
    
    while(true)
    {
        // 为此指令执行
        var op = MMU.rc(Z80._r.pc++);
        Z80._map[op]();
        Z80._r.pc &= 65535;
        Z80._clock.m += Z80._r.m;
        Z80._clock.t += Z80._r.t;
        Z80._r.m = 0;
        Z80._r.t = 0;
    
        // 如果IME处于打开状态，并且在IE中启用了一些中断，并且设置了中断标志，则处理中断
        if(Z80._r.ime && MMU._ie && MMU._if)
        {
            // 遮蔽未启用的整数
            var ifired = MMU._ie & MMU._if;
    
    	if(ifired & 0x01)
    	{
    	    MMU._if &= (255 - 0x01);
    	    Z80._ops.RST40();
    	}
        }
    
        Z80._clock.m += Z80._r.m;
        Z80._clock.t += Z80._r.t;
    }

## 下一次：更大的游戏
运行了俄罗斯方块后，模拟器已经能大致模拟已发布的游戏了。但却是在游戏规模上还存在问题，俄罗斯方块的ROM 32KB，能被完全放入存储器映射中的ROM空间。一般游戏ROM比这个更大，并且遵循将ROM部分映射到存储器的过程。下一次，我们将看看GameBoy最简单的映射形式，以及它在64KB游戏ROM上的实现。


> 原文链接：[GameBoy Emulation in JavaScript: Interrupts](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Interrupts)


