---
title: Linux GPIO-Pinctrl Subsystem
date: 2018-06-08 08:59:59
tags:
- Linux
categories:
- Linux
---


### 前言
对于一个IO Port来讲，有两方面需要配置，一是本身的功能设定，二是如果作为GPIO的话输入输出的设定。前者由Pin Controller控制，后者由GPIO Controller控制。

Pin Controller控制包括：

- 引脚功能配置。例如该I/O pin是一个普通的GPIO还是一些特殊功能引脚（例如memeory bank上CS信号）。
- 引脚特性配置。例如pull-up/down电阻的设定，drive-strength的设定等。

GPIO Controller控制包括：

- 配置GPIO的方向
- 如果是输出，可以配置high level或者low level
- 如果是输入，可以获取GPIO引脚上的电平状态

### 软件架构

linux kernel中的GPIO subsystem则用两个软件模块来对应上面两类硬件功能：

- pin control subsystem的模块图

{% asset_img pinctrl.png Linux pinctrl %}

底层的pin controller driver是硬件相关的模组，初始化的时候会向pin control core模块注册pin control设备（通过pinctrl_register这个bootom level interface）。pin control core模块是一个硬件无关模块，它抽象了所有pin controller的硬件特性，仅仅从用户（各个driver就是pin control subsystem的用户）角度给出了top level的接口函数，这样，各个driver不需要关注pin controller的底层硬件相关的内容。

- GPIO subsystem的模块图

{% asset_img io.png Linux gpio %}

基本上这个软件框架图和pin control subsystem是一样的，其软件抽象的思想也是一样的，当然其内部具体的实现不一样。

后对照Allwinner H5 Linux3.10的实际代码看，Pinctrl和Gpio是整合到一起的，传统gpio的调用最终还是到Pinctrl下面的API来实现。整体框架如下：
{% asset_img sys.png Linux pinctrl subsystem %}
##### 1. 普通Driver如何调用pinctrl
上图从上往下看，普通Driver调用pin control subsystem的主要目标是：1.设定该设备的功能复用；2. 设定该device对应的那些pin的电气特性。以emmc模块驱动(`sunxi-mmc.c`)为例：
DTS中Emmc部分的配置为：
```
  sdc0: sdmmc@01c0f000 {
   compatible = "allwinner,sun50i-sdmmc0";
   pinctrl-names = "default","sleep";
   pinctrl-0 = <&sdc0_pins_a>;
   pinctrl-1 = <&sdc0_pins_b>;
  };
```
Pinctrl部分的DTS配置为：
```
  r_pio: pinctrl@01f02c00 {
   compatible = "allwinner,sun50i-r-pinctrl";

   sdc0_pins_a: sdc0@0 {
    allwinner,pins = "PF0", "PF1", "PF2", "PF3","PF4","PF5";
    allwinner,function = "sdc0";
    allwinner,muxsel = <2>;
    allwinner,drive = <1>;
    allwinner,pull = <1>;
   };

   sdc0_pins_b: sdc0@1 {
    allwinner,pins = "PF0", "PF1", "PF2", "PF3","PF4","PF5";
    allwinner,function = "io_disabled";
    allwinner,muxsel = <7>;
    allwinner,drive = <1>;
    allwinner,pull = <1>;
   };
```
其中pinctrl中有两种状态，default是默认状态，从上面dts配置可以看出配置为sdc0的Function，sleep是休眠时或是emmc关闭时调用的状态，从上可看出配置为io_disabled状态。
代码中调用的流程：
```
host->pinctrl = devm_pinctrl_get(&pdev->dev);      
---  1. 获取设备（设备模型中的struct device）的pin control state holder（struct pinctrl）
host->pins_default = pinctrl_lookup_state(host->pinctrl,PINCTRL_STATE_DEFAULT);  
-- 2a. 根据state name在pin control state holder找到对应的pin control state, 这里是default
host->pins_sleep = pinctrl_lookup_state(host->pinctrl,PINCTRL_STATE_SLEEP);
-- 2b. 这里是sleep状态
ret = pinctrl_select_state(host->pinctrl , host->pins_sleep); -- 3a. 休眠时或是emmc关闭时调用
rval = pinctrl_select_state(host->pinctrl, host->pins_default); -- 3b. 打开时调用
```
##### 2. 板级Pinctrl driver如何注册
pinctrl driver根据pin controller的实际情况，实现struct pinctrl_desc（包括pin/pin group的抽象，function的抽象，pinconf、pinmux的operation API实现，dt_node_to_map的实现，等等），并注册到kernel中。
代码流程-->`pinctrl-sun50iw1p1.c`开始：
```
sun50iw1p1_pinctrl_probe 
    --> sunxi_pinctrl_init(pdev,&sun50iw1p1_pinctrl_data);  -- 其中sun50iw1p1_pinctrl_data描述了所有IO的Function及irq状况
    -->
     pctrl_desc->confops = &sunxi_pconf_ops;   -- 操作函数是用来配置引脚的特性,如pull-up/down，driver strength
     pctrl_desc->pctlops = &sunxi_pctrl_ops;  -- dts 功能group的相关解析
     pctrl_desc->pmxops = &sunxi_pmx_ops; -- 功能复用enable/disable，gpio方向
    -->
     pctl->pctl_dev = pinctrl_register(pctrl_desc,&pdev->dev, pctl); -- 注册到pin control subsystem
```
##### 3. Gpio如何关联
GPIO的HW block应该和其他功能复用的block是对等关系的，它们共同输入到一个复用器block，这个block的寄存器控制哪一个功能电路目前是active的。

