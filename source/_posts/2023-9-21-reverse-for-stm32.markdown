---
title: electron——OC/OD/上下拉/GPIO
categories: electron
tags: [electron]
index_img: /img/led.jpg
banner_img: /img/led.jpg
date: 2023-09-21 12:44:00
---
入坑电子的第一步
<!--more-->
## 一、OC/OD门
OC门和OD门都是集成电路中比较重要的简单电路组合，其中，OC门的核心组件是三极管（以NPN三极管为例），信号从基极输入，基极与发射极之间使用电阻连接，集电极作为输出开路。同理，OD门将OC门中的三极管换成MOSEFT即可。  
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/assets/od.png" width="85%">
    <br>
    <div style="color:orange;
    display: inline-block;
    color: #999;
    padding: 15px;">od门</div>
</center>
OC门的三极管工作在截至状态或者饱和状态，其输入和输出信号均为数字信号，当输入为高电平时，三极管导通，工作在饱和状态，由于该工作状态的限制，所以集电极和发射极（接地）之间的电压不超过0.3V，可以理解为低电平。而当输入为低电平时，三极管工作在截至状态，此时，输出部分处于高阻态，容易受到外界扰动。  
OD门的工作状态与OC门十分相似，且性能在很多情况下好于OC门，在此不作赘述。  

## 二、上下拉电阻
以OC门为例，前文提到，在三极管截止时，输出端为高阻态，无法进行稳定的输出，解决该种问题的方法是在输出端（集电极）上连接一个定值电阻，电阻的另一端与Vcc相连，这种连接方式即为上拉电阻。在该种情况下继续分析：当输入为高电平时，集电极和发射极间的电压由于工作状态的限制仍不超过0.3V，而输出低电平；当输入为低电平时，上拉电阻与输出信号的端口内阻串联分压，输出默认为高电平，这就保证了输出为低时高阻态不会出现。  
下拉电阻与上拉电阻同理，在信号线上用电阻接地，使信号输出默认为低电平，在此不做赘述。  

## 三、推挽输出
推挽输出的电路，由两个相反的MOSEFT构成，PMOS的源级接电源，漏极与NMOS的漏极相连，NMOS的源极接地，输出接到PMOS/NMOS的栅极，输出为NMOS的漏极。  
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/assets/push_pull.png" width="85%">
    <br>
    <div style="color:orange;
    display: inline-block;
    color: #999;
    padding: 15px;">推挽输出简化电路</div>
</center>
对电路进行分析，当输入为高电平时，NMOS导通，PMOS截止，输出引脚和接地的NMOS栅极同一电平，输出低电平，这种状态是“挽”即从输出拉取电流。  
当输入为低电平时，NMOS截止，PMOS导通，Vdd与输出引脚导通，那么输出自然是高电平，这种状态是“推”即向输出推送电流。  

## 四、线与
线与是指将多个输出端互联实现按位与的功能，只有全部的输出为高电平时组合输出才为高电平。  
对于开漏输出（OC/OD门）而言，任意一个三极管或者NMOS导通，都代表着输出端与地之间导通，仅当两个NMOS都截止时才会输出高电平。  
而对于推挽输出而言，当一方输出为0，一方输出为1时，输出为1的一方将通过输出线产生流向输出为0一方NMOS的电流，而此时NMOS是导通的，电阻很小，会有烧坏的风险。

## 五、GPIO
在处理了很多的基本电路知识之后，我们来到了通用输入输出结构，作为发送和接收数字信号或者接收模拟信号的接口，CPU对于GPIO寄存器的操会经过GPIO驱动器映射到GPIO的输出上，下面来分析GPIO的驱动电路。  
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/assets/GPIO_bit.png" width="85%">
    <br>
    <div style="color:orange;
    display: inline-block;
    color: #999;
    padding: 15px;">GPIO 位结构</div>
</center>

在本次分析时，我们先忽略复用功能输入和复用功能输出，进行最基本的GPIO分析。  
首先是输出部分，输出数据寄存器的数据位与输出控制相连，输入给推挽输出的输入端（更进一步，输出控制可以控制推挽输出中MOS管是否工作，这样就可以将输出在推挽和开漏输出间进行转换），最后经过保护二极管作为IO引脚的输出。  
其次是输入部分，输入信号经过二极管保护与上下拉电阻的控制后，是一个施密特触发器，只在电压高于高电平或者低于低电平后开始跳变，可以防止抖动，失真等等情况，如果是模拟输入，该触发器会关闭，GPIO直接读取模拟输入。  
从以上的分析中我们可以看出，GPIO的配置不止需要数据，更需要确定其输入输出的选项功能。  
需要确定的部分如下：  
* 该引脚是否输入？
* 该引脚如果不输入，那么输出模式是开漏还是推挽？
* 该引脚如果输入，那么输入是上拉，下拉，浮空，还是模拟？  

在确定这些之后，我们只需要对数据寄存器进行读写就可以实现利用GPIO信号控制外设了。  

