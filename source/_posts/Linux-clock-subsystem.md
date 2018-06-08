---
title: Linux clock subsystem
date: 2018-06-07 10:44:40
tags:
- Linux
categories:
- Linux
---

### 前言

Linux Clock的框架图：

{% asset_img ccf.png CCF %}

以上的框架图就能说明整个clock子系统的运作情况，从上往下看，clock consumer -- clock core  -- clock provider，同时通过DTS来配置串接。大多数Linux子系统中都呈现这种分层式架构。

clock consumer ：向其它driver提供操作clocks的通用API。
clock core：实现clock控制的通用逻辑，这部分和硬件无关。
clock provider：将和硬件相关的clock控制逻辑封装成操作函数集，交由底层的platform开发者实现，由通用逻辑调用。

### 硬件相关
那么板子上clock相关模块到底是如何分布和规划的？

{% asset_img clk-hw.png Clock Hardware %}

上图从左往右看，最开始的源头产生：Oscillator  --> 中间可能有倍频PLL、分频Divider、多路选择Mux、开关Gate --> clock使用的硬件模块。

也就是说，软件层面的架构 与 硬件层面的架构有一定程度的吻合。比如软件需要提供：1.Enable/Disable Clk；2. Set clk rate; 3. 选择clock的parent。 

### 软件架构
主要是如何体现clock consumer <---> clock core(CCF)  <---> clock provider。两个层面问题，一是clock provider是如何注册进CCF的，二是clock consumer如何从CCF通过相应API获取对应的Clock控制。

##### 1.Clock provider注册进CCF

以Allwinner H5 Linux3.10为例，初始化的起点从start_kernel() 进入板级DT_MACHINE_START，如下：
linux-3.10\arch\arm\mach-sunxi\Sun50i.c 
```
DT_MACHINE_START(SUNXI_DT, CONFIG_SUNXI_SOC_NAME)
 .map_io	= sunxi_map_io,
 .init_time	= sunxi_timer_init,
 .dt_compat	= sunxi_board_dt_compat,
MACHINE_END
```
从sunxi_timer_init -- > of_clk_init --> __clk_of_table中所有需要注册的各种类型clock -- > linux-3.10\drivers\clk\sunxi\clk-sun50iw2.c 中CLK_OF_DECLARE定义的初始化函数。

以上有两个问题阐释下，一是`__clk_of_table`从何而来，二是具体初始化哪些clock类型。
`__clk_of_table`是从所有`CLK_OF_DECLARE`定义而来，而初始化哪些clock则是由`CLK_OF_DECLARE`中的定义与Clock Provider DTS中的定义匹配而来，举个Fixed Clock的例子：
clk-sun50iw2.c中定义`CLK_OF_DECLARE(sunxi_fixed_clk, "allwinner,fixed-clock",of_sunxi_fixed_clk_setup);` ，而DTS(sun50iw1p1-clk.dtsi)中的定义：
```
/* register fixed rate clock*/
  clk_losc: losc {
   #clock-cells = <0>;
   compatible = "allwinner,fixed-clock";
   clock-frequency = <32768>;
   clock-output-names = "losc";
  };

  clk_hosc: hosc {
   #clock-cells = <0>;
   compatible = "allwinner,fixed-clock";
   clock-frequency = <24000000>;
   clock-output-names = "hosc";
  };
```
从上面可以看到是通过`allwinner,fixed-clock`字段来匹配的，匹配之后最终通过clk_register/of_clk_add_provider添加到CCF中以供Clock Consumer使用。这只是其中一种clock类型，那么还有哪些类型呢？
根据clock的特点，clock framework将clock分为fixed rate、gate、devider、mux、fixed factor、composite六类，每一类clock都有相似的功能、相似的控制方式，因而可以使用相同的逻辑，统一处理，这充分体现了面向对象的思想。

- fixed rate clock：具有固定的频率，不能开关、不能调整频率、不能选择parent、不需要提供任何的clk_ops回调函数，是最简单的一类clock。
- gate clock：只可开关，会提供.enable/.disable回调。
- divider clock：可以设置分频值，因而会提供.recalc_rate/.set_rate/.round_rate回调。
- mux clock：可以选择多个parent，因为会实现.get_parent/.set_parent/.recalc_rate回调。
- fixed factor clock：具有固定的factor（即multiplier和divider），clock的频率是由parent clock的频率，乘以mul，除以div，多用于一些具有固定分频系数的clock。由于parent clock的频率可以改变，因而fix factor clock也可该改变频率，因此也会提供.recalc_rate/.set_rate/.round_rate等回调。
- composite clock：顾名思义，就是mux、divider、gate等clock的组合。

##### 2.Clock Consumer如何使用Clock

以Emmc模块为例，首先需要在DTS中EMMC相关部分配置，指明需要消费哪些Clock Provider中提供clock类型：
```
  sdc2: sdmmc@01C11000 {
   compatible = "allwinner,sun50i-sdmmc2";
   ~~~~~~
   clocks = <&clk_hosc>,<&clk_pll_periph1x2>,<&clk_sdmmc2_mod>,<&clk_sdmmc2_bus>,<&clk_sdmmc2_rst>;
   clock-names = "osc24m","pll_periph","mmc","ahb","rst";
   ~~~~~~
       };
```
如以上的clk_sdmmc2_mod、clk_sdmmc2_bus、clk_sdmmc2_rst就有在`sun50iw1p1-clk.dtsi`中定义。然后在emmc驱动相关代码中通过CCF提供的API取得相关clk并配置，如下(`sunxi-mmc.c`)：
```
 host->clk_mmc = devm_clk_get(&pdev->dev, "mmc");  -- 取得clk
 if (IS_ERR(host->clk_mmc)) {
  dev_err(&pdev->dev, "Could not get mmc clock\n");
  ret = PTR_ERR(host->clk_mmc);
  goto error_disable_regulator;
 }

 ret = clk_prepare_enable(host->clk_mmc); -- 使能clk
 if (ret) {
  dev_err(&pdev->dev, "Enable mmc clk err %d\n", ret);
  goto error_disable_clk_ahb;
 }
```
除了`devm_clk_get/clk_prepare_enable`外还有一些clock操作的API：
- clk_enable/clk_disable，启动/停止clock。不会睡眠。
- clk_prepare/clk_unprepare，启动clock前的准备工作/停止clock后的善后工作。可能会睡眠。
- clk_get_rate/clk_set_rate/clk_round_rate，clock频率的获取和设置，其中clk_set_rate可能会不成功（例如没有对应的分频比），此时会返回错误。如果要确保设置成功，则需要先调用clk_round_rate接口，得到和需要设置的rate比较接近的那个值。
- 获取/选择clock的parent clock。
- clk_prepare_enable，将clk_prepare和clk_enable组合起来，一起调用。clk_disable_unprepare,将clk_disable和clk_unprepare组合起来，一起调用。

更详细的解读可参考如下。

### ref

[Linux common clock framework(1)_概述](http://www.wowotech.net/linux_kenrel/clk_overview.html)
[Common Clock Framework系统结构](http://www.wowotech.net/pm_subsystem/ccf-arch.html)