{% asset_img gpio.png gpio %}

代码中，`sunxi_pinctrl_init(pdev,&sun50iw1p1_pinctrl_data)`-->
```
 pctl->chip->request = sunxi_pinctrl_gpio_request,
 pctl->chip->free = sunxi_pinctrl_gpio_free,
 pctl->chip->direction_input = sunxi_pinctrl_gpio_direction_input,
 pctl->chip->direction_output = sunxi_pinctrl_gpio_direction_output,
 pctl->chip->get = sunxi_pinctrl_gpio_get,
 pctl->chip->set = sunxi_pinctrl_gpio_set,
 pctl->chip->set_debounce = sunxi_pinctrl_gpio_set_debounce,
 pctl->chip->of_xlate = sunxi_pinctrl_gpio_of_xlate,
 pctl->chip->to_irq = sunxi_pinctrl_gpio_to_irq,

 ret = gpiochip_add(pctl->chip);
 if (ret)
  goto pinctrl_error;

 for (i = 0; i < pctl->desc->npins; i++) {
  const struct sunxi_desc_pin *pin = pctl->desc->pins + i;

  ret = gpiochip_add_pin_range(pctl->chip, dev_name(&pdev->dev),
          pin->pin.number - pctl->desc->pin_base,
          pin->pin.number, 1);
  if (ret)
   goto gpiochip_error;
 }
```
通过gpiochip_add(`gpiolib.c`)将 pctl->chip中指定的api与gpio子系统关联起来，以`gpiod_direction_output`为例：
```
gpiod_direction_output ----> 
    chip->direction_output(chip, offset, value) ---->
        pctl->chip->direction_output = sunxi_pinctrl_gpio_direction_output  ---->
            pinctrl_gpio_direction_output(chip->base + offset) ---->
                pinctrl_gpio_direction(gpio, false)  ---->
                    pinmux_gpio_direction(pctldev, range, pin, input) ---->
                        ops->gpio_set_direction(pctldev, range, pin, input) ---->
                            sunxi_pmx_gpio_set_direction  ---->  pinmux_ops里面的callback，所以最终回到pinctrl driver
                                sunxi_pmx_set(pctldev, offset, desc->muxval, true)  -- 最终控制 寄存器的地方
                                
```
以上只是根据allwinner H5实际代码大略分析，具体可以参考以下。

### refs
[Linux GOIO子系统-wowo](http://www.wowotech.net/sort/gpio_subsystem)