## 六、stm32 GPIO标准库分析及使用
根据stm32F10X的文档，每个GPI/O端口有两个32位配置寄存器(GPIOx_CRL，GPIOx_CRH)，两个32位数据寄存器(GPIOx_IDR和GPIOx_ODR)，一个32位置位/复位寄存器(GPIOx_BSRR)，一个16位复位寄存器(GPIOx_BRR)和一个32位锁定寄存器(GPIOx_LCKR)。  
其中，CR寄存器的MODE负责调整引脚是输出还是输入，如果是输出，输出速率是多少；CR寄存器的CNF负责调整输入的上下拉和模拟，输出的推挽还是开漏。数据寄存器ODR映射到输出，而BSRR和BRR以位操作的方式来置位某一位的输入输出。  
接下来我们从标准库的源码角度来分析GPIO的操作（对代码做出一系列简化）：  
### RCC_APB2PeriphClockCmd
当外设时钟没有启用时，软件不能读出外设寄存器的数值，返回的数值始终是0x0,所以首先设置APB2总线的时钟即可，APB2的时钟使能寄存器RCC的每一位都代表一个连接在其上的外设，直接置位之后则可以将其打开，所以，函数传入的其实是一个位图。  
```c
void RCC_APB2PeriphClockCmd(uint32_t RCC_APB2Periph, FunctionalState NewState){
  if (NewState != DISABLE){
    RCC->APB2ENR |= RCC_APB2Periph;
  }
  else{
    RCC->APB2ENR &= ~RCC_APB2Periph;
  }
}
```

### GPIO_Init
首先，传入的参数分别是GPIO端口的所属，例如GPIOA、GPIOB等等，然后是GPIO的参数，包括具体引脚，MODE和速率。  
首先是通过结构体提取了MODE部分，该部分包括了寄存器中CNF和MODE的内容，取出与0x10这一位，代表输入/输出，如果是输出，则初始化输出速率。（其实除了MODE+CNF的四位，还有剩下的位数是分辨读写和上下拉的，因为上下拉的MODE+CNF相同）  
```c
void GPIO_Init(GPIO_TypeDef* GPIOx, GPIO_InitTypeDef* GPIO_InitStruct){
  uint32_t currentmode = 0x00, currentpin = 0x00, pinpos = 0x00, pos = 0x00;
  uint32_t tmpreg = 0x00, pinmask = 0x00;

  currentmode = ((uint32_t)GPIO_InitStruct->GPIO_Mode) & ((uint32_t)0x0F);

  if ((((uint32_t)GPIO_InitStruct->GPIO_Mode) & ((uint32_t)0x10)) != 0x00){ 
    currentmode |= (uint32_t)GPIO_InitStruct->GPIO_Speed;
  }
  ……
```
接下来是初始化前8个针脚，按照位图的方式一个个检查需要初始化的针脚是否存在。如果存在，则找到整个CR寄存器的值中设置对应针脚的地方并清空，然后将提取的MODE的四位（GPIO_MODE&0x0F）用按位或的方式置入即可。  

```c  
  if (((uint32_t)GPIO_InitStruct->GPIO_Pin & ((uint32_t)0x00FF)) != 0x00){
    tmpreg = GPIOx->CRL;
    for (pinpos = 0x00; pinpos < 0x08; pinpos++){
      pos = ((uint32_t)0x01) << pinpos;
      /* Get the port pins position */
      currentpin = (GPIO_InitStruct->GPIO_Pin) & pos;
      if (currentpin == pos){
        pos = pinpos << 2;
        /* Clear the corresponding low control register bits */
        pinmask = ((uint32_t)0x0F) << pos;
        tmpreg &= ~pinmask;
        /* Write the mode configuration in the corresponding bits */
        tmpreg |= (currentmode << pos);
        /* Reset the corresponding ODR bit */
        if (GPIO_InitStruct->GPIO_Mode == GPIO_Mode_IPD){
          GPIOx->BRR = (((uint32_t)0x01) << pinpos);
        }
        else{
          /* Set the corresponding ODR bit */
          if (GPIO_InitStruct->GPIO_Mode == GPIO_Mode_IPU)
          {
            GPIOx->BSRR = (((uint32_t)0x01) << pinpos);
          }
        }
      }
    }
    GPIOx->CRL = tmpreg;
  }
}
```

设置高位寄存器的方法十分相似，在此不做赘述。  
GPIO初始化后，便可以通过简单的Set和Reset来输出高低电平了，也可以用来读取电平输入。  
浅搭个小小的按键控制流水灯：
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/assets/led.jpg" width="85%">
    <br>
    <div style="color:orange;
    display: inline-block;
    color: #999;
    padding: 15px;">GPIO点灯</div>
</center>

## Reference
> 详解：开漏输出与推挽输出 https://zhuanlan.zhihu.com/p/637921779  
> 理一理 OC/OD 门、开漏输出、推挽输出等一些相关概念 https://zhuanlan.zhihu.com/p/555471581    
> 江协科技bilibili stm32教程 https://www.bilibili.com/video/BV1th411z7sn/  
> stm32中文手册  
> 新概念模拟电路 杨建国